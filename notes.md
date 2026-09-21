

# CheatSheet

### build

```
// builds the module in an isolated Nix sandbox. No shell needed
nix build

// enters a dev shell with tools like lm and logoscore for testing.
nix develop 
```


```
cd /home/gok/logos-lp-8/logos-agent-module
nix build --extra-experimental-features 'nix-command flakes'
```

Verify new code changes / fns:
```
lm ./result/lib/agent_module_plugin.so
    // double check result's timestamp, it might be from an old build 
```

Verify build:

```
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

