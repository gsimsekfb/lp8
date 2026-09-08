# Step 5: Module Structure Research Notes

## The Logos Module Pattern

Every Logos Core module follows the **exact same pattern**. Here's the blueprint:

### File Structure
```
my-module/
├── CMakeLists.txt          # Uses logos_module() CMake macro
├── flake.nix               # Nix build — calls mkLogosModule{}
├── flake.lock
├── metadata.json           # Module identity, deps, packaging info
├── src/
│   ├── my_module_interface.h    # Pure virtual interface (optional, for complex modules)
│   ├── my_module_plugin.h       # Qt plugin class header
│   └── my_module_plugin.cpp     # Implementation
└── tests/
    ├── CMakeLists.txt
    ├── main.cpp
    └── test_my_module.cpp
```

### The Plugin Class Recipe

Every module class inherits from 3 things:
```cpp
class MyModulePlugin : public QObject,           // Qt base
                       public PluginInterface,    // Logos SDK base (name, version)
                       public IMyModuleInterface  // Custom interface
{
    Q_OBJECT
    Q_PLUGIN_METADATA(IID IMyModuleInterface_iid FILE "metadata.json")
    Q_INTERFACES(PluginInterface IMyModuleInterface)

public:
    // Required by PluginInterface
    QString name() const override { return "my_module"; }
    QString version() const override { return "1.0.0"; }

    // Logos Core injects the API bridge here
    Q_INVOKABLE void initLogos(LogosAPI* logosAPIInstance);

    // Module-specific methods — all Q_INVOKABLE
    Q_INVOKABLE bool doSomething(const QString& param);

signals:
    // Every module emits events through this signal
    void eventResponse(const QString& eventName, const QVariantList& data);

private:
    void* externalCtx;  // Opaque pointer to external C library context
};
```

### Key Observations

| Aspect | How It Works |
|---|---|
| **Language** | All modules are **C++** with Qt6. No Rust modules exist yet. |
| **External libraries** | Each module wraps a **C library via FFI** (chat→liblogoschat, storage→libstorage, LEZ→wallet_ffi, delivery→liblogosdelivery, wallet→libgowalletsdk) |
| **Communication** | Async callbacks from C libs → converted to Qt signals (`eventResponse`) |
| **LogosAPI** | Injected via `initLogos()` — provides event routing to the host app |
| **Build system** | CMake (`logos_module()` macro) + Nix (`mkLogosModule{}`) |
| **Packaging** | Built into `.lgx` files (gzipped tarballs with manifest.json) |
| **Interface ID** | Every module declares a unique IID string, e.g. `"org.logos.StorageModuleInterface"` |

---

## Module-by-Module API Summary

### Chat Module (`logos-chat-module`)
**Wraps:** `liblogoschat` (C library)
**Pattern:** Async callbacks via `void*` context

| Method | What it does |
|---|---|
| `initChat(configJson)` | Create chat context with config |
| `startChat()` / `stopChat()` / `destroyChat()` | Lifecycle |
| `setEventCallback()` | Register for push events (new messages, etc.) |
| `getId()` | Get client identifier |
| `listConversations()` | List all conversations |
| `getConversation(convoId)` | Get conversation details |
| `newPrivateConversation(introBundle, content)` | Start E2E encrypted chat |
| `sendMessage(convoId, contentHex)` | Send message to conversation |
| `getIdentity()` | Get local identity info |
| `createIntroBundle()` | Create shareable public key bundle |

> **For LP-0008**: This is the module we'll use for the **owner channel** and **messaging skills**. The `newPrivateConversation` + `sendMessage` pair is exactly what we need.

### Delivery Module (`logos-delivery-module`)
**Wraps:** `liblogosdelivery` (C library, from logos-messaging/logos-delivery)
**Pattern:** Async callbacks + sync-wrapped operations

| Method | What it does |
|---|---|
| `createNode(cfg)` | Create P2P delivery node with config (cluster, preset, relay, RLN) |
| `start()` / `stop()` | Node lifecycle |
| `send(contentTopic, payload)` | Send message to a content topic |
| `subscribe(contentTopic)` | Subscribe to topic (receive messages) |
| `unsubscribe(contentTopic)` | Stop receiving |
| `getNodeInfo(id)` | Query node metadata |

> **For LP-0008**: Lower-level than chat. The **Agent Card discovery** (publish/subscribe to discovery topics) maps directly to `subscribe(topic)` + `send(topic, agentCard)`.

### LEZ Wallet Module (`logos-execution-zone-module`)
**Wraps:** `wallet_ffi` (Rust FFI library from `lssa` repo)
**Pattern:** Synchronous calls returning JSON strings

| Method | What it does |
|---|---|
| `create_account_public()` / `create_account_private()` | Create accounts |
| `list_accounts()` | List all accounts |
| `get_balance(id, isPublic)` | Query balance |
| `transfer_public/shielded/deshielded/private(...)` | Token transfers |
| `sync_to_block(blockId)` | Blockchain sync |
| `get_current_block_height()` | Query chain height |
| `create_new(configPath, storagePath, password)` | Create wallet |
| `open(configPath, storagePath)` | Open existing wallet |
| `save()` | Persist wallet state |

> **For LP-0008**: This is the module for **wallet skills** (`wallet.balance()`, `wallet.send()`) and **blockchain skills** (`program.query()`, `program.call()`). The agent needs its own `create_account_private()` call on first run.

### Storage Module (`logos-storage-module`)
**Wraps:** `libstorage` (C library)
**Pattern:** Mix of sync (waitForSignal) and async (events)

| Method | What it does |
|---|---|
| `init(cfg)` / `start()` / `stop()` / `destroy()` | Lifecycle |
| `uploadUrl(url, chunkSize)` / `uploadInit/Chunk/Finalize` | Upload files |
| `downloadToUrl(cid, url)` / `downloadChunks(cid)` | Download files |
| `exists(cid)` / `fetch(cid)` / `remove(cid)` | Content management |
| `space()` | Available storage space |
| `manifests()` | List stored files (with CID, size, filename, mimetype) |
| `downloadManifest(cid)` | Fetch file metadata |

> **For LP-0008**: Direct mapping to **storage skills** (`storage.upload()`, `storage.download()`, `storage.list()`).

### Wallet Module (`logos-wallet-module`)
**Wraps:** `libgowalletsdk` (Go library compiled to C)
**Pattern:** Synchronous calls

> This is the Ethereum-compatible wallet (for non-LEZ chains). Not directly needed for LP-0008, but shows the Go→C→C++ FFI pattern.

---

## Critical Architecture Insight: The Rust Question

> [!WARNING]
> **All existing modules are C++.** The LP-0008 spec says the agent module should follow "the Qt Remote Objects module interface," which currently means **C++ with Qt6**.
> 
> However, the LP-0008 spec also mentions Rust is viable via the C ABI bridge. The approach would be:
> 1. **Thin C++ Qt plugin shell** — handles `Q_OBJECT`, `Q_PLUGIN_METADATA`, `initLogos()`, and `eventResponse` signal
> 2. **Rust core logic** — compiled as a C library (`libagent.so`), called via `extern "C"` FFI from the C++ shell
> 3. This is exactly what the LEZ module does: `wallet_ffi` is a Rust library exposed via C FFI

### Recommended Approach for Our Agent Module
```
logos-agent-module/
├── CMakeLists.txt
├── flake.nix
├── metadata.json
├── src/
│   ├── agent_module_interface.h    # Pure virtual interface
│   ├── agent_module_plugin.h       # Thin C++ Qt shell
│   └── agent_module_plugin.cpp     # Delegates to Rust via FFI
├── agent-core/                     # Rust crate
│   ├── Cargo.toml
│   ├── src/
│   │   ├── lib.rs                  # C FFI exports
│   │   ├── skills/                 # Skill implementations
│   │   ├── owner_channel.rs        # Owner messaging
│   │   └── spending.rs             # Threshold logic
│   └── cbindgen.toml               # Generate C headers from Rust
└── tests/
```

---

## Build System Summary

### CMakeLists.txt Pattern
```cmake
cmake_minimum_required(VERSION 3.14)
project(LogosAgentModule LANGUAGES CXX)

if(DEFINED ENV{LOGOS_MODULE_BUILDER_ROOT})
    include($ENV{LOGOS_MODULE_BUILDER_ROOT}/cmake/LogosModule.cmake)
else()
    message(FATAL_ERROR "LogosModule.cmake not found.")
endif()

logos_module(
    NAME agent_module
    SOURCES
        src/agent_module_interface.h
        src/agent_module_plugin.h
        src/agent_module_plugin.cpp
    EXTERNAL_LIBS
        agent_core    # Our Rust library
)
```

### flake.nix Pattern
```nix
{
  description = "Logos Agent Module";
  inputs = {
    logos-module-builder.url = "github:logos-co/logos-module-builder";
    nix-bundle-lgx.url = "github:logos-co/nix-bundle-lgx";
  };
  outputs = inputs@{ logos-module-builder, ... }:
    logos-module-builder.lib.mkLogosModule {
      src = ./.;
      configFile = ./metadata.json;
      flakeInputs = inputs;
      externalLibInputs = {
        agent_core = { /* our Rust library build */ };
      };
    };
}
```

### metadata.json Pattern
```json
{
  "name": "agent_module",
  "version": "0.1.0",
  "description": "Autonomous AI Agent Module for Logos Core",
  "author": "LP-0008 Submission",
  "type": "core",
  "category": "agent",
  "main": "agent_module_plugin",
  "dependencies": [],
  "include": ["libagent_core.so"],
  "capabilities": []
}
```
