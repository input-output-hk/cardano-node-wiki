# Direct Ledger Access for cardano-rpc: Complete Analysis Transcript

This document compiles all substantive analysis from three conversation sessions and their
subagent explorations. It covers the full investigation of cardano-rpc's current architecture,
the consensus/UTxO-HD internals, and the design of the direct ledger access replacement for
Node-to-Client IPC.

---

## Table of Contents

1. [Conversation 1: Deep Codebase Exploration](#conversation-1-deep-codebase-exploration)
   - [1.1 cardano-rpc Implementation Overview](#11-cardano-rpc-implementation-overview)
   - [1.2 Data Path: gRPC to Ledger State](#12-data-path-grpc-to-ledger-state)
   - [1.3 UTxO-HD Architecture Impact](#13-utxo-hd-architecture-impact)
   - [1.4 High-Level Design: Direct Ledger State Access](#14-high-level-design-direct-ledger-state-access)
   - [1.5 Subagent: Comprehensive Implementation Plan](#15-subagent-comprehensive-implementation-plan)
   - [1.6 Subagent: LedgerTables Type Family System](#16-subagent-ledgertables-type-family-system)
   - [1.7 Subagent: UTxO-HD Architecture Report](#17-subagent-utxo-hd-architecture-report)
   - [1.8 Subagent: LocalStateQuery Server-Side Protocol](#18-subagent-localstatequery-server-side-protocol)
   - [1.9 Subagent: Ledger State In-Memory Architecture](#19-subagent-ledger-state-in-memory-architecture)
   - [1.10 Subagent: cardano-rpc Implementation Details](#110-subagent-cardano-rpc-implementation-details)
   - [1.11 Subagent: ADR-018 and Documentation](#111-subagent-adr-018-and-documentation)
2. [Conversation 2: Implementation Attempt and Refined Plan](#conversation-2-implementation-attempt-and-refined-plan)
   - [2.1 Implementation Observations](#21-implementation-observations)
   - [2.2 Final Refined Plan](#22-final-refined-plan)
3. [Conversation 3: Documentation and Plan Writing](#conversation-3-documentation-and-plan-writing)
   - [3.1 Architecture Pattern Discussion](#31-architecture-pattern-discussion)
   - [3.2 Alternatives Considered](#32-alternatives-considered)
   - [3.3 Key Findings from Previous Transcripts](#33-key-findings-from-previous-transcripts)

---

## Conversation 1: Deep Codebase Exploration

Session: `375109c6-497e-43aa-8030-0f1ab95c5d86`

### 1.1 cardano-rpc Implementation Overview

**Three gRPC services** defined in proto files:

| Service | Methods | Proto |
|---------|---------|-------|
| **Node** (custom) | `GetEra`, `GetProtocolParamsJson` | `cardano/rpc/node.proto` |
| **QueryService** (UTxO RPC) | `ReadParams`, `ReadUtxos`, `SearchUtxos` | `utxorpc/v1beta/query/query.proto` |
| **SubmitService** (UTxO RPC) | `SubmitTx` | `utxorpc/v1beta/submit/submit.proto` |

**Architecture:**
- `RpcEnv` holds config, tracer, and node connection info -- injected via `Has` typeclass + `MonadRpc` constraint
- Server runs on a **Unix domain socket** (insecure/local IPC), default `rpc.sock` next to the node socket
- Queries the node via standard Node-to-Client local state queries
- ~2000 lines across 16 modules + generated proto-lens code in `gen/`

**Key modules:**
- `Server.hs` -- entry point, registers method groups, top-level exception handler
- `Type.hs` (~600 lines) -- bidirectional conversions between cardano-api/ledger types and protobuf
- `Predicate.hs` -- UTxO pattern matching (address, asset, composite predicates) with address extraction optimization
- `Query.hs` -- pagination (token = `TxId#OutputIndex`, default 100 items), deterministic sort by TxIn
- `Submit.hs` -- CBOR deserialization -> era validation -> node submission

**Node integration** (in the worktree):
- `--grpc-enable` / `--grpc-socket-path` CLI flags
- `EnableRpc` / `RpcSocketPath` config keys
- Runs concurrently via `withAsync` in `Cardano.Node.Run`
- Structured tracing with metrics (`rpc.request.QueryService.*`, etc.)

**Integration tests** in `cardano-testnet/test/`:
- `hprop_rpc_query_pparams` -- validates all 44 protocol params match ledger
- `hprop_rpc_transaction` -- full round-trip: fetch UTxOs -> build tx -> submit via RPC -> confirm on-chain

---

### ADR-018 (cardano-node-wiki)

**Status:** Proposed (2026-03-11)

**Key decisions:**
- Implements the **UTxO RPC** standard (`utxorpc.org`) plus custom Cardano extensions
- **Opt-in and experimental** -- must be explicitly enabled
- Unix socket keeps the same local-access-only security model as the existing node socket
- Remote access via **Envoy reverse proxy** with TLS, rate limiting, and mTLS auth

**Current limitations acknowledged:**
- Serialization overhead (CBOR encode/decode round-trips)
- Connection-per-request cost (Ouroboros handshake)
- Sequential mini-protocol bottleneck

**Planned improvements:**
- Direct ledger state access (in-memory TVar/STM reads, bypassing IPC)
- Connection pooling over persistent IPC connections

**Dependencies:** `grapesy`, `grpc-spec`, `proto-lens`

---

### 1.2 Data Path: gRPC to Ledger State

cardano-rpc does **not** read ledger state directly. It acts as a **Node-to-Client (N2C) IPC client** that connects to the same node it's running inside, via a Unix domain socket.

#### The connection chain

```
gRPC client
  -> cardano-rpc server (Unix socket: rpc.sock)
    -> cardano-api IPC layer (Unix socket: node.sock)
      -> cardano-node's Ouroboros mini-protocol server
        -> in-memory ledger state
```

#### Step by step

**1. Fresh connection per request** (`Env.hs:16-29`)

`RpcEnv` holds a `LocalNodeConnectInfo` (socket path + network magic + consensus mode params). The TODO on line 19 confirms: there's currently **one connection per RPC request** -- no connection pooling yet.

**2. Era determination** -- every handler starts with:
```haskell
AnyCardanoEra era <- liftIO . throwExceptT $ determineEra nodeConnInfo
```
This opens a N2C connection and runs the `QueryCurrentEra` mini-protocol query.

**3. Local State Query mini-protocol** (`Query.hs:52`, `Node.hs:47`)

The core mechanism is `executeLocalStateQueryExpr` from `cardano-api`. This function (`IPC/Internal/Monad.hs:71-93`):
1. Opens a **new** N2C connection via `connectToLocalNodeWithVersion` (Ouroboros network layer -> `Net.connectTo` on the local Unix socket)
2. Negotiates the Node-to-Client protocol version
3. Runs the **LocalStateQuery** mini-protocol:
   - Sends `MsgAcquire` targeting `VolatileTip` (latest ledger state)
   - The node acquires a **read snapshot** of ledger state at the tip
   - Sends `MsgQuery` for each query (protocol params, UTxOs, chain point, block number)
   - Receives results
   - Sends `MsgRelease` then `MsgDone`

**4. Specific queries used:**

| RPC Method | Ouroboros Queries |
|---|---|
| `ReadParams` | `queryProtocolParameters` + `queryChainPoint` + `queryChainBlockNo` |
| `ReadUtxos` | `queryUtxo` (by TxIn set or whole) + chain point + block no |
| `SearchUtxos` | `queryUtxo` (by address set or whole) + chain point + block no, then **client-side** predicate filtering + pagination |
| `GetProtocolParamsJson` | `queryProtocolParameters` |
| `SubmitTx` | Uses `submitTxToNodeLocal` -- the **LocalTxSubmission** mini-protocol instead |

**5. Query optimization in SearchUtxos** (`Query.hs:104-106`)

Before querying, `extractAddressesFromPredicate` inspects the predicate tree. If addresses can be extracted, it uses `QueryUTxOByAddress` (indexed lookup in the ledger). Otherwise it falls back to `QueryUTxOWhole` (fetches entire UTxO set -- expensive).

#### Key implications

- **Every RPC request opens a new Ouroboros connection** -- full handshake + protocol negotiation each time
- **Queries are sequential** within a connection (the LocalStateQuery protocol is inherently sequential)
- **All data passes through CBOR serialization** -- the node serializes ledger state into CBOR over the socket, cardano-api deserializes it, then cardano-rpc re-serializes to protobuf
- **The node acquires a consistent snapshot** at `VolatileTip` -- so protocol params, UTxOs, and chain point within one `executeLocalStateQueryExpr` call are all from the same ledger state

This is the architecture the ADR acknowledges as having overhead, with the planned improvement being **direct ledger state access** via TVar/STM reads (bypassing IPC entirely).

---

### 1.3 UTxO-HD Architecture Impact

UTxO-HD fundamentally changes what "direct access" means for UTxO data.

The key insight is that **the UTxO set is no longer in the in-memory ledger state**. When you call:

```haskell
atomically (ChainDB.getCurrentLedger chainDB)
  :: STM IO (ExtLedgerState blk EmptyMK)
```

That `EmptyMK` is literal -- the `shelleyLedgerTables` field is `EmptyMK`, meaning **no UTxO data**. The UTxO lives in a backing store (either an in-memory `TVar` map or an on-disk LSM tree), accessed only through `ReadOnlyForker`.

#### The three tiers of data access

**Tier 1: Pure ledger state (EmptyMK) -- free, STM**
- Protocol parameters (`getPParams`), epoch number, stake distribution, chain point, block number
- These live in `NewEpochState` which is fully in-memory regardless of UTxO-HD
- Access: `atomically (getCurrentLedger chainDB)` -> pure extraction
- No forker needed, no IO, no disk

**Tier 2: Point lookups (KeysMK -> ValuesMK) -- cheap, needs forker**
- `GetUTxOByTxIn`: look up specific TxIns
- Access: `roforkerReadTables forker (LedgerTables (KeysMK txinSet))`
- O(m log n) -- goes to backing store for the specific keys
- On V2/LSM: disk reads for specific keys
- On V1/InMemory: map lookups in TVar

**Tier 3: Table traversals (RangeRead) -- expensive, needs forker + iteration**
- `GetUTxOByAddress`, `GetUTxOWhole`: full or filtered scan
- Access: loop calling `roforkerRangeReadTables` with `QueryBatchSize` (default 100k entries per batch)
- O(n) -- must scan entire backing store, applying a predicate filter per batch
- On V2/LSM: sequential range reads through LSM tree levels, deserializing `TxOutBytes` back to `Core.TxOut era`
- The existing consensus code in `answerShelleyTraversingQueries` does exactly this loop

#### What this means for the direct access design

**For Tier 1 (ReadParams pparams):** The win is massive and simple. Replace the entire N2C round-trip with a single STM read. No forker, no table access, no disk. This is the easy win.

**For Tier 2 (ReadUtxos by TxIn):** The win is real but more nuanced. You skip the N2C + CBOR overhead, but you still need:
1. Acquire a `ReadOnlyForker` via `ChainDB.getReadOnlyForkerAtPoint` (allocates resources)
2. Call `roforkerReadTables` (may hit disk on LSM backend)
3. Close the forker

The backing store IO is the same cost either way -- you just eliminate the IPC + serialization wrapper. For small lookups this is a big relative improvement.

**For Tier 3 (SearchUtxos by address / whole UTxO):** The most interesting case. Currently:
- N2C server does the full range-read loop internally, accumulates the entire result, CBOR-serializes the whole `UTxO era`, sends it over the socket
- cardano-api deserializes the whole thing
- cardano-rpc converts to protobuf

With direct access:
- cardano-rpc can call `roforkerRangeReadTables` directly in a loop
- Can apply predicate filtering **per batch** (already happens on the consensus side, now happens in-process)
- Can implement **true streaming pagination** -- instead of reading the entire filtered result and then slicing, stop after collecting enough items for one page
- Converts ledger types -> protobuf directly per batch, no intermediate CBOR or cardano-api types

This is where the design gets the most interesting improvement: **pagination can be pushed down to the backing store level**. Currently `SearchUtxos` fetches the entire filtered UTxO set, sorts it, and slices. With direct range reads, you can stop early after filling a page.

**For tx submission (Mempool.addTx):** The tx needs to be a `GenTx blk` (consensus type), not the cardano-api `TxInMode` wrapper. The current path is: CBOR bytes -> `deserialiseFromCBOR` -> cardano-api `Tx era` -> `toConsensusGenTx` -> N2C LocalTxSubmission -> mempool. Direct path: CBOR bytes -> `deserialiseFromCBOR` -> `GenTx blk` -> `Mempool.addTx`. Skips the entire protocol layer.

#### The HFC complication

`ChainDB` and `NodeKernel` are parameterized by `CardanoBlock StandardCrypto`, which is a `HardForkBlock` across all eras. The `LedgerTables` use `CanonicalTxIn` and `HardForkTxOut` -- canonical representations that abstract across eras. To get era-specific `Core.TxOut era`, you need `ejectLedgerTables` from the HFC layer.

The existing `answerShelleyLookupQueries` and `answerShelleyTraversingQueries` already handle this via injection/ejection functions passed as parameters. cardano-rpc's direct access layer would need to replicate this pattern -- or better, reuse the existing `answerQuery` function from consensus with a directly-obtained forker rather than one obtained through the protocol.

#### Key question for the design: depth of integration

There's a spectrum of how deep cardano-rpc reaches into consensus:

1. **Shallow**: Reuse `answerQuery` from consensus -- get a forker, call `answerQuery cfg forker query`, get back the same types the N2C server would produce. Eliminates IPC + CBOR serialization but still uses the consensus query dispatch. Simplest, lowest risk.

2. **Medium**: Call `roforkerReadTables` / `roforkerRangeReadTables` directly, handle HFC ejection, convert ledger types -> protobuf. More code but enables streaming pagination optimization.

3. **Deep**: Bypass forkers for Tier 1 queries entirely (just STM reads), use forkers only for Tier 2/3. Mix-and-match per query type.

Option 3 is what was outlined in the earlier design. The risk is coupling to consensus internals. But Tier 1 queries via STM are so simple and stable that the coupling is minimal, and the existing `nkQueryLedger` helper in `Queries.hs` already does exactly this for metrics.

---

### 1.4 High-Level Design: Direct Ledger State Access

#### Current Architecture (N2C IPC)

```
gRPC request
  -> cardano-rpc handler
    -> executeLocalStateQueryExpr (opens new Unix socket connection)
      -> Ouroboros handshake + N2C version negotiation
        -> LocalStateQuery protocol: Acquire -> Query -> Query -> Release -> Done
          -> server side: getReadOnlyForkerAtPoint -> answerQuery -> CBOR encode result
        <- CBOR over socket
      -> cardano-api deserialises from CBOR
    -> cardano-rpc converts cardano-api types -> protobuf
  -> gRPC response
```

**Cost per request:** socket connect + Ouroboros handshake + CBOR serialize (server) + CBOR deserialize (client) + protobuf serialize. Every request pays this full cost.

#### Proposed Architecture (Direct Access)

```
gRPC request
  -> cardano-rpc handler
    -> STM read from ChainDB / ReadOnlyForker (in-process, zero-copy)
      -> ledger types already in memory
    -> cardano-rpc converts ledger types -> protobuf directly
  -> gRPC response
```

#### What cardano-rpc needs from NodeKernel

The queries map to three access patterns in consensus:

| RPC Query | Consensus Query | Footprint | Direct Access Path |
|---|---|---|---|
| `ReadParams` (pparams) | `GetCurrentPParams` | **QFNoTables** | `atomically (getCurrentLedger chainDB)` -> pure `getPParams` extraction |
| `ReadParams` (chain point, block no) | `GetChainPoint`, `GetChainBlockNo` | **QFNoTables** | `atomically (getCurrentLedger chainDB)` -> `headerState` extraction |
| `ReadUtxos` (by TxIn) | `GetUTxOByTxIn` | **QFLookupTables** | `getReadOnlyForkerAtPoint` -> `roforkerReadTables` (O(m log n) key lookup) |
| `ReadUtxos` (whole) | `GetUTxOWhole` | **QFTraverseTables** | `getReadOnlyForkerAtPoint` -> `roforkerRangeReadTables` (paginated full scan) |
| `SearchUtxos` (by address) | `GetUTxOByAddress` | **QFTraverseTables** | `getReadOnlyForkerAtPoint` -> `roforkerRangeReadTables` + filter predicate |
| `SearchUtxos` (whole) | `GetUTxOWhole` | **QFTraverseTables** | Same as above, no filter |
| `SubmitTx` | N/A (uses LocalTxSubmission) | N/A | `Mempool.addTx AddTxForLocalClient` |
| `GetEra` | N/A (hardcoded Conway) | -- | Could stay hardcoded or read from ledger state |

#### Key Design Decisions

**1. What to pass to cardano-rpc**

Currently `runRpcServer` receives `(RpcConfig, NetworkMagic)`. It needs access to:

```haskell
data RpcNodeHandle m blk = RpcNodeHandle
  { rpcChainDB         :: ChainDB m blk
  , rpcMempool         :: Mempool m blk
  , rpcTopLevelConfig  :: TopLevelConfig blk
  , rpcResourceRegistry :: ResourceRegistry m
  }
```

This can be obtained from `NodeKernel` inside the `rnNodeKernelHook`. The existing pattern is already there -- `NodeKernelData` wraps the kernel in an `IORef (StrictMaybe ...)` and is written via `setNodeKernel` in the hook. The RPC server can either:

- **(a)** Receive the `NodeKernelData` IORef and wait for it to become `SJust` (like `LedgerMetrics` does via `mapNodeKernelDataIO`)
- **(b)** Be started from inside `rnNodeKernelHook` after the kernel is available -- but this changes startup ordering and the RPC server wouldn't be listening during node sync

Option **(a)** is better -- matches the existing pattern and lets the RPC server start immediately, returning "not ready" errors until the kernel is initialized.

**2. QFNoTables queries (pparams, chain point, era) -- trivial**

These are pure functions on `ExtLedgerState blk EmptyMK`. The access is:

```haskell
extLedger <- atomically $ ChainDB.getCurrentLedger chainDB
let pparams = getPParams (ledgerState extLedger)
let chainPoint = headerStatePoint (headerState extLedger)
let blockNo = headerStateBlockNo (headerState extLedger)
```

Single atomic STM read, all consistent. No forker needed. This is exactly what `nkQueryLedger` in `Queries.hs` does.

**3. QFLookupTables queries (UTxO by TxIn) -- needs forker**

UTxO-HD moved the UTxO set to a backing store (on-disk/LSM). Reading specific UTxOs requires:

```haskell
bracket
  (ChainDB.getReadOnlyForkerAtPoint chainDB registry VolatileTip)
  (either (const $ pure ()) roforkerClose)
  $ \case
    Left err -> -- handle acquisition failure
    Right forker -> do
      LedgerTables (ValuesMK values) <-
        roforkerReadTables forker (LedgerTables (KeysMK txinSet))
      -- values :: Map TxIn TxOut, already in-memory ledger types
```

This is O(m log n) -- efficient point lookups. No serialization overhead.

**4. QFTraverseTables queries (UTxO by address, whole) -- needs range reads**

`GetUTxOByAddress` and `GetUTxOWhole` are **QFTraverseTables** -- they iterate the backing store with `roforkerRangeReadTables`, applying a predicate filter:

```haskell
loop predicate NoPreviousQuery emptyUtxo
  where
    loop pred prev !acc = do
      (LedgerTables (ValuesMK batch), mNext) <-
        roforkerRangeReadTables forker (castRangeQueryPrevious prev)
      let filtered = Map.filter pred batch
          acc' = acc <> filtered
      case mNext of
        Nothing -> pure acc'  -- end of store
        Just nextKey -> loop pred (PreviousQueryWasUpTo nextKey) acc'
```

This naturally supports **streaming pagination** -- cardano-rpc's `SearchUtxos` pagination can map directly to `roforkerRangeReadTables` iterations rather than fetching everything then slicing.

**5. Tx submission -- Mempool.addTx**

Replace N2C `submitTxToNodeLocal` with direct mempool access:

```haskell
result <- Mempool.addTx mempool AddTxForLocalClient genTx
case result of
  MempoolTxAdded _     -> pure txId
  MempoolTxRejected err -> throwError err
```

This skips the entire LocalTxSubmission protocol. Note: the tx must be converted to `GenTx blk` (consensus type) rather than `TxInMode` (cardano-api wrapper for N2C).

**6. The type abstraction problem**

`NodeKernel` and `ChainDB` are parameterized by `blk`, which in cardano-node is `CardanoBlock StandardCrypto` -- a hard-fork combinator type. cardano-rpc currently works with cardano-api types (`Tx era`, `UTxO era`, etc.).

Two approaches:
- **(a)** cardano-rpc works with consensus/ledger types directly, converts to protobuf from `SL.UTxO era`, `LC.PParams era`, etc. -- skips the cardano-api layer entirely
- **(b)** A thin adapter in cardano-node converts consensus types -> cardano-api types, then passes to cardano-rpc

Option **(a)** eliminates the most serialization but couples cardano-rpc to ledger internals. Option **(b)** keeps a cleaner boundary but retains some conversion cost (though no CBOR -- just in-memory type mapping).

Leaning toward **(a)** for the hot path (UTxO data is large) with an adapter module that handles the HFC `CardanoBlock` unwrapping.

#### What This Eliminates

| Overhead | N2C (current) | Direct (proposed) |
|---|---|---|
| Socket connect | Per request | None |
| Ouroboros handshake | Per request | None |
| N2C version negotiation | Per request | None |
| CBOR serialization (server) | Per query result | None |
| CBOR deserialization (client) | Per query result | None |
| cardano-api type wrapping | Per query result | None (or minimal) |
| Forker acquisition | Implicit (via protocol) | Explicit (same cost, no IPC wrapper) |
| STM ledger state read | Same | Same |

For QFNoTables queries (pparams, chain point), the improvement is dramatic -- from a full socket round-trip to a single `atomically` call. For UTxO queries, the forker + backing store IO is the real cost and stays the same, but the IPC + serialization overhead is removed.

#### Proposed Module Structure

```
cardano-rpc/
  src/Cardano/Rpc/Server/
    Internal/
      DirectAccess.hs          -- NEW: direct ChainDB/Mempool access layer
      DirectAccess/
        Query.hs               -- NEW: replaces N2C queries
        Submit.hs              -- NEW: replaces N2C tx submission
      UtxoRpc/
        Query.hs               -- MODIFIED: calls DirectAccess instead of executeLocalStateQueryExpr
        Submit.hs              -- MODIFIED: calls DirectAccess instead of submitTxToNodeLocal
        Type.hs                -- MODIFIED: converts from ledger types instead of cardano-api types
        Predicate.hs           -- unchanged (already works on cardano-api types)
      Env.hs                   -- MODIFIED: holds NodeKernelData instead of LocalNodeConnectInfo

cardano-node/
  src/Cardano/Node/
    Run.hs                     -- MODIFIED: pass NodeKernelData to runRpcServer
    Rpc/
      Adapter.hs               -- NEW: unwraps HFC CardanoBlock, provides era-indexed access
```

#### Startup Sequence

```
1. mkNodeKernelData -> IORef SNothing
2. runRpcServer starts (receives NodeKernelData ref)
   - gRPC server listening immediately
   - queries return UNAVAILABLE until kernel ready
3. Node.run -> rnNodeKernelHook fires
4. setNodeKernel writes SJust kernel into IORef
5. RPC queries now read kernel -> ChainDB -> ledger state directly
```

---

### 1.5 Subagent: Comprehensive Implementation Plan

(From subagent `agent-aed46478c4e93ea72`)

#### 1. The Core Problem: Abstraction Boundary

The fundamental challenge is that `cardano-rpc` lives in the `cardano-api` repo and currently has no dependency on `ouroboros-consensus-diffusion` or `ouroboros-consensus-cardano`. The `NodeKernel` type is parameterized by `blk`, which in `cardano-node` is `CardanoBlock StandardCrypto`. The `ReadOnlyForker`, `Mempool`, `ChainDB`, `answerQuery`, etc. all live in `ouroboros-consensus`.

There are two viable approaches:

**Approach A: Abstract interface in cardano-rpc, concrete implementation in cardano-node.**
Define an abstract "ledger accessor" record of functions in `cardano-rpc`. Implement it concretely in `cardano-node` using `NodeKernel`. This keeps `cardano-rpc` independent of consensus.

**Approach B: Add consensus dependencies to cardano-rpc.**
Make `cardano-rpc` directly depend on `ouroboros-consensus` and `ouroboros-consensus-cardano`, parameterize `RpcEnv` by `blk`, and use `NodeKernelData blk` directly.

**Recommended: Approach A** -- it is cleaner for the abstraction boundary, keeps `cardano-rpc` lightweight, and follows the same pattern used for tracers.

#### 2. LedgerAccess Design Evolution

Multiple iterations were considered:

**First attempt** -- per-query concrete functions:
```haskell
data LedgerAccess = LedgerAccess
  { laQueryProtocolParams :: IO (Ledger.PParams (ShelleyLedgerEra ConwayEra))
  , laQueryUtxoByTxIn :: Set TxIn -> IO (UTxO ConwayEra)
  , laQueryUtxoByAddress :: Set AddressAny -> IO (UTxO ConwayEra)
  , laQueryUtxoWhole :: IO (UTxO ConwayEra)
  , laQueryChainPoint :: IO (ChainPoint, WithOrigin BlockNo)
  , laSubmitTx :: TxInMode -> IO (SubmitResult TxValidationErrorInCardanoMode)
  }
```

**Problem:** Hardcodes Conway era.

**Second attempt** -- era-polymorphic:
```haskell
data LedgerAccess = LedgerAccess
  { laQueryProtocolParams :: forall era. Era era -> IO (Ledger.PParams (LedgerEra era))
  , laQueryUtxoByTxIn :: forall era. Era era -> Set TxIn -> IO (UTxO era)
  ...
```

**Problem:** The `era` would need to match the current node era, but the caller chooses `era` statically.

**Third attempt** -- GADT query accessor:
```haskell
data QueryAccessor a where
  QACurrentEra :: QueryAccessor AnyCardanoEra
  QAChainPoint :: QueryAccessor (ChainPoint, WithOrigin BlockNo)
  QAProtocolParams :: ShelleyBasedEra era -> QueryAccessor (Ledger.PParams (ShelleyLedgerEra era))
  ...
```

**Problem:** Introduces complexity.

**Fourth attempt** -- re-inventing the N2C interface. Rejected.

**Final design:**
```haskell
data LedgerAccess = LedgerAccess
  { laDetermineEra :: IO AnyCardanoEra
  , laQueryChainTip :: IO (ChainPoint, WithOrigin BlockNo)
  , laQueryProtocolParams :: ShelleyBasedEra era -> IO (Ledger.PParams (ShelleyLedgerEra era))
  , laQueryUtxo :: ShelleyBasedEra era -> QueryUTxOFilter -> IO (UTxO era)
  , laSubmitTx :: TxInMode -> IO (SubmitResult TxValidationErrorInCardanoMode)
  }
```

Return types are cardano-api types so existing `Type.hs` / `Predicate.hs` conversion code is unchanged.

#### 3. Phase-by-Phase Plan

**Phase 1: LedgerAccess type** -- New file in cardano-rpc

**Phase 2: RpcEnv changes** -- Add `IORef (Maybe LedgerAccess)` field, add `Has` instance, add helper:
```haskell
withLedgerAccessOrUnavailable
  :: MonadRpc e m
  => (LedgerAccess -> IO a) -> m a
```
This handles the startup race: queries during initialization return gRPC UNAVAILABLE.

**Phase 3: runRpcServer signature change** -- Accept `IORef (Maybe LedgerAccess)` parameter

**Phase 4: mkLedgerAccess in cardano-node** -- Bridge module using NodeKernel

The key insight for implementation: **Reuse `toConsensusQuery` + `fromConsensusQueryResult` from cardano-api** to construct consensus queries and interpret results. The chain is:
1. Construct `QueryInShelleyBasedEra era result`
2. Use `toConsensusQuery sbe query` to get `Some (Query (CardanoBlock c) result')`
3. Acquire forker at `VolatileTip`
4. Use `answerQuery cfg forker consensusQuery` to get the result
5. Use `fromConsensusQueryResult` to convert back to cardano-api types
6. Close forker

This reuses almost all existing code and just replaces the IPC transport.

**Phase 5: Wire up in Run.hs** -- IORef creation + `writeIORef` in `rnNodeKernelHook`

**Phase 6: Rewrite query methods** -- Each method changes from N2C pattern to `withLedgerAccess` pattern

**Phase 7: Type conversion** -- Minimal changes. Because LedgerAccess returns cardano-api types, `Type.hs`, `Predicate.hs` stay unchanged.

**Phase 8: Error handling** -- Startup window handling, forker errors, tx submission errors, new trace constructors

**Phase 9: Dependencies** -- No new consensus deps for cardano-rpc (Approach A). No new deps for cardano-node.

**Phase 10: Config** -- Remove `nodeSocketPath` (later, keep for backward compat initially)

**Phase 11: Testing** -- Unit tests, integration tests, regression, benchmarks

**Phase 12: Migration path** -- Phase 1 with fallback -> Phase 2 remove fallback -> Phase 3 streaming optimization

#### 4. Risks and Mitigations

| Risk | Mitigation |
|------|-----------|
| ResourceRegistry lifetime management | Use `withRegistry` per-query, matching N2C pattern |
| Forker not closed on exception | Use `bracket` pattern consistently |
| HFC query dispatch complexity | Reuse `toConsensusQuery` + `fromConsensusQueryResult` |
| Era mismatch between query construction and execution | `answerQuery` handles this via HFC dispatch |
| Startup race (kernel not ready) | IORef with Nothing + UNAVAILABLE status |
| Thread safety of IORef writes | Single writer (onKernel hook), multiple readers -- IORef fine |

#### Critical Files for Implementation

- `cardano-api/cardano-rpc/src/Cardano/Rpc/Server/Internal/Env.hs` -- Must add `IORef (Maybe LedgerAccess)` field
- `cardano-api/cardano-rpc/src/Cardano/Rpc/Server/Internal/UtxoRpc/Query.hs` -- Core query methods to rewrite
- `cardano-node/@worktree/add-grpc-interface/cardano-node/src/Cardano/Node/Run.hs` -- Wire NodeKernel to LedgerAccess
- `cardano-api/cardano-api/src/Cardano/Api/Query/Internal/Type/QueryInMode.hs` -- Contains `toConsensusQuery`/`fromConsensusQueryResult` to reuse
- `ouroboros-consensus/ouroboros-consensus/src/ouroboros-consensus/Ouroboros/Consensus/Ledger/Query.hs` -- `answerQuery` function

---

### 1.6 Subagent: LedgerTables Type Family System

(From subagent `agent-a6ac5c0eb8de8c465`)

#### Core Type Definition

**File:** `ouroboros-consensus/Ouroboros/Consensus/Ledger/Tables/Basics.hs`

```haskell
type LedgerTables :: LedgerStateKind -> MapKind -> Type
newtype LedgerTables l mk = LedgerTables
  { getLedgerTables :: mk (TxIn l) (TxOut l)
  }
```

Where `MapKind` is `Type -> Type -> Type` (key and value parameters).

#### MapKind Definitions

**File:** `ouroboros-consensus/Ouroboros/Consensus/Ledger/Tables/MapKind.hs`

- **EmptyMK**: `data EmptyMK k v = EmptyMK` -- no UTxO data present
- **KeysMK**: `newtype KeysMK k v = KeysMK (Set k)` -- only key set
- **ValuesMK**: `newtype ValuesMK k v = ValuesMK {getValuesMK :: Map k v}` -- full key-value map
- **DiffMK**: `newtype DiffMK k v = DiffMK {getDiffMK :: Diff k v}` -- changes (Insert v | Delete)
- **TrackingMK**: `data TrackingMK k v = TrackingMK !(Map k v) !(Diff k v)` -- values + accumulated diffs
- **SeqDiffMK**: finger tree of diffs

#### What EmptyMK Means Concretely

When you have `LedgerState era EmptyMK`:
- The `EmptyMK` represents that **the UTxO map data is literally not present** in this ledger state
- The Consensus layer stores actual UTxO data separately on disk (in the backing store)
- The UTxO can be "stowed" (extracted from the ledger) or "unstowed" (injected back into the ledger)

#### Shelley-Specific Instance

```haskell
type instance TxIn (LedgerState (ShelleyBlock proto era)) = BigEndianTxIn
type instance TxOut (LedgerState (ShelleyBlock proto era)) = Core.TxOut era
```

`BigEndianTxIn` is a wrapper that ensures proper byte ordering for serialization.

#### Cardano HFC Handling

```haskell
type instance TxIn (LedgerState (HardForkBlock xs)) = CanonicalTxIn xs
type instance TxOut (LedgerState (HardForkBlock xs)) = HardForkTxOut xs
```

Key functions: `injectLedgerTables` and `ejectLedgerTables` transform between era-specific and canonical representations via `bimapLedgerTables`.

#### How cardano-api Gets UTxO from Queries

The flow:
1. cardano-api calls `queryUtxo` with a filter
2. Converts to `toConsensusQuery` -> `GetUTxOWhole` / `GetUTxOByAddress` / `GetUTxOByTxIn`
3. Consensus queries in-memory LedgerState (with `ValuesMK`)
4. Result comes as `Shelley.UTxO (ShelleyLedgerEra era)`
5. Converts via `fromLedgerUTxO` to cardano-api `UTxO era`

---

### 1.7 Subagent: UTxO-HD Architecture Report

(From subagent `agent-a9a268af332ae9082`)

#### LedgerDB and Backing Store

The LedgerDB is responsible for:
- Maintaining in-memory ledger state at the tip
- Maintaining past k in-memory ledger states (supports rollback)
- Providing LedgerTables at any of the last k states
- Storing snapshots on disk
- Flushing LedgerTable differences to backing store

#### Backing Store Abstraction

```haskell
data BackingStore m keys key values diff = BackingStore
  { bsClose :: m ()
  , bsCopy :: SerializeTablesHint values -> FS.FsPath -> m ()
  , bsValueHandle :: m (BackingStoreValueHandle m keys key values)
  , bsWrite :: SlotNo -> WriteHint diff -> diff -> m ()
  , bsSnapshotBackend :: SnapshotBackend
  }
```

#### Two Backend Implementations

**V1 Backend -- InMemory:**
- Uses a `TVar` holding full map
- All data stays in memory
- Used for testing or small deployments

**V2 Backend -- LSM (Log-Structured Merge):**
- Uses LSM trees from the `lsm-tree` library
- Keys and values serialized to bytes
- Uses indexed packing (`IndexedMemPack`) for efficient serialization

#### The Forker

```haskell
data Forker m l = Forker
  { forkerReadTables :: LedgerTables l KeysMK -> m (LedgerTables l ValuesMK)
  , forkerRangeReadTables :: RangeQueryPrevious l -> m (LedgerTables l ValuesMK, Maybe (TxIn l))
  , forkerGetLedgerState :: STM m (l EmptyMK)    -- NO UTxO
  , forkerReadStatistics :: m Statistics
  , forkerPush :: l DiffMK -> m ()
  , forkerCommit :: STM m ()
  }
```

**Critical insight:** `forkerGetLedgerState` returns `l EmptyMK` -- **the UTxO data is NOT included**. You must explicitly call `forkerReadTables`.

#### Range Queries and BatchSize

`QueryBatchSize` defaults to 100,000 entries. `roforkerRangeReadTables` returns up to that many entries plus one additional key for pagination. Client iterates until `Nothing` (end of table).

#### ReadOnlyForker vs getCurrentLedger

| Aspect | getCurrentLedger | getReadOnlyForker |
|--------|------------------|------------------|
| Returns | `l EmptyMK` (STM) | `Forker m l` (IO) |
| UTxO access | NO | YES via `roforkerReadTables` |
| Speed | Fast (STM, in-memory) | Slower (allocates, I/O) |
| Use case | Ledger state queries, consensus | Query answering requiring UTxO |
| Resource management | None | Must close forker |

#### Three Footprint Categories for Shelley Queries

**QFNoTables:** `GetLedgerTip`, `GetEpochNo`, `GetCurrentPParams`, etc. -- uses `answerPureBlockQuery`, just in-memory ledger state

**QFLookupTables:** `GetUTxOByTxIn` -- uses `answerShelleyLookupQueries`, O(m * log n) for m inputs

**QFTraverseTables:** `GetUTxOByAddress`, `GetUTxOWhole` -- uses `answerShelleyTraversingQueries`, O(n) full scan with batched range reads

#### UTxO-HD Pipeline Diagram

```
Client Query (e.g., GetUTxOByAddress)
  |
  v
answerShelleyTraversingQueries
  |
  v
getReadOnlyForker(cfg, pt) -> Forker
  |
  v
roforkerRangeReadTables (QueryBatchSize = 100k)
  |
  v
BackingStore.bsvhRangeRead
  |- V1: reads from TVar (InMemory)
  |- V2: reads from LSM tree
  |
  v
Apply DbChangelog diffs (forward apply) [V1 only]
  |
  v
Filter by predicate (e.g., address match)
  |
  v
Accumulate into result (loop until end)
  |
  v
Return UTxO era to client
```

#### Why MapKind Matters

The MapKind parameter elegantly separates concerns:
- **`EmptyMK`** = "I have the ledger rules state but no UTxO"
- **`ValuesMK`** = "I have loaded UTxO from disk"
- **`KeysMK`** = "Here are the keys I want to read"
- **`DiffMK`** = "Here are the changes since last checkpoint"

This allows the type system to enforce that you can't accidentally use UTxO when you haven't loaded it.

---

### 1.8 Subagent: LocalStateQuery Server-Side Protocol

(From subagent `agent-a8865259424a84932`)

#### Protocol Setup

The server is initialized in `NodeToClient.hs`:
```haskell
hStateQueryServer =
    localStateQueryServer (ExtLedgerCfg cfg)
      . ChainDB.getReadOnlyForkerAtPoint getChainDB
```

#### Protocol States

```
StIdle --> Acquire --> StAcquiring --> Acquired --> StAcquired
StAcquired --> Query --> StQuerying --> Result --> StAcquired
StAcquired --> Release --> StIdle
StAcquired --> ReAcquire --> StAcquiring
```

#### Query Answering

```haskell
answerQuery config forker query = case query of
  BlockQuery blockQuery ->
    case sing :: Sing footprint of
      SQFNoTables ->
        answerPureBlockQuery config blockQuery
          <$> atomically (roforkerGetLedgerState forker)
      SQFLookupTables ->
        answerBlockQueryLookup config blockQuery forker
      SQFTraverseTables ->
        answerBlockQueryTraverse config blockQuery forker
  GetSystemStart ->
    pure $ getSystemStart config
  GetChainBlockNo ->
    headerStateBlockNo . headerState
      <$> atomically (roforkerGetLedgerState forker)
  GetChainPoint ->
    headerStatePoint . headerState
      <$> atomically (roforkerGetLedgerState forker)
```

#### Acquiring: ReadOnlyForker

When a client sends "Acquire" with a target point:
```haskell
getReadOnlyForkerAtPoint ::
    ResourceRegistry m ->
    Target (Point blk) ->
    m (Either GetForkerError (ReadOnlyForker' m blk))
```

Returns a read-only forker providing:
- Consistent view of ledger state at the requested point
- Access to ledger tables for queries that need them
- Automatic resource cleanup

#### What Direct Ledger Access Must Replicate

1. Accessing the ChainDB (from NodeKernel)
2. Reading current ledger state atomically: `atomically (ChainDB.getCurrentLedger chainDB)`
3. For specific point queries: `ChainDB.getReadOnlyForkerAtPoint chainDB reg target`
4. Reading ledger tables if needed
5. Converting query results to appropriate response types

---

### 1.9 Subagent: Ledger State In-Memory Architecture

(From subagent `agent-acf0ebc07f9f6a8a5`)

#### Node Kernel Structure

```haskell
data NodeKernel m addrNTN addrNTC blk = NodeKernel
  { getChainDB :: ChainDB m blk
  , getMempool :: Mempool m blk
  , getTopLevelConfig :: TopLevelConfig blk
  , ... (peer management, tracers, block forging state, etc.)
  }
```

Wrapped in `NodeKernelData`:
```haskell
newtype NodeKernelData blk =
  NodeKernelData
  { unNodeKernelData :: IORef (StrictMaybe (NodeKernel IO RemoteAddress LocalConnectionId blk))
  }
```

#### Access Points to Ledger State

**Via ChainDB API:**
```haskell
getCurrentLedger :: STM m (ExtLedgerState blk EmptyMK)
getImmutableLedger :: STM m (ExtLedgerState blk EmptyMK)
getPastLedger :: Point blk -> STM m (Maybe (ExtLedgerState blk EmptyMK))
getCurrentChain :: STM m (AnchoredFragment (Header blk))
getReadOnlyForkerAtPoint :: ResourceRegistry m -> Target (Point blk) -> m (Either GetForkerError (ReadOnlyForker' m blk))
```

**Via ReadOnlyForker:**
```haskell
data ReadOnlyForker m l = ReadOnlyForker
  { roforkerGetLedgerState :: STM m (l EmptyMK)
  , roforkerReadTables :: LedgerTables l KeysMK -> m (LedgerTables l ValuesMK)
  , roforkerRangeReadTables :: RangeQueryPrevious l -> m (LedgerTables l ValuesMK, Maybe (TxIn l))
  }
```

#### Concrete Ledger State Types (from Shelley)

```haskell
data NewEpochState era = NewEpochState
  { nesEL :: EpochNo, nesEs :: EpochState era, nesRu :: PulsingRewUpdate era
  , nesPd :: PoolDistr, nesBprev :: BlocksMade, nesBcur :: BlocksMade }

data UTxOState era = UTxOState
  { _utxosUtxo :: UTxO era, _utxosDeposited :: Coin, _utxosFees :: Coin
  , _utxosGovState :: GovState era, _utxosDonation :: Coin }
```

#### Existing In-Process Access Patterns

**LedgerMetrics Tracer** -- traces metrics every N slots using `mapNodeKernelDataIO` and `nkQueryLedger`

**Forging Loop** -- gets ledger state for a specific point using `getReadOnlyForkerAtPoint`

**Peer Selection** -- reads immutable ledger via `getImmutableLedger`

---

### 1.10 Subagent: cardano-rpc Implementation Details

(From subagent `agent-ac320c0d681d17783`)

#### Protocol Definitions

1. **`cardano/rpc/node.proto`** -- `GetEra()`, `GetProtocolParamsJson()`, Era enum
2. **`utxorpc/v1beta/query/query.proto`** -- `ReadParams()`, `ReadUtxos()`, `SearchUtxos()` with pagination
3. **`utxorpc/v1beta/submit/submit.proto`** -- `SubmitTx()` with CBOR-serialized transactions
4. **`utxorpc/v1beta/cardano/cardano.proto`** -- Complete Cardano data structures (Tx, TxOutput, PParams with 44 fields, certificates, scripts PlutusV1-V4, governance)

#### Key Architectural Patterns

- **Dependency Injection**: `RpcEnv` + `Has` typeclass + `MonadRpc` constraint + `RIO` monad
- **Error Handling**: `RpcException` GADT, `throwEither`/`throwExceptT` helpers, top-level handler
- **Tracing**: Span-based with random IDs, metrics emission, integration with cardano-node tracers
- **Query Optimization**: Address extraction from predicates for indexed lookup (`QueryUTxOByAddress`)
- **Pagination**: Token = `TxId#OutputIndex`, deterministic sort by TxIn, default 100 items
- **Number Representation**: BigInt follows RFC 8949 CBOR encoding (int64, big_u_int, big_n_int)

#### Integration Tests

1. `hprop_rpc_query_pparams` -- validates all 44 protocol params match ledger
2. `hprop_rpc_transaction` -- full round-trip: fetch UTxOs -> build tx -> submit -> confirm on-chain

---

### 1.11 Subagent: ADR-018 and Documentation

(From subagent `agent-a2478f641b7c6bc52`)

**Primary document:** `/work/cardano-node-wiki/docs/ADR-018-cardano-rpc-grpc-server.md`

**Status:** Proposed (2026-03-11)

**Core decision:** Add `cardano-rpc` package implementing UTxO RPC standard plus custom extensions.

**Transport:** Unix domain socket (IPC), default `rpc.sock`. Remote via Envoy reverse proxy.

**Feature flags:** `--grpc-enable`, `EnableRpc` config key. Opt-in and experimental.

**Current limitations:** CBOR overhead, connection-per-request, sequential mini-protocol

**Future improvements:** Direct ledger state access, connection pooling

**Related:** ADR-015 (JavaScript/WASM API) -- browser-side access via gRPC-Web through Envoy

---

## Conversation 2: Implementation Attempt and Refined Plan

Session: `c3650666-8da9-45e1-be40-b0875f425d08`

### 2.1 Implementation Observations

The second conversation attempted to implement the changes directly. Key findings during implementation:

1. **PParams type/import strategy** -- Verified `Ledger.PParams (ShelleyLedgerEra era)` is the correct type from `cardano-ledger-core`

2. **Config.hs decision** -- Keeping `nodeSocketPath` in `RpcConfig` because removing it breaks downstream testnet/POM code. The field is no longer used at runtime for N2C connections (LedgerAccess replaces that), but it's still needed for configuration parsing and deriving the RPC socket path.

3. **Dependency verification** -- Confirmed `cardano-ledger-core` and `grpc-spec` are already in cabal deps; no new dependencies needed.

4. **Import consistency** -- Confirmed `SubmitResult` and `TxValidationErrorInCardanoMode` need explicit imports from `Cardano.Api.Network.IPC`, not re-exported from `Cardano.Api`.

5. **`withLedgerAccess` vs `withLedgerAccessOrUnavailable`** -- The "OrUnavailable" variant was chosen: it reads the IORef and throws a gRPC `UNAVAILABLE` status if `Nothing`, cleanly handling the startup window.

All changes were reverted after exploration, and the refined plan was written from the knowledge gained.

### 2.2 Final Refined Plan

The plan was saved and is reproduced in full below. It represents the final, validated design after the implementation attempt.

#### Context

cardano-rpc queries the node via Node-to-Client (N2C) IPC. Every RPC request pays: socket connect + Ouroboros handshake + N2C version negotiation + CBOR serialize (server) + CBOR deserialize (client) + protobuf serialize. This is the #1 performance bottleneck identified in ADR-018.

**Goal:** Eliminate the N2C IPC layer by reading ledger state directly from the in-process `NodeKernel` handles (ChainDB, Mempool), converting ledger types -> protobuf without intermediate CBOR serialization.

**UTxO-HD constraint:** The UTxO set is NOT in the in-memory ledger state (`EmptyMK`). It lives in a backing store. Reading UTxO requires acquiring a `ReadOnlyForker`. Pure ledger metadata is available via STM `getCurrentLedger` without a forker.

#### Approach: Abstract Interface + Reuse `answerQuery`

Define a `LedgerAccess` record of callbacks in cardano-rpc (no consensus dependencies). Implement concretely in cardano-node using `NodeKernel`. Reuse consensus's `answerQuery` for HFC dispatch + forker management, and cardano-api's `toConsensusQuery` / `fromConsensusQueryResult` for type conversion.

#### Step 1: Define LedgerAccess

New file: `cardano-api/cardano-rpc/src/Cardano/Rpc/Server/Internal/LedgerAccess.hs`

```haskell
data LedgerAccess = LedgerAccess
  { laDetermineEra    :: IO AnyCardanoEra
  , laQueryChainTip   :: IO (ChainPoint, WithOrigin BlockNo)
  , laQueryPParams    :: ShelleyBasedEra era -> IO (Ledger.PParams (ShelleyLedgerEra era))
  , laQueryUtxo       :: ShelleyBasedEra era -> QueryUTxOFilter -> IO (UTxO era)
  , laSubmitTx        :: TxInMode -> IO (SubmitResult TxValidationErrorInCardanoMode)
  }

withLedgerAccess :: IORef (Maybe LedgerAccess) -> (LedgerAccess -> IO a) -> IO a
-- Reads IORef; throws gRPC UNAVAILABLE if Nothing
```

#### Step 2: Update RpcEnv and MonadRpc

- Add `rpcLedgerAccess :: !(IORef (Maybe LedgerAccess))` to `RpcEnv`
- Remove `rpcLocalNodeConnectInfo` (no longer needed)
- Replace `Has LocalNodeConnectInfo RpcEnv` with `Has (IORef (Maybe LedgerAccess)) RpcEnv`

#### Step 3: Update runRpcServer signature

```haskell
runRpcServer :: Tracer IO TraceRpc -> RpcConfig -> IORef (Maybe LedgerAccess) -> IO ()
```

`NetworkMagic` and `nodeSocketPath` no longer needed -- `LedgerAccess` already knows these.

#### Step 4: Implement mkLedgerAccess in cardano-node

New file: `cardano-node/src/Cardano/Node/Rpc/LedgerAccess.hs`

```haskell
mkLedgerAccess
  :: NodeKernel IO RemoteAddress LocalConnectionId (CardanoBlock StandardCrypto)
  -> LedgerAccess
```

Implementation per field:
- **laDetermineEra**: Read ExtLedgerState, extract era from HFC HardForkLedgerState
- **laQueryChainTip**: Forker at VolatileTip, answerQuery with GetChainPoint/GetChainBlockNo
- **laQueryPParams**: Forker at VolatileTip, answerQuery with HFC-wrapped GetCurrentPParams
- **laQueryUtxo**: Forker at VolatileTip, answerQuery with appropriate HFC-wrapped UTxO query
- **laSubmitTx**: `addLocalTxs mempool (MkSolo genTx)` directly

ResourceRegistry: `withRegistry` per query, matching N2C server pattern.

Key functions to reuse:
- `toConsensusQuery` / `fromConsensusQueryResult` from `Cardano.Api.Query.Internal.Type.QueryInMode`
- `toConsensusGenTx` / `fromConsensusApplyTxErr` from `Cardano.Api.Network.IPC.Internal`
- `answerQuery` from `Ouroboros.Consensus.Ledger.Query`
- `ChainDB.getReadOnlyForkerAtPoint` from ChainDB API
- `addLocalTxs` from Mempool API

#### Step 5: Wire up in Run.hs

```haskell
ledgerAccessRef <- newIORef Nothing

withAsync (runRpcServer (rpcTracer tracers) (ncRpcConfig nc) ledgerAccessRef) $ \_ ->
  Node.run nodeArgs {
    rnNodeKernelHook = \registry nodeKernel -> do
      writeIORef ledgerAccessRef (Just (mkLedgerAccess nodeKernel))
      rnNodeKernelHook nodeArgs registry nodeKernel
  }
```

#### Step 6: Rewrite query methods

Each method changes from N2C pattern to:
```haskell
laRef <- grab @(IORef (Maybe LedgerAccess))
withLedgerAccess laRef $ \la -> do
  AnyCardanoEra era <- laDetermineEra la
  ...
```

Predicate filtering, pagination, protobuf conversion remain unchanged.

#### Step 7: Update tracing

- Remove `TraceRpcSubmitN2cConnectionError`
- Add `TraceRpcLedgerAccessUnavailable`
- Add `TraceRpcForkerError`

#### Step 8: Update cabal and Config

- Add `LedgerAccess` module to cabal
- No new dependencies
- Remove `nodeSocketPath` from `RpcConfig` (or keep for backward compat initially)

#### Step 9: Update integration tests

Tests should work as-is since gRPC client API is unchanged. May need to adjust testnet startup.

#### Files Summary

**New files:**
| File | Purpose |
|------|---------|
| `cardano-rpc/src/.../LedgerAccess.hs` | LedgerAccess record type + withLedgerAccess helper |
| `cardano-node/src/.../Rpc/LedgerAccess.hs` | mkLedgerAccess from NodeKernel |

**Modified files:**
| File | Change |
|------|--------|
| `Server.hs` | New signature, IORef wiring |
| `Env.hs` | RpcEnv holds IORef |
| `Monad.hs` | Has instance + MonadRpc update |
| `Query.hs` | Rewrite 3 methods |
| `Submit.hs` | Rewrite submitTxMethod |
| `Node.hs` | Rewrite getEraMethod, getProtocolParamsJsonMethod |
| `Tracing.hs` | New/removed trace constructors |
| `Config.hs` | Remove nodeSocketPath |
| `Run.hs` | IORef creation + writeIORef in kernel hook |
| `cardano-node.cabal` | Add new module |
| `Rpc.hs (tracing)` | Update trace instances |

---

## Conversation 3: Documentation and Plan Writing

Session: `9df210e3-519d-4f05-996d-8910d66f77ed`

### 3.1 Architecture Pattern Discussion

The chosen pattern is **record-of-IO-callbacks** with an `IORef (Maybe LedgerAccess)` bridge.

- **cardano-rpc side**: A `LedgerAccess` record with 5 `IO` callback fields. Uses only cardano-api types -- no consensus dependency. Holds an `IORef (Maybe LedgerAccess)` and throws `UNAVAILABLE` until populated.

- **cardano-node side**: `mkLedgerAccess` fills in the callbacks using `NodeKernel` -- acquires a read-only forker from ChainDB, runs `answerQuery` for queries, and uses `addLocalTxs` on the mempool for tx submission. The `IORef` is written in `rnNodeKernelHook` once consensus finishes initialization.

This keeps the dependency graph clean (cardano-rpc never imports consensus).

### 3.2 Alternatives Considered

1. **Record-of-callbacks (chosen)** -- `LedgerAccess` record of `IO` functions in cardano-rpc, implemented in cardano-node. Clean dependency inversion, testable with mocks.

2. **Typeclass** -- e.g. `class MonadLedgerAccess m where ...` in cardano-rpc, with an instance in cardano-node. Similar decoupling but heavier machinery (orphan instances or newtype wrappers).

3. **Direct consensus dependency in cardano-rpc** -- Import `NodeKernel`, `answerQuery`, etc. directly. Simplest code but creates a heavy dependency from cardano-rpc to ouroboros-consensus, coupling the packages tightly.

4. **Keep N2C but optimize** -- Connection pooling / persistent connection instead of per-request connect. Reduces overhead without architectural change, but still pays CBOR serialization round-trips for in-process communication.

5. **Shared TVar/IORef of ledger state snapshots** -- Node periodically writes a snapshot of frequently-queried data (tip, pparams, utxo) to a shared ref. Simple reads, but stale data and doesn't cover arbitrary queries or tx submission.

### 3.3 Key Findings from Previous Transcripts

The second conversation (implementation attempt) yielded these practical findings:

1. **PParams type/import strategy** -- verified `Ledger.PParams (ShelleyLedgerEra era)` is the right type from `cardano-ledger-core`
2. **Config.hs decision** -- keeping `nodeSocketPath` because removing it breaks downstream testnet/POM code
3. **Dependency verification** -- confirmed `cardano-ledger-core` and `grpc-spec` already in cabal deps
4. **Import consistency** -- confirmed `SubmitResult` and `TxValidationErrorInCardanoMode` need explicit imports, not re-exported from `Cardano.Api`

There was no alternatives analysis or architectural trade-off discussion in the implementation transcript -- the plan was written directly from practical experience.

---

## Appendix: Key File Paths Referenced

### cardano-rpc (in cardano-api repo)
- `/work/cardano-api/cardano-rpc/src/Cardano/Rpc/Server.hs`
- `/work/cardano-api/cardano-rpc/src/Cardano/Rpc/Server/Internal/Env.hs`
- `/work/cardano-api/cardano-rpc/src/Cardano/Rpc/Server/Internal/Monad.hs`
- `/work/cardano-api/cardano-rpc/src/Cardano/Rpc/Server/Internal/Node.hs`
- `/work/cardano-api/cardano-rpc/src/Cardano/Rpc/Server/Internal/UtxoRpc/Query.hs`
- `/work/cardano-api/cardano-rpc/src/Cardano/Rpc/Server/Internal/UtxoRpc/Submit.hs`
- `/work/cardano-api/cardano-rpc/src/Cardano/Rpc/Server/Internal/UtxoRpc/Type.hs`
- `/work/cardano-api/cardano-rpc/src/Cardano/Rpc/Server/Internal/UtxoRpc/Predicate.hs`
- `/work/cardano-api/cardano-rpc/src/Cardano/Rpc/Server/Internal/Tracing.hs`
- `/work/cardano-api/cardano-rpc/src/Cardano/Rpc/Server/Config.hs`
- `/work/cardano-api/cardano-rpc/cardano-rpc.cabal`
- `/work/cardano-api/cardano-rpc/proto/` (proto definitions)

### cardano-node (worktree)
- `/work/cardano-node/@worktree/add-grpc-interface/cardano-node/src/Cardano/Node/Run.hs`
- `/work/cardano-node/@worktree/add-grpc-interface/cardano-node/src/Cardano/Node/Queries.hs`
- `/work/cardano-node/@worktree/add-grpc-interface/cardano-node/src/Cardano/Node/Tracing/Tracers/Rpc.hs`
- `/work/cardano-node/@worktree/add-grpc-interface/cardano-node/src/Cardano/Node/Tracing/Tracers/LedgerMetrics.hs`
- `/work/cardano-node/@worktree/add-grpc-interface/cardano-node/cardano-node.cabal`
- `/work/cardano-node/@worktree/add-grpc-interface/cardano-testnet/test/Cardano/Testnet/Test/Rpc/`

### ouroboros-consensus
- `/work/ouroboros-consensus/ouroboros-consensus/src/ouroboros-consensus/Ouroboros/Consensus/Ledger/Query.hs`
- `/work/ouroboros-consensus/ouroboros-consensus/src/ouroboros-consensus/Ouroboros/Consensus/Ledger/Tables/Basics.hs`
- `/work/ouroboros-consensus/ouroboros-consensus/src/ouroboros-consensus/Ouroboros/Consensus/Ledger/Tables/MapKind.hs`
- `/work/ouroboros-consensus/ouroboros-consensus/src/ouroboros-consensus/Ouroboros/Consensus/Storage/ChainDB/API.hs`
- `/work/ouroboros-consensus/ouroboros-consensus/src/ouroboros-consensus/Ouroboros/Consensus/Storage/LedgerDB/Forker.hs`
- `/work/ouroboros-consensus/ouroboros-consensus/src/ouroboros-consensus/Ouroboros/Consensus/Storage/LedgerDB/API.hs`
- `/work/ouroboros-consensus/ouroboros-consensus/src/ouroboros-consensus/Ouroboros/Consensus/Storage/LedgerDB/V1/BackingStore/API.hs`
- `/work/ouroboros-consensus/ouroboros-consensus-diffusion/src/ouroboros-consensus-diffusion/Ouroboros/Consensus/NodeKernel.hs`
- `/work/ouroboros-consensus/ouroboros-consensus-diffusion/src/ouroboros-consensus-diffusion/Ouroboros/Consensus/Network/NodeToClient.hs`
- `/work/ouroboros-consensus/ouroboros-consensus/src/ouroboros-consensus/Ouroboros/Consensus/MiniProtocol/LocalStateQuery/Server.hs`
- `/work/ouroboros-consensus/ouroboros-consensus-cardano/src/shelley/Ouroboros/Consensus/Shelley/Ledger/Query.hs`
- `/work/ouroboros-consensus/ouroboros-consensus-cardano/src/shelley/Ouroboros/Consensus/Shelley/Ledger/Ledger.hs`
- `/work/ouroboros-consensus/ouroboros-consensus/src/ouroboros-consensus/Ouroboros/Consensus/HardFork/Combinator/Ledger.hs`

### cardano-api
- `/work/cardano-api/cardano-api/src/Cardano/Api/Query/Internal/Type/QueryInMode.hs`
- `/work/cardano-api/cardano-api/src/Cardano/Api/Network/IPC/Internal/Monad.hs`

### cardano-ledger
- `/work/cardano-ledger/eras/shelley/impl/src/Cardano/Ledger/Shelley/LedgerState.hs`

### Documentation
- `/work/cardano-node-wiki/docs/ADR-018-cardano-rpc-grpc-server.md`
- `/work/cardano-node-wiki/docs/ADR-019-direct-ledger-access-for-cardano-rpc.md`
- `/work/ouroboros-consensus/docs/website/contents/explanations/queries.md`
