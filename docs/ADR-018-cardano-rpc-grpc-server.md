# ADR-018: gRPC Server for Cardano Node (cardano-rpc)

# Status

📜 Proposed 2026-03-11

# Context

Cardano node exposes its API via the Node-to-Client mini-protocol over a Unix socket.
This requires clients to implement the Ouroboros protocol, which is a significant barrier for third-party tools, wallets, and indexers that just need to query chain state or submit transactions.

The [UTxO RPC](https://utxorpc.org/) specification defines a blockchain-agnostic gRPC API standard for UTxO-based blockchains.
Implementing this standard would let Cardano benefit from cross-chain tooling and give clients a well-documented, strongly-typed interface with automatic code generation in many languages.

# Decision

We will add a `cardano-rpc` package to `cardano-api` that provides a gRPC server exposing Cardano node functionality.
The server implements the **UTxO RPC** standard plus custom Cardano-specific extensions.

## Transport

The server communicates over a **Unix domain socket** (IPC), keeping the same security model as the existing node socket - local access only, no network exposure by default.
The default socket path is `rpc.sock` in the same directory as the node socket.
It can be overridden via `--grpc-socket-path` or the `RpcSocketPath` config key.

The Unix socket is a deliberate starting point for an experimental feature: it exposes no network ports, so the node's attack surface does not change.
The RPC server handles business logic (correct data, strongly typed responses).
Network concerns (TLS, authentication, rate limiting, DoS protection) are delegated to purpose-built infrastructure.

Operators who need remote access can use [Envoy](https://www.envoyproxy.io/) as a reverse proxy in front of the Unix socket.
Envoy terminates TLS, enforces access control (e.g., rate limiting, IP allowlists, mTLS client authentication), and translates between HTTP (including gRPC-Web for browser clients) and the upstream gRPC-over-Unix-socket connection.
This separation follows standard practice in service mesh architectures.
Working Envoy configurations are available in the [examples directory](https://github.com/IntersectMBO/cardano-api/tree/master/cardano-wasm/examples/grpc/).

Adding in-process HTTP exposure is not precluded by this design and could be added later if it helps adoption.

## Integration with cardano-node

The feature is **opt-in and experimental** at both build time and run time.
A cabal flag (`+rpc`, off by default) controls whether the gRPC dependencies are included in the build at all.
At run time, the RPC server is enabled via the `--grpc-enable` CLI flag or `EnableRpc` config key.

The RPC server runs concurrently in the node process.
It connects back to the node's own socket via the standard Node-to-Client IPC protocol to execute local state queries and submit transactions.

Although the RPC server runs in the same process as the node, it currently communicates through the Node-to-Client IPC socket - the same external interface any separate client would use.
This has several limitations:

- **Serialisation overhead** - every query round-trips through CBOR encoding and decoding even though both sides share the same process.
- **Connection-per-request cost** - each gRPC call opens a new IPC connection, paying the Ouroboros handshake cost every time.
- **Mini-protocol bottleneck** - the local state query mini-protocol is [sequential within a connection](https://ouroboros-network.cardano.intersectmbo.org/pdfs/network-spec/network-spec.pdf), limiting concurrency.
  The gRPC API can only expose what the Node-to-Client protocol supports.

Future iterations should move toward **direct ledger state access** - reading the node's in-memory state (e.g., via shared `TVar`/`STM` references) instead of going through IPC.
This would eliminate serialisation overhead and connection management entirely.
As a nearer-term improvement, **connection pooling** over persistent IPC connections would amortise handshake costs without requiring changes to the node's internals.

## Observability

All RPC operations are traced through the node's standard tracing infrastructure.
Trace events cover query and submit operations with timing and error reporting.

## Dependencies

The package introduces new dependencies:
- **grapesy** - gRPC server and client framework
- **grpc-spec** - gRPC protocol specification
- **proto-lens** - Protocol Buffer lens-based encoding/decoding

# Alternatives Considered

## JSON-RPC over HTTP

JSON-RPC over HTTP was considered as a simpler alternative.
However, it lacks streaming support, strong typing, and automatic code generation across languages that gRPC provides out of the box.

## REST API

A REST API was considered, but it is less suitable for the request/response patterns needed by blockchain clients.
There is also no established blockchain-agnostic REST specification comparable to UTxO RPC.

## Standalone sidecar in another language

A standalone sidecar (e.g., Go, as in [cardano-node-api](https://github.com/blinklabs-io/cardano-node-api)) provides better process isolation, but has several drawbacks.
It must re-implement Cardano type encoders and decoders in its own language, maintained by separate teams, which can lag behind hard forks, miss new fields, or encode CBOR differently from the canonical Haskell implementation.
It adds deployment complexity: a separate process to run, monitor, and keep version-compatible with the node, plus additional resource consumption and IPC overhead.
It brings its own supply chain (e.g., `gouroboros`, `go-codegen`, `connectrpc`, the Go toolchain) which is less auditable by the Haskell team maintaining the node.

By writing the RPC layer in Haskell and embedding it in the node, `cardano-rpc` uses the same `cardano-api`/`cardano-ledger` types and serialisation code as the node itself.
When a type changes, the RPC layer either compiles with the updated types or fails to build - it cannot silently drift out of sync.
The only custom conversion layer is from already-decoded Haskell values to protobuf, which eliminates an entire class of encoding bugs.
The RPC server shares the node's lifecycle and needs no configuration beyond a CLI flag.

Serving wrong data to clients due to encoder bugs is arguably a worse outcome than the security risks of running in-process behind a local Unix socket.

# Consequences

- Third-party tools, wallets, and indexers get a standard gRPC interface to the node, with auto-generated client libraries in many languages.
- UTxO RPC compatibility enables cross-chain tooling that targets multiple UTxO blockchains.
- The Unix socket transport preserves the existing security model - no network exposure by default.
- The node gains gRPC/protobuf build dependencies (`grapesy`, `grpc-spec`, `proto-lens`).
  A cabal flag (`+rpc`, off by default) gates these dependencies so that builds without the flag exclude `grapesy`, `grpc-spec`, `proto-lens`, and their transitive dependencies entirely.
  Operators who want the RPC interface must opt in at build time by enabling the flag.
- The feature is opt-in and experimental, so it does not affect existing node deployments.

# References

- [UTxO RPC specification](https://utxorpc.org/)
- [Proto definitions](https://github.com/IntersectMBO/cardano-api/tree/master/cardano-rpc/proto/)
- [Server implementation](https://github.com/IntersectMBO/cardano-api/blob/master/cardano-rpc/src/Cardano/Rpc/Server.hs)
- [Node integration](https://github.com/IntersectMBO/cardano-node/blob/master/cardano-node/src/Cardano/Node/Run.hs)
- [Envoy gRPC-Web example](https://github.com/IntersectMBO/cardano-api/tree/master/cardano-wasm/examples/grpc/)
- [Istio architecture - sidecar proxy pattern](https://istio.io/latest/docs/ops/deployment/architecture/)
- [Istio security - transparent mTLS via sidecar](https://istio.io/latest/docs/concepts/security/)
- [cardano-node-api (Blink Labs) - Go sidecar](https://github.com/blinklabs-io/cardano-node-api)

# Related ADRs

- [ADR-015](ADR-015-Cardano-API-WASM-library-for-browser) - the WASM library provides browser-side access to `cardano-api`.
  The gRPC server with Envoy/gRPC-Web enables browser clients to query the node.

# Appendix: Build flag dependency analysis

The `+rpc` cabal flag (off by default) gates `cardano-rpc` and its exclusive dependencies.
`cardano-rpc`'s total transitive closure is 479 packages, but 460 of those are already pulled in by `cardano-api`.
The flag eliminates **19 packages** that exist solely to support the gRPC stack.

## Protocol Buffers (5 packages)

| Package | Version | Maintainer |
|---|---|---|
| `proto-lens` | 0.7.1.6 | Google (proto-lens@googlegroups.com) |
| `proto-lens-runtime` | 0.7.0.7 | Google (proto-lens@googlegroups.com) |
| `proto-lens-protobuf-types` | 0.7.2.2 | Google (proto-lens@googlegroups.com) |
| `lens-family` | 2.1.3 | Russell O'Connor |
| `lens-family-core` | 2.1.3 | Russell O'Connor |

## gRPC + HTTP/2 transport (10 packages)

| Package | Version | Maintainer |
|---|---|---|
| `grapesy` | 1.1.1 | Edsko de Vries (Well-Typed) |
| `grpc-spec` | 1.0.0 | Edsko de Vries (Well-Typed) |
| `http2` | 5.3.9 | Kazu Yamamoto (IIJ) |
| `http2-tls` | 0.4.9 | Kazu Yamamoto (IIJ) |
| `http-semantics` | 0.3.0 | Kazu Yamamoto (IIJ) |
| `network-control` | 0.1.7 | Kazu Yamamoto (IIJ) |
| `network-run` | 0.4.4 | Kazu Yamamoto (IIJ) |
| `recv` | 0.1.1 | Kazu Yamamoto (IIJ) |
| `record-hasfield` | 1.0.1 | Neil Mitchell |
| `snappy-c` | 0.1.3 | Finley McIlwaine (Well-Typed) |

## Miscellaneous (4 packages)

| Package | Version | Maintainer |
|---|---|---|
| `rio` | 0.1.24.0 | Michael Snoyman |
| `generic-data` | 1.1.0.2 | Li-yao Xia |
| `ap-normalize` | 0.1.0.1 | Li-yao Xia |
| `show-combinators` | 0.2.0.0 | Li-yao Xia |

Builds without the flag exclude all 19 packages and their compiled code entirely.

## Risk profile

Including these 19 packages in the default build introduces risks across several dimensions.

### Supply chain breadth

The 19 packages come from approximately 7 distinct maintainer groups (Google, Well-Typed, Kazu Yamamoto, Russell O'Connor, Neil Mitchell, Michael Snoyman, Li-yao Xia).
Each maintainer account is an additional supply chain link: a compromised Hackage or CHaP credential, or a malicious commit to a source-repository-package, could inject code into the node binary.
The build flag eliminates these additional supply chain links for operators who do not need RPC.

### Network input parsing

Four of the 19 packages -- `grapesy`, `grpc-spec`, `http2`, and `proto-lens` -- parse untrusted input from gRPC clients.
Even behind a Unix domain socket, any local process (or an Envoy-exposed remote client) can send arbitrary bytes.
These are Haskell packages, so GHC's runtime prevents memory corruption (buffer overruns, use-after-free).
The realistic risks are unhandled exceptions that crash the RPC server and resource exhaustion from crafted inputs.
Since the RPC server runs in the node process, either would affect the node.
HTTP/2 implementations have a history of protocol-level denial-of-service vulnerabilities; the rapid-reset attack (CVE-2023-44487) affected most HTTP/2 implementations across languages in 2023.

### C foreign function interface

`snappy-c` binds Google's C++ snappy library via FFI, bypassing GHC's memory safety guarantees.
The decompression path allocates a buffer sized by the attacker-controlled input, which could cause excessive memory allocation.
`grpc-spec` enables snappy by default, so every gRPC-enabled build links the C library.

### Version coupling

`grapesy` pins `http2 == 5.3.9` (exact version).
If a security fix lands in a newer `http2` release, `grapesy` must cut a new release before the node can adopt it.
This creates a patching bottleneck: the node team cannot independently upgrade `http2` without coordinating with Well-Typed.

### Maintenance continuity

Several packages in the chain are maintained by a single individual.
`http2` and five related packages depend on Kazu Yamamoto (IIJ).
`lens-family` depends on Russell O'Connor.
`record-hasfield` depends on Neil Mitchell.
If any of these maintainers steps away, security patches and GHC compatibility updates stall.
The `proto-lens` family is nominally maintained by Google but has seen limited activity; the project uses a fork pinned via `source-repository-package` rather than Hackage releases.

### Risk-free packages

Not all 19 packages carry equal risk.
`generic-data`, `ap-normalize`, `show-combinators`, `lens-family`, `lens-family-core`, and `record-hasfield` are pure Haskell with no I/O, no FFI, and no network input parsing.
Their risk is limited to supply chain compromise of their maintainer accounts.

### Mitigation

The build flag is the primary mitigation: operators who do not enable RPC compile and link none of these 19 packages.
Block producers -- the highest-value targets on the network -- have no reason to enable the flag in production.
For operators who do enable RPC, the Unix domain socket transport and the recommendation to front it with Envoy (TLS, rate limiting, authentication) provide defence in depth at the network layer.

# Authors

- Mateusz Gałażyn (initial version)
