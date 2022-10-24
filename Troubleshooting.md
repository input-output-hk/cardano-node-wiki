## Unable to build on ARM64

When building `cardano-node` on `arm64`, the following error may occur:

```
[ 9 of 14] Compiling Cardano.Logging.Tracer.Standard ( src/Cardano/Logging/Tracer/Standard.hs, /src/cardano-node/dist-newstyle/build/aarch64-linux/ghc-8.10.4/trace-dispatcher-1.29.0/build/Cardano/Logging/Tracer/Standard.o, /src/cardano-node/dist-newstyle/build/aarch64-linux/ghc-8.10.4/trace-dispatcher-1.29.0/build/Cardano/Logging/Tracer/Standard.dyn_o )
src/Cardano/Logging/Tracer/Standard.hs:9:1: error: [-Wdeprecations, -Werror=deprecations]
    Module `Control.Concurrent.Chan.Unagi.Bounded':
      This library is unlikely to perform well on architectures without a fetch-and-add instruction
  |
9 | import           Control.Concurrent.Chan.Unagi.Bounded
  | ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
src/Cardano/Logging/Tracer/Standard.hs:73:26: error: [-Wdeprecations, -Werror=deprecations]
    In the use of `newChan'
    (imported from Control.Concurrent.Chan.Unagi.Bounded):
    "This library is unlikely to perform well on architectures without a fetch-and-add instruction"
   |
73 |     (inChan, outChan) <- newChan 2048
   |                          ^^^^^^^
```

When this happens ensure the `cabal.project.local` file exists in the `cardano-node` project root directory and contains the following lines:

```
package trace-dispatcher
  ghc-options: -Wwarn
```

## `cabal` is unable to find `libsodium` via `pkg-config` on MacOS

When the following error occurs:

```
Warning: Requested index-state 2022-10-19T00:00:00Z is newer than
'cardano-haskell-packages'! Falling back to older state
(2022-10-17T00:00:00Z).
Resolving dependencies...
Error: cabal: Could not resolve dependencies:
[__0] trying: cardano-api-1.36.0 (user goal)
[__1] next goal: cardano-crypto-class (dependency of cardano-api)
[__1] rejecting: cardano-crypto-class-2.0.0.1 (constraint from project config
/Users/jky/wrk/iohk/cardano-node/cabal.project requires <2.0.0.1)
[__1] rejecting: cardano-crypto-class-2.0.0 (conflict: pkg-config package
libsodium-any, not found in the pkg-config database)
[__1] fail (backjumping, conflict set: cardano-api, cardano-crypto-class)
After searching the rest of the dependency tree exhaustively, these were the
goals I've had most trouble fulfilling: cardano-api, cardano-crypto-class
```

It is possible that `libsodium` is installed correctly, but some other unrelated brew installed package has a corrupt `pc` file.

See https://github.com/haskell/cabal/issues/8494#issuecomment-1273802608 for details.  The offending `pc` file will need to be patched or its package uninstalled.

## Delegated stake is not earning rewards

There are a number of reasons you may not be earning rewards.  Check for the following things to ensure the testnet is set up properly:

* `d` parameter above zero: Check protocol parameters
* Pool makes blocks: There's a forge adopted metric on prometheus exporter built-in to node. One could also check logs.
* Pool meets pledge: This can be checked via `ledgerState.nesES.esLState._delegationState._pstate._pParams.<POOL_ID>._poolPledege` in the ledger dump. To get the owners' stake (to see if they meet the pledge, get the key hashes from `ledgerState.nesES.esLState._delegationState._pstate._pParams.<POOL_ID>._poolOwners`. Then maybe use the CLI to get the stake associated with each owner, and sum them.
* Rho above zero: check protocol parameters
* Reserves above zero (total supply > sum of all utxo).  Check for `ledgerState.nesES.esAccountState._reserves` in the ledger state.

## Missing blocks
There are some troubleshoot tips on how to diagnose and fix missing blocks [here](https://input-output.atlassian.net/wiki/spaces/QA/pages/2368897711/Debug+missing+block).