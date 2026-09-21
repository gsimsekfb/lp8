# Task 1.5 — Owner Channel (basic): Plan

**Goal:** Owner can send a command to the agent and get a response back.

#### Steps

1. **Research** — Study the chat module's API to understand how to create topics and send messages via Logos Messaging. Check if it also has the LIDL issue.

2. **Decide integration approach:**
   - **Option A:** Raw LogosAPI call to chat module (if LIDL is missing, same as wallet)
   - **Option B:** Our `processOwnerCommand()` already works as a direct RPC — any Logos app can call it via the generated `AgentModule` API. This might be sufficient for MVP.

3. **Test module loading in Logos Core** — First time we actually load our `.so` into the full runtime alongside other modules. This validates everything.

4. **Implement owner channel** — Create a dedicated messaging topic, wire up send/receive.

5. **Build & test** — Verify owner→agent→response flow.

6. **Spec check** — Compare against LP-0008 spec.

7. **Your review → commit.**


.  
------

# Impl - Phase A:

Ran command: `grep -n "subscribe\|LpSubscription" /nix/store/nd4dxlgjqn798xvql6qd9jrnzrpf5kdi-logos-cpp-sdk/include/logos_lp_client.h | head -20`

`LpClient` has:
- `invoke()` — sync call
- `invokeAsync()` — async call
- `subscribe(event, callback)` → `LpSubscription` — event subscription

### Task 1.5 Changes

**Header (`agent_module_impl.h`)**:
- `setupOwnerChannel(ownerIntroBundle)` — creates E2E encrypted channel via chat module
- `sendToOwner(message)` — sends via messaging or falls back to event
- `getOwnerChannelStatus()` — channel connection status
- `OwnerChannel` struct (forward-declared, PIMPL pattern)

**Implementation (`agent_module_impl.cpp`)**:
- `OwnerChannel` struct wraps `logos::LpClient` targeting `chat_module`
- `setupOwnerChannel()` calls `newPrivateConversation()` and subscribes to `messageReceived` events
- `sendToOwner()` hex-encodes and calls `sendMessage()`, falls back to event emission
- `onChatMessage()` routes incoming messages through `processOwnerCommand()` → `sendToOwner()`
- `processOwnerCommand()` now handles `"channel"` keyword
- `getStatus()` now includes `owner_channel_connected`

.  
------

# Task 1.5 — Owner Channel: Research Findings

## Key Discovery: We CAN call legacy modules without LIDL

The `logos_lp_client.h` header provides `logos::LpClient` — a **Qt-free** typed client that calls any module by name using JSON over the logos-protocol C ABI. No LIDL needed.

```cpp
// Create a client targeting "chat_module", calling from "agent_module"
logos::LpClient chatClient("chat_module", "agent_module");

// Invoke a method — args is a JSON array, returns JSON
auto result = chatClient.invoke("sendMessage", 
    nlohmann::json::array({"convo-id", "hello"}), &err);
```

This means:
- ✅ We can call the **chat module** (Logos Messaging) to create topics and send messages
- ✅ We can call the **wallet module** (LEZ) to create accounts — solving the NPK/ISK issue too
- ❌ No typed `modules()` accessor — but the raw `LpClient` is Qt-free and works fine

## Chat Module API (Logos Messaging)

| Method | Args | What it does |
|---|---|---|
| `initChat(configJson)` | JSON config | Initialize messaging |
| `startChat()` | — | Start messaging service |
| `getId()` | — | Get this client's messaging identity |
| `newPrivateConversation(introBundle, contentHex)` | Bundle + message | Create 1:1 encrypted channel |
| `sendMessage(convoId, contentHex)` | Conv ID + hex msg | Send message on a conversation |
| `listConversations()` | — | List active conversations |
| `getConversation(convoId)` | Conv ID | Get conversation details |
| `createIntroBundle()` | — | Create intro bundle for pairing |
| `setEventCallback()` | — | Register for incoming message events |

## Proposed Approach (two phases)

### Phase A: Direct RPC (what we have now — already works)
Our `processOwnerCommand()` is already callable by any Logos app via the generated `AgentModule` API. An owner running Basecamp can call it directly. This is sufficient for MVP testing.

### Phase B: Logos Messaging integration (real owner channel)
1. On `initialize()`, create a dedicated `LpClient("chat_module", "agent_module")`
2. Call `newPrivateConversation()` to create an E2E encrypted channel with the owner
3. Subscribe to incoming messages via `setEventCallback()`
4. When a message arrives, route it to `processOwnerCommand()` and send the response back via `sendMessage()`

> [!IMPORTANT]
> Phase B requires Logos Core running with both `agent_module` and `chat_module` loaded. We can't test this in isolation — need the full runtime.

## Recommendation

Do **Phase A first** — verify the module loads in Logos Core and `processOwnerCommand()` works via direct RPC. Then add Phase B (Messaging integration) once we confirm the runtime works.

## Also: NPK/ISK via LpClient

The same `LpClient` approach works for the wallet module:
```cpp
logos::LpClient walletClient("logos_execution_zone", "agent_module");
auto result = walletClient.invoke("create_account_private", nlohmann::json::array(), &err);
// result is the account ID hex string
```
This can replace our Ed25519 placeholder in Phase 2 (wallet skills) without needing LIDL.


.  
---

# Manual actions to test the changes


**Verify new methods are exported** — enter the dev shell and run `lm`:

```bash
cd /home/gok/logos-lp-8/logos-agent-module
nix develop --extra-experimental-features 'nix-command flakes'
lm ./result/lib/agent_module_plugin.so
```

You should see `setupOwnerChannel`, `sendToOwner`, and `getOwnerChannelStatus` in the method list alongside the existing ones.

**What can't be tested yet:**
- Actual messaging (needs Logos Core running with both `agent_module` + `chat_module` loaded)
- The `logoscore` CLI (GLIBC 2.38 mismatch on your machine)
