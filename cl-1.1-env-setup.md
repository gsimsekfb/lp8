# Environment Setup Guide — LP-0008 (v3)

## Current Status

| Tool | Status | Version |
|---|---|---|
| Ubuntu (WSL2) | ✅ Installed | 22.04.5 LTS |
| Rust | ✅ Installed | 1.98.0 (stable) |
| Docker | ✅ Installed | 29.1.3 |
| RISC Zero (`cargo-risczero`) | ✅ Installed | 2.3.1 |
| `rzup` | ✅ Installed | — |
| Nix | ❌ **Not installed** | Needed for `logos-module-builder` |

### Resources

| Resource | Current | Target | Notes |
|---|---|---|---|
| WSL2 RAM | 7.6 GB (default) | **Keep as-is** | Host Windows at 94% usage — don't starve it |
| WSL2 Swap | ✅ **16 GB (DONE)** | 16 GB | Verified working |
| **C:\\ drive (real disk)** | **81 GB free** (of 953 GB) | See estimates below | WSL2's virtual disk lives on C:\\, so this is the real constraint |
| WSL2 virtual disk | Shows 923 GB (virtual max) | — | This is the VHD max size, **not** actual free space |

### Storage Estimates

| Scope | Estimated Disk Usage | What's Included |
|---|---|---|
| **MVP** | **~15 GB** | Logos repos (~2 GB), Rust build artifacts (~8 GB), RISC Zero toolchain (~2 GB), Docker images (~3 GB) |
| **Full version** | **~40 GB** | MVP + additional repos, testnet data, proof artifacts, CI cache, larger Docker images |
| **Available** | **81 GB** on C:\\ | ✅ Sufficient for both MVP and full version |

> [!NOTE]
> After all repos are cloned and built, you'd still have ~40+ GB free even for the full version. No disk concerns.

---

## Step 1: Increase WSL2 Swap ✅ DONE

Swap successfully increased to 16 GB. Verified:

```
Mem:           7.6Gi       424Mi       6.4Gi       3.0Mi       707Mi       7.0Gi
Swap:           16Gi          0B        16Gi
```

---

## Step 2: Install Nix Package Manager

```bash
# Install Nix (single-user mode, simpler for WSL2)
sh <(curl -L https://nixos.org/nix/install) --no-daemon

# Add to your shell profile
echo '. /home/gok/.nix-profile/etc/profile.d/nix.sh' >> ~/.bashrc
source ~/.bashrc

# Verify
nix --version
```

---

## Step 3: Clone Logos Repos (into `logos-repos/`)

All repos go into `/home/gok/logos-repos/`:

```bash
mkdir -p /home/gok/logos-repos
cd /home/gok/logos-repos

# Core LEZ blockchain
git clone https://github.com/logos-blockchain/logos-execution-zone.git

# LEZ programs (token, AMM, etc.)
git clone https://github.com/logos-blockchain/lez-programs.git

# Logos modules (wallet, storage, messaging, module builder)
git clone https://github.com/logos-co/logos-modules.git

# Logos Basecamp (the desktop app)
git clone https://github.com/logos-co/logos-basecamp.git
```

Expected result:
```
/home/gok/logos-repos/
├── logos-execution-zone/
├── lez-programs/
├── logos-modules/
└── logos-basecamp/
```

---

## Step 4: Verify LEZ Builds

```bash
cd /home/gok/logos-repos/logos-execution-zone

# Build in dev mode (skip ZK proofs, limit to 6 parallel jobs)
RISC0_DEV_MODE=1 cargo build -j 6
```

---

## Step 5: Study Existing Modules

```bash
# See how existing modules are structured
ls -la /home/gok/logos-repos/logos-modules/
find /home/gok/logos-repos/logos-modules -name "*.rs" -o -name "Cargo.toml" | head -30
```

---

## Execution Order

| Step | Action | Status |
|---|---|---|
| 1 | Set swap to 16 GB via `.wslconfig` | ✅ **DONE** |
| 2 | Install Nix | ⏳ Waiting for your go-ahead |
| 3 | Clone repos into `/home/gok/logos-repos/` | ⏳ Waiting for your go-ahead |
| 4 | Verify LEZ builds (`RISC0_DEV_MODE=1`) | ⏳ After step 3 |
| 5 | Explore module structure & document findings | ⏳ After step 4 |

> [!IMPORTANT]
> I will execute each step **only after you say go**. No automatic proceeding.

---

## Token Usage Note

> Regarding your question about remaining token quota: I don't have visibility into your Gemini API free tier quota or your Antigravity session token limits. For the Gemini API free tier, you can check your usage at [Google AI Studio](https://aistudio.google.com/). The free tier typically allows 15 RPM / 1M TPM / 1,500 RPD for Gemini 2.0 Flash.
