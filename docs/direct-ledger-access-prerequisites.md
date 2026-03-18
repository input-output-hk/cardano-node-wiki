# Direct Ledger Access: Prerequisites, Open Questions, and Build Instructions

This document captures everything a fresh implementer needs to verify before writing code,
plus practical knowledge about the build environment and codebase conventions that aren't
derivable from the source files alone.

---

## 0. Project Layout

The working directory is `/work/`. It contains git submodules, each a separate Haskell
project with its own flake/cabal setup. Only the submodules relevant to this task are
listed here.

### Submodules relevant to this task

```
/work/
├── cardano-api/                    ← cardano-api repo (contains cardano-rpc)
│   ├── cardano-api/                  ← the cardano-api library package
│   │   └── src/Cardano/Api/          ← cardano-api source (Query, Network.IPC, Era, Consensus, etc.)
│   └── cardano-rpc/                  ← the cardano-rpc package (THIS IS THE MAIN TARGET)
│       ├── src/Cardano/Rpc/
│       │   ├── Client.hs             ← gRPC client (not modified)
│       │   ├── Proto/Api/            ← proto-lens API wrappers (not modified)
│       │   └── Server/
│       │       ├── Config.hs         ← RpcConfig (kept unchanged)
│       │       ├── Server.hs         ← runRpcServer entry point (MODIFIED)
│       │       └── Internal/
│       │           ├── Env.hs        ← RpcEnv record (MODIFIED)
│       │           ├── Monad.hs      ← Has typeclass, MonadRpc (MODIFIED)
│       │           ├── Error.hs      ← throwEither, throwExceptT (kept)
│       │           ├── Node.hs       ← getEra, getProtocolParamsJson (MODIFIED)
│       │           ├── Tracing.hs    ← TraceRpc types (MODIFIED)
│       │           ├── LedgerAccess.hs ← NEW FILE
│       │           ├── Orphans.hs    ← (not modified)
│       │           └── UtxoRpc/
│       │               ├── Query.hs  ← readParams, readUtxos, searchUtxos (MODIFIED)
│       │               ├── Submit.hs ← submitTx (MODIFIED)
│       │               ├── Type.hs   ← protobuf conversion (not modified)
│       │               └── Predicate.hs ← UTxO filtering (not modified)
│       ├── gen/                      ← generated proto-lens code (NEVER edit by hand)
│       ├── proto/                    ← .proto definitions
│       ├── test/                     ← unit tests
│       └── cardano-rpc.cabal        ← (MODIFIED — add LedgerAccess module)
│
├── cardano-node/                   ← cardano-node repo
│   ├── cardano-node/                ← the cardano-node package
│   │   ├── src/Cardano/Node/
│   │   │   ├── Run.hs              ← node startup, withAsync runRpcServer (MODIFIED)
│   │   │   ├── Queries.hs          ← existing nkQueryLedger pattern (reference only)
│   │   │   ├── Rpc/
│   │   │   │   └── LedgerAccess.hs ← NEW FILE — mkLedgerAccess from NodeKernel
│   │   │   └── Tracing/Tracers/
│   │   │       └── Rpc.hs          ← LogFormatting/MetaTrace instances (MODIFIED)
│   │   └── cardano-node.cabal      ← (MODIFIED — add new module)
│   ├── cardano-testnet/             ← integration test infrastructure
│   │   └── test/Cardano/Testnet/Test/Rpc/ ← gRPC integration tests (not modified)
│   └── @worktree/add-grpc-interface/ ← worktree for the gRPC feature branch
│
├── ouroboros-consensus/            ← consensus repo (READ ONLY — verify APIs here)
│   ├── ouroboros-consensus/src/ouroboros-consensus/Ouroboros/Consensus/
│   │   ├── Ledger/Query.hs         ← answerQuery function
│   │   ├── Ledger/Tables/          ← MapKind, LedgerTables definitions
│   │   ├── Storage/ChainDB/API.hs  ← getReadOnlyForkerAtPoint, getCurrentLedger
│   │   ├── Storage/LedgerDB/       ← Forker, BackingStore
│   │   ├── Mempool/API.hs          ← addLocalTxs
│   │   └── MiniProtocol/LocalStateQuery/Server.hs ← existing N2C query server (reference)
│   ├── ouroboros-consensus-diffusion/src/.../
│   │   ├── NodeKernel.hs           ← NodeKernel type definition
│   │   └── Network/NodeToClient.hs ← existing N2C wiring (reference)
│   └── ouroboros-consensus-cardano/src/shelley/.../
│       └── Shelley/Ledger/Query.hs ← Shelley-specific query answering
│
├── cardano-ledger/                 ← ledger repo (READ ONLY)
│   └── eras/shelley/impl/src/Cardano/Ledger/Shelley/LedgerState.hs
│
├── cardano-node-wiki/              ← documentation (THIS DOCUMENTATION SET)
│   └── docs/
│       ├── ADR-018-cardano-rpc-grpc-server.md
│       ├── ADR-019-direct-ledger-access-for-cardano-rpc.md
│       ├── direct-ledger-access-implementation-plan.md
│       ├── direct-ledger-access-full-analysis.md
│       └── direct-ledger-access-prerequisites.md  ← THIS FILE
│
└── ouroboros-network/              ← network repo (READ ONLY — RemoteAddress, etc.)
```

### Key relationships

- **cardano-rpc** depends on **cardano-api** (types, re-exports) but NOT on consensus
- **cardano-node** depends on both **cardano-rpc** and **ouroboros-consensus**
- The `LedgerAccess` record lives in **cardano-rpc** (cardano-api types only)
- The `mkLedgerAccess` implementation lives in **cardano-node** (knows consensus types)
- This is dependency inversion: cardano-rpc defines the interface, cardano-node implements it

### Path convention in the implementation plan

The implementation plan uses paths relative to `/work/`:
- `cardano-api/cardano-rpc/src/...` means `/work/cardano-api/cardano-rpc/src/...`
- `cardano-node/cardano-node/src/...` means `/work/cardano-node/cardano-node/src/...`

---

## 1. Consensus API Signatures to Verify

These functions are referenced in the implementation plan but their exact signatures were
**not verified** against the pinned `ouroboros-consensus ^>=0.30` in the cardano-node cabal.
Before writing `mkLedgerAccess`, read each of these and confirm the types match expectations.

### 1.1 `answerQuery`

**Expected location:** `Ouroboros.Consensus.Ledger.Query`

**Expected signature (approximate):**
```haskell
answerQuery
  :: ExtLedgerCfg blk
  -> ReadOnlyForker m (ExtLedgerState blk)
  -> Query blk result
  -> m result
```

**Verify:**
- Does it take `ExtLedgerCfg` or `TopLevelConfig`? (The analysis says `ExtLedgerCfg (getTopLevelConfig nk)`)
- Does it take a `ReadOnlyForker` directly or a different forker type?
- What is the exact `Query blk result` type — is it the same as what `toConsensusQuery` produces?
- Does it handle all three footprints (`QFNoTables`, `QFLookupTables`, `QFTraverseTables`) internally?

**File to read:** `/work/ouroboros-consensus/ouroboros-consensus/src/ouroboros-consensus/Ouroboros/Consensus/Ledger/Query.hs`

### 1.2 `getReadOnlyForkerAtPoint`

**Expected location:** `Ouroboros.Consensus.Storage.ChainDB.API`

**Expected signature (approximate):**
```haskell
getReadOnlyForkerAtPoint
  :: ChainDB m blk
  -> ResourceRegistry m
  -> Target (Point blk)
  -> m (Either GetForkerError (ReadOnlyForker' m blk))
```

**Verify:**
- Is `Target` from `Ouroboros.Consensus.Block` or elsewhere?
- Is `VolatileTip` a constructor of `Target`?
- What is `GetForkerError` — what cases does it have?
- Is it `ReadOnlyForker'` or `ReadOnlyForker`? (The `'` suffix matters)
- What does `roforkerClose` look like — is it a field or a standalone function?

**File to read:** `/work/ouroboros-consensus/ouroboros-consensus/src/ouroboros-consensus/Ouroboros/Consensus/Storage/ChainDB/API.hs`

### 1.3 `toConsensusQuery` / `fromConsensusQueryResult`

**Expected location:** `Cardano.Api.Query.Internal.Type.QueryInMode`

**Verify:**
- Are these actually exported from that module?
- What is the return type of `toConsensusQuery`? Is it `Some (Query blk)` or something else?
- Does `fromConsensusQueryResult` need the original `QueryInMode` as input (for type-level dispatch)?
- Does `QueryCurrentEra` work through `toConsensusQuery`, or does it need special handling via `QueryHardFork GetCurrentEra`?

**File to read:** `/work/cardano-api/cardano-api/src/Cardano/Api/Query/Internal/Type/QueryInMode.hs`

### 1.4 `toConsensusGenTx` / `fromConsensusApplyTxErr`

**Expected location:** `Cardano.Api.Consensus.Internal.InMode` or `Cardano.Api.Network.IPC.Internal`

**Verify:**
- Which exact module are these in? The plan says two different locations.
- Is it `toConsensusGenTx` or some other name? Check exports.
- Does `fromConsensusApplyTxErr` produce `TxValidationErrorInCardanoMode`?

**Files to check:**
- `/work/cardano-api/cardano-api/src/Cardano/Api/Consensus/Internal/InMode.hs`
- `/work/cardano-api/cardano-api/src/Cardano/Api/Network/IPC/Internal/Monad.hs`
- Grep for `toConsensusGenTx` across `cardano-api/src/`

### 1.5 `addLocalTxs` (Mempool API)

**Expected location:** `Ouroboros.Consensus.Mempool.API`

**Expected usage:**
```haskell
result <- addLocalTxs (getMempool nk) (Solo genTx)
```

**Verify:**
- Is it `addLocalTxs` or `addTx` or `addLocalTx`? (The name may have changed)
- Does it take `Solo` (from `ghc-prim`) or `MkSolo` or something else?
- What does the result look like — `MempoolAddTxResult` with `MempoolTxAdded` / `MempoolTxRejected`?
- What is the exact error type in `MempoolTxRejected`?

**File to read:** `/work/ouroboros-consensus/ouroboros-consensus/src/ouroboros-consensus/Ouroboros/Consensus/Mempool/API.hs`

### 1.6 `withRegistry`

**Expected location:** `Ouroboros.Consensus.Util.ResourceRegistry`

**Verify:**
- Is it `withRegistry` that takes `\reg -> ...` or does it have a different name?
- Is the import `Ouroboros.Consensus.Util.ResourceRegistry` correct?

**File to read:** Check the actual module name via grep for `withRegistry` in ouroboros-consensus.

### 1.7 `NodeKernel` type parameters

**Expected:**
```haskell
NodeKernel IO RemoteAddress LocalConnectionId (CardanoBlock StandardCrypto)
```

**Verify:**
- Where do `RemoteAddress` and `LocalConnectionId` come from? (likely `ouroboros-network`)
- Check the actual type alias used in `cardano-node` — it might use a different concrete type.
- Look at how `rnNodeKernelHook` receives the `nodeKernel` parameter in `Run.hs` for the actual type.

---

## 2. Nix Build Instructions

### The git submodule problem

`/work` contains git submodules. Each submodule's `.git` file points to a parent modules
directory that doesn't exist in this environment. This breaks `nix build .#...` because
nix's git fetcher fails.

### Workaround: use `path:` instead of `.`

```bash
# In cardano-api submodule:
cd /work/cardano-api
nix build 'path:/work/cardano-api#cardano-rpc:lib:cardano-rpc' \
  --allow-import-from-derivation \
  --accept-flake-config

# In cardano-node submodule:
cd /work/cardano-node
nix build 'path:/work/cardano-node#cardano-node:lib:cardano-node' \
  --allow-import-from-derivation \
  --accept-flake-config
```

### Required flags (always)

- `--allow-import-from-derivation` -- haskell.nix needs IFD to enumerate packages
- `--accept-flake-config` -- to trust the flake's nixConfig settings

### Package names (haskell.nix component style)

| Package | Nix attribute |
|---------|---------------|
| cardano-rpc library | `cardano-rpc:lib:cardano-rpc` |
| cardano-rpc gen library | `cardano-rpc:lib:gen` |
| cardano-rpc tests | `cardano-rpc:test:cardano-rpc-test` |
| cardano-api library | `cardano-api:lib:cardano-api` |
| cardano-node library | `cardano-node:lib:cardano-node` |

### Listing all packages

```bash
nix eval 'path:/work/cardano-api#packages.x86_64-linux' --apply 'builtins.attrNames' \
  --allow-import-from-derivation --accept-flake-config
```

### Running code generation (proto-lens)

Never manually edit files in `gen/`. Use:
```bash
nix develop --command bash -c "cd cardano-rpc && buf generate proto"
```

---

## 3. Codebase Conventions and Gotchas

### RIO import hiding

RIO re-exports most of Prelude but **hides some common functions**:
- `sortBy` -- import from `Data.List`
- `on` -- import from `Data.Function` or `Data.Ord`
- `toList` -- import from `GHC.IsList` (not `Data.Foldable`)

Always check what RIO re-exports before assuming a function is in scope.

### Proto wrapper: `Proto msg`

`Proto` is a grapesy newtype wrapper. Rules:
- Handler signatures use `Proto` in parameters and return types
- Internal functions use **plain proto-lens types** (unwrapped)
- Use `getProto` / `fmap getProto` only at the RPC handler boundary
- Never wrap intermediate values in `Proto`

### `Inject` typeclass

After removing `import Cardano.Api` from `Monad.hs`, the `Inject` typeclass (used by
`putTrace`) must be imported explicitly:
```haskell
import Cardano.Api.Era (Inject (..))
```

### hlint preferences

- Prefer backtick-infix sections over lambdas: `` (`f` y) `` not `\x -> f x y`
- Don't mix styles in the same function
- hlint will catch these

### GHC version

The project uses GHC 9.10 (from the `inotifywait.sh` commit: "bump ghc to 9.10").

### `-Wunused-packages` is enabled

The cabal file has `-Wunused-packages`. If you remove an import that was the only use of a
dependency, the build will fail. Conversely, don't add deps without checking if they're
already transitively available.

### `-Wredundant-constraints` is enabled

Don't add typeclass constraints that aren't actually used. The `IsEra` constraint mistake
in the past was caught by this.

---

## 4. Subtle Implementation Details

### 4.1 The `eon` existential must escape `withLedgerAccess`

The `eon` value (from `forEraInEon @Era era ...`) is needed after the `withLedgerAccess`
callback for protobuf conversion. The callback must return it as part of its result tuple:

```haskell
-- CORRECT: eon escapes the callback
(pparams, chainPoint, blockNo, eon) <- liftIO $ withLedgerAccess laRef $ \la -> do
  AnyCardanoEra era <- laDetermineEra la
  eon <- forEraInEon @Era era (error "Minimum Conway era required") pure
  ...
  pure (pparams, chainPoint, blockNo, eon)

-- WRONG: eon would be scoped inside the callback only
liftIO $ withLedgerAccess laRef $ \la -> do
  ...
  -- can't use eon out here for protobuf conversion
```

This works because `eon` is an existential that's pattern-matched later. The tuple captures
the existential witness.

### 4.2 `RankNTypes` in `LedgerAccess`

The `laQueryPParams` and `laQueryUtxo` fields have `forall era.` quantification:
```haskell
laQueryPParams :: forall era. ShelleyBasedEra era -> IO (Ledger.PParams (ShelleyLedgerEra era))
```

This requires `{-# LANGUAGE RankNTypes #-}` on `LedgerAccess.hs`. Callers can instantiate
`era` freely (it's determined by the `ShelleyBasedEra` witness passed in).

### 4.3 `throwEither` / `throwExceptT` still needed in Submit.hs

After the rewrite, `Query.hs` and `Node.hs` no longer need `throwExceptT` or the double
`throwEither` pattern (because `LedgerAccess` callbacks throw directly on error). But
`Submit.hs` still uses `throwEither` via `putTraceThrowEither` for tx validation errors.
Don't remove the `Error` module import from `Submit.hs`.

### 4.4 `SomeException` import in Tracing.hs

After removing `TraceRpcSubmitN2cConnectionError SomeException`, the `Control.Exception`
import for `SomeException` is **still needed** because `TraceRpc` (defined in the same
module) uses `SomeException` in `TraceRpcError` and `TraceRpcFatalError`.

### 4.5 The `SearchUtxos` tracing gap (pre-existing bug)

In `cardano-node/src/Cardano/Node/Tracing/Tracers/Rpc.hs`, `TraceRpcQuerySearchUtxosSpan`
is **missing from `forMachine`**, `asMetrics`, and the `MetaTrace` instance. This is a
pre-existing bug. The implementation plan (step 11) includes fixing this.

### 4.6 `Net.Tx.SubmitFail` / `Net.Tx.SubmitSuccess` still needed

After the rewrite, `Submit.hs` still pattern-matches on `SubmitFail` / `SubmitSuccess`
from `Cardano.Api.Network.IPC`. Don't remove that qualified import.

### 4.7 Config.hs backward compatibility

The plan **keeps** `nodeSocketPath` in `RpcConfig`. The earlier design iteration wanted to
remove it, but analysis showed it would break:
- `cardano-node/cardano-testnet/src/Testnet/Types.hs` (line 155: `nodeRpcSocketPath`)
- `cardano-node/cardano-node/src/Cardano/Node/Configuration/POM.hs`
- `makeRpcConfig` uses it to derive the default `rpcSocketPath`

The field just isn't used at runtime for N2C connections anymore.

### 4.8 Two `withLedgerAccess` calls in Submit.hs

The submit method needs two separate `withLedgerAccess` calls:
1. First for `laDetermineEra` (to get the era for tx deserialization)
2. Second for `laSubmitTx` (after the tx is deserialized and validated)

This is because tx deserialization happens between the two calls and is a pure/monadic
operation that doesn't need `LedgerAccess`.

### 4.9 Era mismatch in `mkLedgerAccess` result unwrapping

`fromConsensusQueryResult` for era-specific queries (pparams, utxo) returns
`Either EraMismatch result`. The `mkLedgerAccess` implementation must unwrap this:

```haskell
laQueryPParams = \sbe -> do
  raw <- runQuery (QueryInEra (QueryInShelleyBasedEra sbe QueryProtocolParameters))
  case raw of
    Left eraMismatch -> throwIO (userError $ "Era mismatch: " <> show eraMismatch)
    Right pparams    -> pure pparams
```

An era mismatch means the node transitioned to a new era between `laDetermineEra` and the
actual query. This is a rare race condition at hard fork boundaries. Throwing is acceptable
since the client can retry.

### 4.10 `QueryCurrentEra` may not route through `toConsensusQuery`

The implementation plan's risk table flags this: `QueryCurrentEra` might need to be
constructed directly as `BlockQuery (QueryHardFork GetCurrentEra)` rather than going through
`toConsensusQuery`. Verify by reading `toConsensusQuery` in
`/work/cardano-api/cardano-api/src/Cardano/Api/Query/Internal/Type/QueryInMode.hs`.

If it's not supported, `laDetermineEra` in `mkLedgerAccess` needs to use the consensus
query type directly instead of the cardano-api wrapper.

### 4.11 `runQuery` helper must use `bracket` for forker lifecycle

The implementation plan's code snippet shows:
```haskell
forker <- getReadOnlyForkerAtPoint ...
result <- answerQuery cfg forker cQuery
roforkerClose forker   -- NOT exception-safe!
```

This leaks the forker if `answerQuery` throws. Must use `bracket`:
```haskell
bracket
  (getReadOnlyForkerAtPoint chainDB reg VolatileTip >>= either (throwIO . mkError) pure)
  roforkerClose
  (\forker -> answerQuery cfg forker cQuery >>= convertResult)
```

This is noted in the risk table but contradicts the code snippet. Use `bracket`.

### 4.12 IORef thread safety during startup

The `IORef (Maybe LedgerAccess)` has one writer (`rnNodeKernelHook`) and multiple readers
(gRPC handler threads). This is safe because:
- GHC's `IORef` guarantees atomicity for single-word writes
- The write is `Nothing -> Just la` (a single pointer write)
- Readers see either `Nothing` (return UNAVAILABLE) or `Just la` (proceed)
- No read-modify-write cycle exists

There is NO race condition here. But if someone later adds a second write site, this
assumption breaks. Document it in a comment in `Run.hs`.

### 4.13 `Solo` constructor name on GHC 9.10

GHC 9.10 uses `MkSolo` as the constructor (renamed from `Solo` in earlier GHC versions).
The mempool submission code needs:
```haskell
import GHC.Tuple (Solo(MkSolo))
-- or just use Solo pattern if the re-export works
```

Verify which constructor name is in scope. Check how existing cardano-node code uses `Solo`
(grep for `Solo` in the cardano-node source).

### 4.14 `getReadOnlyForkerAtPoint` takes `ResourceRegistry` from where?

Each call to `getReadOnlyForkerAtPoint` needs a `ResourceRegistry`. The implementation plan
says to use `withRegistry` (from ouroboros-consensus) to create a fresh one per query. This
matches the pattern used by the existing N2C LocalStateQuery server in
`ouroboros-consensus-diffusion/Network/NodeToClient.hs`.

However, verify that `withRegistry` cleans up the forker automatically if the registry is
closed before `roforkerClose` is called. If the forker is registered in the registry,
`withRegistry` handles cleanup. If not, you need explicit `bracket`.

### 4.15 The `asType` in `deserialiseTx` comes from where?

In `Submit.hs`, the existing code uses `asType` (line 52):
```haskell
deserialiseTx sbe = shelleyBasedEraConstraints sbe $ deserialiseFromCBOR asType
```

After the rewrite, this code is unchanged but `asType` comes from `Cardano.Api`
(re-exported by the blanket `import Cardano.Api`). If you change the `Cardano.Api` import
to be more specific, make sure `AsType` / `asType` is still in scope. It's from
`Cardano.Api.Serialise.SerialiseUsing` or `Cardano.Api.Serialise.Cbor`.

### 4.16 `NoFieldSelectors` on `Env.hs`

`Env.hs` has `{-# LANGUAGE NoFieldSelectors #-}`. This means `RpcEnv` fields like
`rpcLedgerAccess` are NOT available as accessor functions. The `Has` instance in `Monad.hs`
must use `NamedFieldPuns`:

```haskell
-- This works (NamedFieldPuns):
instance Has (IORef (Maybe LedgerAccess)) RpcEnv where
  obtain RpcEnv{rpcLedgerAccess} = rpcLedgerAccess

-- This does NOT work (NoFieldSelectors blocks it):
instance Has (IORef (Maybe LedgerAccess)) RpcEnv where
  obtain = rpcLedgerAccess  -- ERROR: not a function
```

The existing `Has LocalNodeConnectInfo RpcEnv` instance already uses `NamedFieldPuns`, so
just follow the same pattern.

### 4.17 Removing `import Cardano.Api` is high-risk

Three files (Env.hs, Monad.hs, Server.hs) currently have blanket `import Cardano.Api` which
re-exports hundreds of names. The plan replaces these with specific imports. This is the
most likely source of compilation errors because it's easy to miss a name that was silently
in scope.

**Strategy:** For each file where `import Cardano.Api` is removed:
1. Remove the import
2. Try to build
3. Add specific imports for each "not in scope" error
4. Repeat until clean

Files that **keep** `import Cardano.Api` (because they use many names from it):
- `Query.hs` — uses `AnyCardanoEra`, `forEraInEon`, `Era`, `convert`, `ShelleyBasedEra`,
  `UTxO`, `TxIn`, `TxIx`, `TxOut`, `CtxUTxO`, `QueryUTxOFilter`, `ChainPoint`, `BlockNo`,
  `WithOrigin`, `serialiseToRawBytesHexText`, `serialiseToRawBytes`, `deserialiseFromRawBytes`,
  `AsTxId`, `IsEra`, `obtainCommonConstraints`, `fromList`, etc.
- `Submit.hs` — uses `AnyCardanoEra`, `forEraInEon`, `ShelleyBasedEra`, `Tx`, `TxId`,
  `TxInMode`, `shelleyBasedEraConstraints`, `deserialiseFromCBOR`, `getTxId`, `getTxBody`,
  `serialiseToRawBytes`, etc.
- `Node.hs` — uses `AnyCardanoEra`, `forEraInEon`, `Era`, `convert`, `obtainCommonConstraints`, etc.

For these files, **keep `import Cardano.Api`** and just add the `LedgerAccess` import alongside it.

---

## 5. Directory layout and worktrees

### Main checkout paths

The cardano-node files are available in the main checkout:
```
/work/cardano-node/cardano-node/src/Cardano/Node/Run.hs
/work/cardano-node/cardano-node/src/Cardano/Node/Tracing/Tracers/Rpc.hs
/work/cardano-node/cardano-node/cardano-node.cabal
```

The cardano-rpc files are in the cardano-api submodule:
```
/work/cardano-api/cardano-rpc/src/Cardano/Rpc/Server/...
/work/cardano-api/cardano-rpc/cardano-rpc.cabal
```

### Worktree for the gRPC feature branch

A worktree exists at `/work/cardano-node/@worktree/add-grpc-interface/`. This is the
branch where the gRPC feature was originally developed. The analysis file references
files in this worktree path — they may differ from the main checkout if the branch
has been merged or rebased.

**When implementing:** Check whether to work in the main checkout or the worktree,
depending on the current git branch state. Use `git branch` and `git log` to determine
which is current.

### Worktree rules (from AGENTS.md)

Worktrees ALWAYS reside in each subproject's `@worktree/` directory. After creating a
git worktree inside a submodule, update its `.git` configuration to use relative paths.
`git worktree add` writes absolute paths in submodules in two places that must both be
fixed:
1. `<worktree>/.git` -- the `gitdir:` line pointing to the worktree metadata
2. `.git/modules/<submodule>/worktrees/<name>/gitdir` -- the back-pointer to the worktree

---

## 6. Verification Checklist

After implementing all steps:

- [ ] `nix build 'path:/work/cardano-api#cardano-rpc:lib:cardano-rpc' --allow-import-from-derivation --accept-flake-config`
- [ ] `nix build 'path:/work/cardano-api#cardano-rpc:test:cardano-rpc-test' --allow-import-from-derivation --accept-flake-config`
- [ ] `nix build 'path:/work/cardano-node#cardano-node:lib:cardano-node' --allow-import-from-derivation --accept-flake-config` (adjust path for worktree)
- [ ] No `-Wunused-packages` warnings
- [ ] No `-Wredundant-constraints` warnings
- [ ] hlint passes (backtick-infix, no mixed styles)
- [ ] Integration tests pass (testnet gRPC tests exercise the full path)
- [ ] Startup window test: gRPC query before kernel init returns `UNAVAILABLE`

---

## 7. Files in this documentation set

| File | Purpose |
|------|---------|
| `ADR-019-direct-ledger-access-for-cardano-rpc.md` | Architecture Decision Record |
| `direct-ledger-access-implementation-plan.md` | Step-by-step implementation plan with before/after code |
| `direct-ledger-access-full-analysis.md` | Complete analysis from 3 conversation sessions + 7 subagent explorations |
| `direct-ledger-access-prerequisites.md` | **This file** -- open questions, build instructions, gotchas |
