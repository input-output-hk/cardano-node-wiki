# ADR-019: Node Kernel Access for cardano-rpc

# Status

Proposed

# Context

cardano-rpc is a gRPC server that runs inside the cardano-node process.
Despite sharing a process, it communicates with the node over a Unix socket using the Node-to-Client (N2C) mini-protocol.
Every RPC request opens a fresh socket connection, negotiates the Ouroboros protocol, serialises queries to CBOR, and deserialises the results back - only to re-serialise them as protobuf for the gRPC response.

This is the architecture inherited from when cardano-rpc was a separate process.
Now that it runs in-process, the IPC overhead is unnecessary.

## Current architecture

```mermaid
graph LR
    subgraph node["cardano-node process"]
        rpc["cardano-rpc"]
        n2c{{"N2C socket"}}
        consensus[("Consensus")]
        rpc -- "N2C IPC" --> n2c
        n2c --> consensus
    end
```

cardano-rpc acts as an IPC client to its own host process.
Each request pays for connection setup, protocol negotiation, and CBOR round-trips.

## Proposed architecture

```mermaid
graph LR
    subgraph node["cardano-node process"]
        rpc["cardano-rpc"]
        na["NodeKernelAccess"]
        subgraph kernel["NodeKernel"]
            chaindb[("ChainDB")]
            mempool[("Mempool")]
            config[("Config")]
        end
        rpc -- "in-process calls" --> na
        na --> chaindb
        na --> mempool
        na --> config
    end
```

cardano-rpc reads node state directly through a record of callbacks called `NodeKernelAccess`.
The node populates this record once the consensus layer has initialised.
There is no socket, no protocol negotiation, and no serialisation between cardano-rpc and the node internals.

# Decision

Provide cardano-rpc with a `NodeKernelAccess` record that exposes three capabilities:

1. **Ledger snapshots** - acquire a consistent, read-only view of the current ledger state and run any number of queries against it.
   All queries within one snapshot see the same chain tip, preserving the consistency that the N2C protocol provides via its acquire/query/release cycle.
2. **Transaction submission** - submit a transaction to the mempool for validation and inclusion.
3. **Block retrieval** - fetch raw block bytes from on-chain storage by slot and hash.

These three capabilities correspond to the three subsystems inside `NodeKernel`: ChainDB (ledger state and block storage), Mempool (pending transactions), and TopLevelConfig (genesis and era configuration).

## Dependency inversion

cardano-rpc defines the `NodeKernelAccess` interface using only cardano-api types.
It has no dependency on consensus internals.
cardano-node implements the interface using `NodeKernel`, which is the natural provider of all three capabilities.

This means cardano-rpc can be tested with a mock `NodeKernelAccess` that requires no running node.

## Startup sequencing

`NodeKernel` only becomes available after consensus initialisation completes.
The gRPC server starts earlier, so the `NodeKernelAccess` value is held behind a mutable reference that starts empty.
Requests arriving before the kernel is ready receive a gRPC `UNAVAILABLE` status.
The node populates the reference in its kernel-ready callback, after which all requests are served.

## Snapshot consistency

The current N2C code runs all queries within a single protocol session, which acquires one ledger snapshot.
Protocol parameters, UTxO results, chain tip, and block number all come from the same state.

A naive replacement with one callback per query type would break this: each call could see a different chain tip.
The snapshot-based design avoids this by letting the caller open a snapshot once and run arbitrarily many queries against it.

## UTxO RPC spec coverage

`NodeKernel` covers 16 of the 18 RPCs defined in the UTxO RPC v1beta specification.

| Subsystem | Capabilities | RPCs |
|-----------|-------------|------|
| ChainDB | Block retrieval, ledger queries, chain following | FetchBlock, DumpHistory, FollowTip, ReadTip, ReadParams, ReadUtxos, SearchUtxos, ReadGenesis, ReadEraSummary, ReadState, EvalTx |
| Mempool | Transaction submission, snapshot inspection | SubmitTx, ReadMempool, WatchMempool |
| TopLevelConfig | Genesis config, era history | ReadGenesis (also via ChainDB) |

The two RPCs that `NodeKernel` cannot serve are **ReadTx** (transaction lookup by hash) and **ReadData** (datum lookup by hash).
Both require indexes that the node does not maintain.

# Alternatives Considered

We considered keeping the N2C IPC path and optimising it - for example, by reusing connections or caching protocol negotiation.
This would reduce per-request overhead but not eliminate the fundamental cost of serialising to CBOR and back, or the inability to access ChainDB capabilities (block-by-point lookup, chain following) that the N2C protocol does not expose.

We also considered exposing `NodeKernel` directly to cardano-rpc without an abstraction layer.
This would be simpler initially but would couple cardano-rpc to consensus internals, making it impossible to test without a running node and fragile across consensus upgrades.

The `NodeKernelAccess` record-of-callbacks approach gives us the performance of direct access with the testability of an abstract interface.

# Consequences

- Eliminates the N2C overhead for all current and future RPC methods.
- Removes the per-request socket connection pattern.
- Existing RPC methods (ReadParams, ReadUtxos, SearchUtxos, SubmitTx, EvalTx) are migrated incrementally - each method is rewritten independently.
- `nodeSocketPath` remains in the configuration because it is used to derive the default gRPC socket path.
- cardano-rpc gains a compile-time dependency on cardano-api query types but not on consensus internals.
- Testing with a mock `NodeKernelAccess` becomes possible, removing the need for a running node in unit tests.

# References

- [ADR-018: cardano-rpc gRPC server](./ADR-018-cardano-rpc-grpc-server.md) - established cardano-rpc's architecture, acknowledged the N2C serialisation overhead and listed direct ledger state access as a planned improvement.
  This ADR delivers that improvement.
- [UTxO RPC specification](https://utxorpc.org/) - the gRPC service specification that cardano-rpc implements.

# Authors

- Mateusz Galazyn

