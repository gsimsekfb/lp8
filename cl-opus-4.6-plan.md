# LP-0008: Autonomous AI Agent Module — Research & Implementation Plan

> **Status: APPROVED — Executing**

## Glossary of Key Terms

Before diving in, here's what every domain-specific keyword in the LP-0008 spec actually means:

| Term | What It Is | Formerly Known As |
|---|---|---|
| **Logos** | A unified, privacy-first technology stack for sovereign digital communities. A single ecosystem combining blockchain, messaging, and storage. | The consolidation of Nomos, Waku, and Codex |
| **LEZ** (Logos Execution Zone) | The programmable blockchain layer — the "smart contract" environment. Uses RISC Zero ZK-proofs for privacy. Supports both public *and* shielded (private) accounts. Analogous to "Ethereum + privacy". | Part of Nomos |
| **LEZ Wallet** | A shielded cryptocurrency wallet on the LEZ blockchain. Holds tokens using an NPK/ISK keypair. NPK = Nullifier Public Key (identity), ISK = Incoming Secret Key (for receiving). The agent gets its *own* wallet, separate from the owner's. | — |
| **Logos Core** | The modular runtime/daemon that hosts all Logos services (wallet, storage, messaging, etc.) as dynamically-loaded plugins. Uses **Qt Remote Objects** for inter-module IPC. Think of it as a local service bus. | — |
| **Logos App (Basecamp)** | The desktop application / UI launcher. A local-first, modular client that bundles chat, wallet, file sharing, and blockchain explorer. Runs on the owner's machine. | — |
| **Logos Messaging** | P2P encrypted messaging layer with **no central server**. Uses libp2p relay network, end-to-end encryption (de-MLS for groups), and RLN for spam protection. | Waku v2 |
| **Logos Storage** | Decentralized, encrypted file storage. P2P network for censorship-resistant, durable data availability. Files are content-addressed (like IPFS). | Codex |
| **A2A Protocol** | Google's open Agent2Agent protocol (now Linux Foundation). Standard for agent discovery (Agent Cards), task delegation (Task lifecycle: submitted → working → input-required → completed/failed), and messaging. Uses HTTP/JSON-RPC normally, but LP-0008 replaces transport with Logos Messaging. | — |
| **Agent Card** | An A2A-standard JSON document advertising an agent's identity, skills, input/output schemas, and pricing. Normally at `/.well-known/agent.json`; here published to a Logos Messaging discovery topic. | — |
| **Logos Core Module** | A dynamically-loadable plugin for Logos Core, following the Qt Remote Objects interface. Written in **C++ (primary) or Rust** (via stable C ABI bridge). Uses `logos-module-builder` for packaging. | — |
| **NPK / ISK** | Nullifier Public Key / Incoming Secret Key. The cryptographic keypair for a shielded LEZ account. NPK identifies the account on-chain; ISK lets you receive funds. The agent's Messaging address is derived from this identity. | — |
| **RISC Zero** | The ZK-proof system underlying LEZ. Programs compile to RISC-V ELF binaries and run inside a zkVM. `RISC0_DEV_MODE=1` skips proof generation for faster dev; final demo must use `RISC0_DEV_MODE=0`. | — |
| **Cryptarchia** | Logos blockchain consensus — private proof-of-stake with anonymous block proposers. | — |

---

## Development Environment Decision

> [!IMPORTANT]
> **Where to develop: WSL2 (Ubuntu) on your Surface Pro 7 (16 GB RAM)**

| Option | Verdict | Reasoning |
|---|---|---|
| **WSL2 Ubuntu** | ✅ **Recommended** | Native Linux toolchain, direct access to Rust/Nix/Docker, shared filesystem with Windows. Logos Core and LEZ tooling are Linux-first. 16 GB is tight but workable with `RISC0_DEV_MODE=1` during development. |
| **VirtualBox Ubuntu** | ⚠️ **Backup only** | Higher overhead (VBox hypervisor on top of Hyper-V/WSL2 causes conflicts). Use only if you need isolated network testing or a clean snapshot-based workflow. |
| **Windows native** | ❌ **Not recommended** | Logos tooling (Nix, Qt build system, RISC Zero) assumes Linux. Cross-compilation would add massive friction. |

> [!WARNING]
> **RAM constraint**: RISC Zero proof generation (`RISC0_DEV_MODE=0`) is extremely memory-intensive (can exceed 16 GB for complex programs). For the final demo recording, consider:
> - Using a cloud VM (e.g., 32+ GB) just for the final `RISC0_DEV_MODE=0` demo
> - Keeping all development and testing in `RISC0_DEV_MODE=1` (skip proofs)
> - Closing all other Windows applications during proof generation

---

## High-Level Architecture

The system has **three deployment zones** and **six major components**:

```
┌─────────────────────────────────────┐
│     OWNER'S DEVICE (Surface Pro)    │
│                                     │
│  ┌───────────┐    ┌──────────────┐  │
│  │   Owner   │───▶│  Logos App   │  │
│  │ (Human)   │    │ (Basecamp)   │  │
│  └───────────┘    └──────┬───────┘  │
│                          │          │
│            E2E encrypted │ chat     │
│             (owner channel)         │
└──────────────────────────┼──────────┘
                           │
            ═══════════════╪═══════════  Distributed Logos Network
                           │
┌──────────────────────────┼──────────┐
│     REMOTE NODE (agent host)        │
│                                     │
│  ┌───────────────────────▼───────┐  │
│  │      🤖 AGENT MODULE         │  │
│  │    (this is what we build)    │  │
│  │                               │  │
│  │  ┌─────────┐ ┌────────────┐  │  │
│  │  │ AI/LLM  │ │  Spending  │  │  │
│  │  │ Engine  │ │  Threshold │  │  │
│  │  └─────────┘ └────────────┘  │  │
│  │  ┌─────────┐ ┌────────────┐  │  │
│  │  │  Skill  │ │   A2A      │  │  │
│  │  │Dispatch │ │ Compat     │  │  │
│  │  └─────────┘ └────────────┘  │  │
│  └───────────────────────────────┘  │
│            │         │        │     │
│       ┌────┘    ┌────┘   ┌────┘     │
│       ▼         ▼        ▼          │
│  ┌─────────┐┌────────┐┌─────────┐  │
│  │   LEZ   ││ Logos  ││  Logos  │  │
│  │ Wallet  ││Storage ││Messaging│  │
│  │(shielded)│(P2P enc)│(P2P,E2E)│  │
│  └─────────┘└────────┘└─────────┘  │
│                                     │
│         Logos Core Runtime          │
└─────────────────────────────────────┘
                   │
                   ▼
        ┌──────────────────┐
        │  Other Agents    │
        │  (on their own   │
        │   remote nodes)  │
        └──────────────────┘
```

---

## Implementation Plan — Three Phases

### Phase 1: Foundation & Core Module (Weeks 1–3)

> Goal: Get a Logos Core module that loads, has its own identity, and can talk to the owner.

#### 1.1 Environment Setup
- [ ] Set up WSL2 Ubuntu dev environment
- [ ] Install Rust toolchain (stable + nightly) via `rustup`
- [ ] Install RISC Zero toolchain via `rzup`
- [ ] Install Nix package manager (for `logos-module-builder`)
- [ ] Install Docker (for reproducible guest builds)
- [ ] Clone repos: `logos-execution-zone`, `logos-modules`, `lez-programs`
- [ ] Verify local LEZ sequencer runs (`RISC0_DEV_MODE=1`)
- [ ] Study existing Logos Core module examples (wallet, storage modules)

#### 1.2 Agent Core Module Skeleton
- [ ] Create module project using `logos-module-builder`
- [ ] Implement Qt Remote Objects interface (C++ glue + Rust logic via C ABI)
- [ ] Module loads into Logos Core alongside wallet/storage/messaging modules
- [ ] Agent generates its own NPK/ISK keypair on first run → owns a shielded LEZ account
- [ ] Derive Logos Messaging address from agent identity

#### 1.3 Owner Channel
- [ ] Create dedicated Logos Messaging topic (agent ↔ owner)
- [ ] Implement E2E encrypted chat over this topic
- [ ] Owner can send commands from any Logos App instance
- [ ] Agent responds with structured messages (status, confirmations, results)

#### 1.4 Spending Threshold
- [ ] Owner configures per-transaction and per-period token limits
- [ ] Below threshold → agent executes autonomously
- [ ] Above threshold → agent sends proposal to owner via chat, waits for approval
- [ ] Timeout/retry logic for unreachable owner

#### 1.5 CLI for Deployment
- [ ] `logos-agent deploy <node> --config <file>` — deploys module to a remote node
- [ ] `logos-agent fund <amount>` — sends initial LEZ tokens to agent wallet
- [ ] `logos-agent status` — checks agent health and balance
- [ ] `logos-agent configure <key> <value>` — updates runtime config

---

### Phase 2: Default Skills (Weeks 3–5)

> Goal: Implement all required skill categories with a composable skill interface.

#### 2.1 Skill Interface / SDK
- [ ] Define a `Skill` trait/interface with:
  - Name, description, parameter schema, return schema
  - `execute(params) → Result` method
  - Error isolation (a failing skill must not crash others)
- [ ] Skill registry — auto-discovers and loads skills at startup
- [ ] Documentation for third-party skill development

#### 2.2 Storage Skills
- [ ] `storage.upload(path, label)` — encrypt file → upload to Logos Storage → return CID
- [ ] `storage.download(address, path)` — fetch from Logos Storage → decrypt → save locally
- [ ] `storage.list()` — list stored files with labels and CIDs
- [ ] `storage.share(address, recipient)` — share decryption access with another identity

#### 2.3 Messaging Skills
- [ ] `messaging.send(recipient, message)` — send E2E encrypted message
- [ ] `messaging.join(group_id)` — join a group topic
- [ ] `messaging.create_group(members)` — create new group topic, invite members

#### 2.4 Blockchain / Wallet Skills
- [ ] `wallet.balance()` — query shielded balance
- [ ] `wallet.send(recipient, amount)` — transfer tokens (respects spending threshold)
- [ ] `wallet.history()` — recent transactions summary
- [ ] `program.query(program_id, params)` — read on-chain program state
- [ ] `program.call(program_id, instruction, params)` — submit transaction (threshold-gated)
- [ ] `program.deploy(binary_path)` — deploy compiled LEZ program, return program ID

#### 2.5 Meta Skills
- [ ] `meta.skills()` — list all loaded skills with schemas
- [ ] `meta.status()` — agent state: balance, storage usage, active tasks
- [ ] `meta.configure(key, value)` — runtime config updates

---

### Phase 3: A2A Coordination & Demo (Weeks 5–8)

> Goal: Multi-agent interop, use case demos, final submission.

#### 3.1 A2A-Compatible Agent Coordination
- [ ] **Agent Card**: generate A2A-schema-compliant JSON (identity, skills, schemas, LEZ price)
- [ ] **Discovery**: publish Agent Card to a Logos Messaging discovery topic; implement `agent.discover(topic)`
- [ ] **Task lifecycle**: implement submitted → working → input-required → completed/failed state machine
- [ ] **Task delegation**: `agent.task(agent_address, skill, params)` — send task, pay LEZ on acceptance
- [ ] **Streaming**: `agent.subscribe(agent_address, task_id)` — SSE-like updates over Messaging
- [ ] **Cancellation**: `agent.cancel(agent_address, task_id)` — cancel + refund logic
- [ ] Document this as an "A2A transport binding over Logos Messaging"

#### 3.2 AI / LLM Integration
- [ ] Define pluggable inference interface (local model or API-based)
- [ ] Agent can reason about incoming requests, pick skills, compose responses
- [ ] Natural language chat with owner (not just structured commands)

#### 3.3 End-to-End Use Case Demos (pick 3+)
- [ ] **Personal file vault**: owner sends file via chat → agent encrypts → stores → returns CID
- [ ] **Agent services marketplace**: agents advertise skills with LEZ prices → discover → pay → execute
- [ ] **Multi-agent workflow**: orchestrator decomposes task → delegates to specialist agents → aggregates results

#### 3.4 Testnet Deployment & Evidence
- [ ] Deploy 3 separate agents on LEZ testnet (one per skill category: Storage, Messaging, Blockchain)
- [ ] Document deployment steps (reproducible)
- [ ] Provide evidence (screenshots, logs, transaction hashes)

#### 3.5 Final Deliverables
- [ ] Public GitHub repo (MIT or Apache-2.0)
- [ ] End-to-end demo video (narrated, not silent) showing terminal output with `RISC0_DEV_MODE=0`
- [ ] Reproducible demo script (`dev.sh` or similar) against local sequencer
- [ ] CI pipeline (green on default branch)
- [ ] Write-up: architecture, skill interface, spending threshold, A2A protocol, security model, limitations

---

## Risk Assessment & Mitigations

| Risk | Impact | Mitigation |
|---|---|---|
| **16 GB RAM insufficient for RISC0_DEV_MODE=0** | 🔴 High — can't generate final demo | Use cloud VM for final proof generation; develop entirely in dev mode |
| **Qt Remote Objects learning curve** | 🟡 Medium — unfamiliar IPC paradigm | Study existing modules (wallet, chat) first; use `logos-module-builder` to generate boilerplate |
| **Logos testnet instability** | 🟡 Medium — sequencer changes break things | Pin to specific commit/version; test against local sequencer in CI |
| **A2A spec evolving** | 🟢 Low — spec is stable (v1) | Follow spec as-is; document deviations |
| **Shielded account complexity** | 🟡 Medium — ZK crypto is non-trivial | Wrap existing LEZ wallet module APIs; don't reimplement crypto |

---

## Resolved Technical Decisions

| Decision | Choice | Notes |
|---|---|---|
| **Primary language** | ✅ **Rust** | With C ABI bridge for Qt Remote Objects glue. LEZ is Rust-native, so this minimizes friction. |
| **AI/LLM backend** | ✅ **Google Gemini API** (free tier) | Start with API-based inference via Gemini free tier key. Design a pluggable `InferenceBackend` trait so we can swap in Ollama/OpenAI later. |
| **Scope strategy** | ✅ **MVP first** | Build minimum viable version to learn the stack. Improve step-by-step, commit-by-commit. No overengineering. |
| **Demo environment** | ✅ **Local PC + swap file** | Use a 16 GB+ swap file on WSL2 for `RISC0_DEV_MODE=0` proof generation. Slower but avoids cloud costs. Reduce proof segment size if needed. |

---

## MVP Scope (What We Build First)

> [!TIP]
> **Philosophy**: Get something loading and running end-to-end first, even if it's minimal. Then iterate.

The MVP targets the bare minimum to demonstrate the core loop: **owner → agent → Logos stack → response**.

### MVP Includes
1. **Module skeleton** that loads into Logos Core (Qt Remote Objects interface via C ABI)
2. **Agent identity** — generate NPK/ISK keypair, own a shielded LEZ account
3. **Owner channel** — basic E2E encrypted chat via Logos Messaging
4. **2-3 simple skills** — `wallet.balance()`, `meta.status()`, `storage.upload()` 
5. **Spending threshold** — basic per-transaction limit check
6. **CLI** — `deploy`, `status`, `fund` commands

### MVP Excludes (for later iterations)
- A2A agent-to-agent coordination
- Full skill suite (all 20+ skills)
- Multi-agent demos
- Testnet deployment (start with local sequencer)
- AI/LLM natural language reasoning (start with structured commands)
- Video demo recording
