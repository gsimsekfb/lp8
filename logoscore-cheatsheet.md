
Download `logoscore`:  
https://github.com/logos-co/logos-logoscore-cli/releases/download/v0.2.2/logoscore-x86_64-linux.tar.gz

### Human vs Machine Readable Calls
```
logoscore --human call agent_module listSkills | jq
logoscore         call agent_module listSkills
```

### Load Module and Make Module Calls
```
logoscore load-module agent_module
logoscore --human call agent_module listSkills | jq
logoscore --human call agent_module executeSkill meta.status '{}' | jq 
    # should return within ~3s
logoscore --human call agent_module executeSkill wallet.balance '{}' | jq
logoscore --human call agent_module executeSkill echo 'hello world' | jq
```

### Start
```
logoscore daemon -m ~/logoscore-modules & sleep 3
  # !! use same terminal after this cmd !!
logoscore status
logoscore stop
```

### Setup
```
cd ~/lp8/logos-agent-module
nix build --extra-experimental-features 'nix-command flakes' 
ll result   # check timestamp
ll result/  # check files
mkdir -p ~/logoscore-modules/agent_module
cp result/lib/agent_module_plugin.so manifest.json ~/logoscore-modules/agent_module/
rm -rf ~/.logoscore
    # Reset, then restart the daemon
```

