# LP-0008 Task List

## Phase 1: Foundation (MVP)

### 1.1 Environment Setup
- [x] Check existing Rust/toolchain setup on WSL2
- [x] Install RISC Zero toolchain (`rzup`)
- [x] Install Nix package manager
- [x] Install Docker (if not present)
- [x] Clone `logos-execution-zone` repo
- [x] Clone `logos-modules` / `logos-co` repos
- [x] Clone `lez-programs` repo
- [x] Verify local LEZ sequencer builds and runs (`RISC0_DEV_MODE=1`)
- [x] Set up 16 GB swap file for future `RISC0_DEV_MODE=0`

### 1.2 Repo Exploration & Learning
- [x] Study existing Logos Core module structure (wallet module, chat module)
- [x] Understand Qt Remote Objects ↔ C ABI bridge pattern
- [x] Understand `logos-module-builder` Nix tooling
- [x] Identify the exact APIs for: wallet, storage, messaging modules
- [x] Document findings in a research notes artifact

### 1.3 Module Skeleton
- [ ] Create agent module project (Rust + C ABI)
- [ ] Implement minimal Qt Remote Objects interface
- [ ] Module loads into Logos Core without errors
- [ ] First commit: bare skeleton

### 1.4 Agent Identity
- [ ] Generate NPK/ISK keypair on first run
- [ ] Derive Logos Messaging address from identity
- [ ] Persist keys securely
- [ ] Second commit: identity generation

### 1.5 Owner Channel (basic)
- [ ] Create dedicated Logos Messaging topic
- [ ] Send/receive plaintext messages (E2E encrypted by Messaging layer)
- [ ] Owner can send a command, agent echoes back
- [ ] Third commit: basic owner channel

### 1.6 First Skills
- [ ] `meta.status()` — return agent state
- [ ] `wallet.balance()` — query shielded balance
- [ ] Fourth commit: first skills

---

## Phase 2: Expand Skills (post-MVP)
- [ ] Skill trait/interface + registry
- [ ] Storage skills
- [ ] Messaging skills
- [ ] Full blockchain/wallet skills
- [ ] Spending threshold
- [ ] CLI tool

## Phase 3: A2A & Demo (post-MVP)
- [ ] A2A Agent Cards
- [ ] Task lifecycle
- [ ] Multi-agent coordination
- [ ] Gemini API integration
- [ ] Use case demos
- [ ] Testnet deployment
