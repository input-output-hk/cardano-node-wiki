# cardano-rpc Roadmap

## Phase 1 — UTxORPC service parity

Close the feature gap with [cardano-node-api](https://github.com/blinklabs-io/cardano-node-api) (Blink Labs).
See [feature comparison](https://github.com/IntersectMBO/cardano-api/pull/1139/files) for full gap analysis.

**Query** ([proto](https://github.com/IntersectMBO/cardano-api/tree/master/cardano-rpc/proto/utxorpc/v1beta/query/query.proto), [handler](https://github.com/IntersectMBO/cardano-api/tree/master/cardano-rpc/src/Cardano/Rpc/Server/Internal/UtxoRpc/Query.hs)):
- **SearchUtxos** — predicate-based UTxO search. Needs tracing wiring in [cardano-node Rpc.hs](https://github.com/IntersectMBO/cardano-node/tree/master/cardano-node/src/Cardano/Node/Tracing/Tracers/Rpc.hs).
- **ReadTip** — simple chain tip query, low effort
- **ReadEraSummary** — era boundaries, slot/epoch arithmetic. Very useful for wallets.
- **ReadGenesis** — genesis config + CAIP-2 network identifier
- **ReadData** — datum lookup by hash (Plutus dApp use case)
- **ReadTx** — transaction lookup by hash

**Submit** ([proto](https://github.com/IntersectMBO/cardano-api/tree/master/cardano-rpc/proto/utxorpc/v1beta/submit/submit.proto), [handler](https://github.com/IntersectMBO/cardano-api/tree/master/cardano-rpc/src/Cardano/Rpc/Server/Internal/UtxoRpc/Submit.hs)):
- **EvalTx** — script evaluation without submission. Stateless, uses existing `evaluateTransactionExecutionUnits` from cardano-api. Relatively self-contained.
- **WaitForTx** — tx lifecycle streaming (`ACKNOWLEDGED → MEMPOOL → CONFIRMED`). Biggest piece — needs server-streaming via grapesy + ChainSync/TxMonitor or mempool+ChainDB access.

**Suggested order:** SearchUtxos → ReadTip/ReadEraSummary/ReadGenesis (quick wins) → EvalTx → ReadData/ReadTx → WaitForTx (most complex)

---

## Phase 2 — Correctness & Performance

- **Conformance tests** against existing endpoints first ([utxorpc/spec PR #188](https://github.com/utxorpc/spec/pull/188)). Provides regression safety net before the DLA refactor.
- **Property-based round-trip tests** for the 600+ lines of type conversions in [`Type.hs`](https://github.com/IntersectMBO/cardano-api/tree/master/cardano-rpc/src/Cardano/Rpc/Server/Internal/UtxoRpc/Type.hs) (`cardano-api type → proto → cardano-api type ≡ identity`). Hedgehog generators already exist in cardano-api-gen.
- **Performance baseline benchmarks** — tx generator-based, covering query and submit hot paths.
- **Direct ledger access** ([ADR-019, draft PR #94](https://github.com/input-output-hk/cardano-node-wiki/pull/94)) — 13-step plan in [implementation plan](https://github.com/input-output-hk/cardano-node-wiki/blob/mgalazyn/adr-rpc-direct-access/docs/direct-ledger-access-implementation-plan.md). `LedgerAccess` record-of-callbacks, `IORef (Maybe LedgerAccess)` populated in [`rnNodeKernelHook`](https://github.com/IntersectMBO/cardano-node/tree/master/cardano-node/src/Cardano/Node/Run.hs). Eliminates N2C serialization overhead, connection-per-request cost, and mini-protocol bottleneck. Also eliminates the need for connection pooling entirely.
- **Post-DLA benchmark comparison** to quantify gains.

---

## Phase 3 — Accessibility

- **HTTP(s) endpoint** — two options:
  - **Envoy gRPC-JSON transcoder** — zero cardano-rpc code, translates REST↔gRPC from proto descriptors. Configs already in [cardano-wasm/examples/grpc/](https://github.com/IntersectMBO/cardano-api/tree/master/cardano-wasm/examples/grpc/).
  - **Native Warp server** — minimal HTTP server alongside grapesy, serving JSON-encoded proto responses. More work but no external dependency.

---

## Phase 4 — Streaming (major architectural piece)

All require ChainSync mini-protocol integration — a significant change to the current architecture which only uses LocalStateQuery and LocalTxSubmission.

**SyncService:**
- FollowTip (server-stream of apply/undo/reset)
- FetchBlock (block lookup by point)
- DumpHistory (paginated block history)

**WatchService:**
- WatchTx (real-time tx filtering with predicates)

**SubmitService:**
- ReadMempool (snapshot)
- WatchMempool (server-stream of mempool changes)

---

## Phase 5 — Cardano-specific extensions

Where cardano-rpc has a unique advantage — embedded in-process with the same `cardano-api`/`cardano-ledger` types, v1beta [proto definitions](https://github.com/IntersectMBO/cardano-api/tree/master/cardano-rpc/proto/) with Conway governance types already defined.

- **Governance queries** — active proposals, DRep delegation, committee state, treasury withdrawals (proto types already exist)
- **Stake queries** — pool parameters, delegation info, reward account balances
- **Epoch/slot arithmetic** — POSIX time ↔ slot conversion (clients currently do this themselves from era summaries)
- **OpenTelemetry export** — full OTel traces + metrics beyond current span [tracing](https://github.com/IntersectMBO/cardano-api/tree/master/cardano-rpc/src/Cardano/Rpc/Server/Internal/Tracing.hs) and Prometheus counters

---

## Phase 6 — Ecosystem & DX

- **Client SDK publishing** — auto-generated packages from [proto defs](https://github.com/IntersectMBO/cardano-api/tree/master/cardano-rpc/proto/) in Python, TypeScript, Go, Rust with examples
- **gRPC-Web native** in grapesy — browser integration without requiring Envoy
- **gRPC health check protocol** (`grpc.health.v1`) — reflecting node sync status and kernel readiness

---

## Key references

| Document | Link |
|----------|------|
| ADR-018 (gRPC server) | [ADR-018-cardano-rpc-grpc-server.md](./ADR-018-cardano-rpc-grpc-server.md) |
| ADR-019 (direct ledger access, draft) | [PR #94](https://github.com/input-output-hk/cardano-node-wiki/pull/94), [ADR-019](https://github.com/input-output-hk/cardano-node-wiki/blob/mgalazyn/adr-rpc-direct-access/docs/ADR-019-direct-ledger-access-for-cardano-rpc.md) |
| DLA implementation plan | [direct-ledger-access-implementation-plan.md](https://github.com/input-output-hk/cardano-node-wiki/blob/mgalazyn/adr-rpc-direct-access/docs/direct-ledger-access-implementation-plan.md) |
| Feature comparison vs Blink Labs | [cardano-api PR #1139](https://github.com/IntersectMBO/cardano-api/pull/1139) |
| Conformance test spec | [utxorpc/spec PR #188](https://github.com/utxorpc/spec/pull/188) |
| UTxO RPC specification | [utxorpc.org](https://utxorpc.org/) |
| Server entry point | [`Server.hs`](https://github.com/IntersectMBO/cardano-api/tree/master/cardano-rpc/src/Cardano/Rpc/Server.hs) |
| Node integration | [`Run.hs`](https://github.com/IntersectMBO/cardano-node/tree/master/cardano-node/src/Cardano/Node/Run.hs) |
