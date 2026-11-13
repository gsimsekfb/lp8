

# CheatSheet


## wsl

```sh
wsl -d Ubuntu-24.04
```

## build

```sh
cd ~/logos-lp-8/logos-agent-module

# 1. Build — compiles the module in Nix's isolated sandbox
nix build --extra-experimental-features 'nix-command flakes'
    # ✅ Success = exit code 0, `result/` symlink created
    # ❌ Failure = missing deps, Nix not installed properly

ls -l result
    # check result's timestamp, it might be from an old build 


# Incr. build / C++ syntax check
nix develop --extra-experimental-features 'nix-command flakes'
    # Enter dev shell — to access lm + logoscore binaries
cmake -B build -G Ninja -Wno-dev     # once
cmake --build build                  # each time you change code
```

## check agent_module methods respects the code changes
```
# 2. Enter dev shell — to access lm + logoscore binaries
nix develop --extra-experimental-features 'nix-command flakes'

# 3. lm — inspects the plugin, lists all methods/events
#    More on lm: Logos Module inspector - provided by logos-module-builder 
#    in the Nix dev shell - loads a compiled Logos plugin .so file and 
#    inspects/lists its metadata and methods without needing to run the full 
#    logoscore daemon.
#
lm ./result/lib/agent_module_plugin.so
    # should show setupOwnerChannel, sendToOwner etc.

echo 'alias lm="$LOGOS_MODULE_ROOT/bin/lm"' >> ~/.bashrc
    # need to use this lm

# 4. logoscore — actually RUNS the module and calls methods
    see logoscore-*.md
```

### Verify build - more detailed:

```
// double check result's timestamp, it might be from an old build 
ls -l result
    lrwxrwxrwx 1 gok gok 69 Sep 18 12:10 result -> /nix/store/imlcc0r2557v10yjbb03c21r0gmm7dc0-logos-agent_module-modul

ll /nix/store/imlcc0r2557v10yjbb03c21r0gmm7dc0-logos-agent_module-module
    total 680K
    dr-xr-xr-x   4 gok 4.0K Jan  1  1970 .
    drwxr-xr-x 518 gok 664K Sep 18 12:10 ..
    dr-xr-xr-x   2 gok 4.0K Jan  1  1970 include
    dr-xr-xr-x   2 gok 4.0K Jan  1  1970 lib

ls -lh result/lib/agent_module_plugin.so
    -r-xr-xr-x 1 gok gok 2.8M Jan  1  1970 result/lib/agent_module_plugin.so
```


###  exit NIX shell
```
exit 

echo $IN_NIX_SHELL
    # should print nothing once you're fully out.

echo $IN_NIX_SHELL
impure
    # still in, try exit again
```
