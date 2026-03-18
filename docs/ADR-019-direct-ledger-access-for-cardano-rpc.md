# ADR-019: Direct Ledger State Access for cardano-rpc

## Status

Proposed

## Context

cardano-rpc currently queries the node via N2C IPC (Ouroboros mini-protocols over a Unix
socket). Every RPC request pays:

1. Socket connect
2. Handshake + version negotiation
3. CBOR round-trip (encode query, decode result)
4. Protobuf serialization for the gRPC response

This overhead is unnecessary when cardano-rpc runs **in-process** alongside the node. The
node already holds the ledger state in memory via `NodeKernel`; we can read it directly.

### Current architecture (N2C)

```
gRPC client
   |
   v
cardano-rpc  --(Unix socket / N2C mini-protocol)--> cardano-node
   |                                                      |
   v                                                      v
protobuf response                                   ledger state
```

### Proposed architecture (direct access)

```
gRPC client
   |
   v
cardano-rpc  --(IORef (Maybe LedgerAccess))--> NodeKernel (in-process)
   |                                                |
   v                                                v
protobuf response                             ledger state
```

## Decision

Define a `LedgerAccess` record of IO callbacks in cardano-rpc (no consensus dependencies).
Implement it in cardano-node using `NodeKernel` + consensus `answerQuery` + cardano-api's
`toConsensusQuery`/`fromConsensusQueryResult` for type conversion.

### Why a record-of-callbacks?

- **Dependency inversion:** cardano-rpc depends on cardano-api but NOT on consensus.
  The `LedgerAccess` record lives in cardano-rpc with cardano-api types only.
  cardano-node provides the implementation via `NodeKernel`.
- **Testability:** Tests can supply a mock `LedgerAccess` without a running node.
- **Startup sequencing:** `NodeKernel` is only available after consensus initialization.
  Using `IORef (Maybe LedgerAccess)` lets the gRPC server start immediately and return
  `UNAVAILABLE` until the kernel is ready.

## Consequences

- **Eliminates N2C overhead** for all RPC queries (UTxO, protocol params, chain tip, era)
  and transaction submission.
- **Removes the per-request socket connect** pattern (the "TODO replace with better
  connection management" in Env.hs).
- **`nodeSocketPath` stays in `RpcConfig`** because it's still needed to derive the default
  `rpcSocketPath` via `nodeSocketPathToRpcSocketPath`, and is imported by testnet code.
- **Trace constructors change:** `TraceRpcSubmitN2cConnectionError` is replaced by
  `TraceRpcLedgerAccessUnavailable` and `TraceRpcForkerError`.

## Files affected

See [Direct Ledger Access Implementation Plan](./direct-ledger-access-implementation-plan.md)
for the full file-by-file breakdown.
