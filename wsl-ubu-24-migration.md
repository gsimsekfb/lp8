### 1. Create the WSL

```powershell
# From Windows PowerShell
wsl --install -d Ubuntu-24.04
```

### 2. Copy into the new WSL

| What | Path | Why |
|---|---|---|
| Project repo | `~/logos-lp-8/` | Your code |
| Logos repos | `~/logos-repos/` | Reference modules |
| Nix store | (reinstall) | Can't copy — reinstall Nix |

From your **current** WSL, tar up the project:
```bash
# In Ubuntu 22.04 WSL
tar czf /tmp/logos-lp-8.tar.gz -C ~ logos-lp-8
tar czf /tmp/logos-repos.tar.gz -C ~ logos-repos
```

From **Windows**, copy between WSLs:
```powershell
cp \\wsl$\Ubuntu-22.04\tmp\logos-lp-8.tar.gz \\wsl$\Ubuntu-24.04\tmp\
cp \\wsl$\Ubuntu-22.04\tmp\logos-repos.tar.gz \\wsl$\Ubuntu-24.04\tmp\
```

In **Ubuntu 24.04** WSL:
```bash
tar xzf /tmp/logos-lp-8.tar.gz -C ~
tar xzf /tmp/logos-repos.tar.gz -C ~
```

### 3. Install dependencies

```bash
# Git
sudo apt update && sudo apt install -y git curl build-essential

# Nix
sh <(curl -L https://nixos.org/nix/install) --daemon

# Restart shell, then verify
nix --version
```

### 4. Verify everything works

```bash
# Build
cd ~/logos-lp-8/logos-agent-module
nix build --extra-experimental-features 'nix-command flakes'

# Dev shell + lm
nix develop --extra-experimental-features 'nix-command flakes'
lm ./result/lib/agent_module_plugin.so

# logoscore (should work on Ubuntu 24 — has GLIBC 2.38+)
logoscore -m ./result/lib -l agent_module -c "agent_module.getStatus()"
```

### 5. GLIBC check

```bash
ldd --version | head -1
# Ubuntu 24.04 should show GLIBC 2.39 — logoscore will work
```