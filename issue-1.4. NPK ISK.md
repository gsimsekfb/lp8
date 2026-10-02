

## MVP 1.4 Agent Identity

> can we try to do this later? "Replace Ed25519 with LEZ NPK/ISK via wallet module API" ?
and at which step should we do it?


Yes. Best place is **Phase 2, Task 2.4 (Blockchain/Wallet Skills)** — when we implement `wallet.balance()` and `wallet.send()`. At that point we *must* integrate with the LEZ wallet module via raw LogosAPI anyway, so replacing Ed25519 with real NPK/ISK comes naturally as part of that work. No extra step needed.

===
---

> question: Why did you revert to using OpenSSL ? we were doing this: "Replace Ed25519 with LEZ NPK/ISK via wallet module API	" no?

The build failed when we tried it. The LEZ wallet module uses the **older** (non-universal) plugin model and doesn't publish a `.lidl` interface file. The code generator crashed with:

```
Interface file not found for 'logos_execution_zone':
  .../logos_execution_zone.lidl
```

The `modules().logos_execution_zone.createAccountPrivate()` typed accessor **only works when the dependency module uses the universal authoring model** and has a LIDL file. The LEZ wallet module doesn't.

**Two ways forward** (neither explored yet):

1. **Raw LogosAPI** — call `logosAPI->getClient("logos_execution_zone")` and invoke methods manually via Qt IPC. Requires Qt types in our code.
2. **Ask Logos team** — if they can add a LIDL export to the LEZ module, our typed accessor would work.

I reverted to OpenSSL to keep a working build. Want me to try the raw LogosAPI approach instead?

===  
---  

> Is there a way to test if the first option in this doc. could work w/o changing lots of code e.g. reverting last commit?
Do not code yet, just answer

Yes, but only at **runtime inside Logos Core** — not in isolation.

The test would be: add a single method like `testWalletConnection()` (alongside existing Ed25519 code, not replacing it) that tries the raw LogosAPI call. But it would only succeed when:

1. Logos Core is running
2. The wallet module is also loaded
3. Both modules are in the same runtime

We'll naturally hit this scenario in **Task 1.5 (Owner Channel)** — that's the first time we need Logos Core running with multiple modules. At that point, adding one extra test call to `createAccountPrivate()` is trivial — no revert needed, just one additional method.

Short answer: **wait for 1.5**, test it there as a side-effect, zero risk to current code.
