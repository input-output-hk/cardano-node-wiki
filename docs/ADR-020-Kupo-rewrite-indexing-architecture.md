# ADR-020: Kupo Rewrite - Single-Threaded Pipelined Indexing Architecture

# Status

📜 Proposed 2026-07-06

# Context

Kupo is a chain-index: given a set of patterns, it tracks every UTxO on
Cardano matching those patterns — when it was created, when it was spent, and
by what. We maintain the IntersectMBO fork and are planning a reimplementation.

Today kupo:

- **follows the chain** from one of three producers — a local node socket,
  ogmios, or a Hydra head (the latter two over WebSockets) — decoding blocks
  from every era;
- **matches transaction outputs** against user-configured patterns (addresses,
  payment/delegation credentials, assets, output references, metadata tags)
  and records the spend when a matched output is consumed;
- **rewinds the index on rollbacks** — announced by ChainSync when the node
  switches forks, or forced by an operator: `PUT /patterns` takes a mandatory
  rollback point, because a new pattern's history exists only in the chain,
  not in the index;
- **resumes from checkpoints**, **serves queries over a REST API**, and, with
  `--prune-utxo`, **garbage-collects spent rows**: a spent output's row stays
  in the table, marked spent, so a rollback can un-spend it; once the spend
  is deeper than the stability window (3k/f = 129,600 slots ≈ 36 hours on
  mainnet), no rollback can reach it and the row is deleted.

The existing architecture is a two-thread producer/consumer design: a
producer thread receives and decodes blocks into a bounded mailbox, with
rollbacks on a separate single-slot channel and merge logic restoring an
ordering the protocol stream already had. The ChainSync client is already
pipelined (depth 100/5/1 by distance to the tip). Writes are batched by
mailbox drain: everything pending commits in one transaction — a single block
at the tip. A third, timer-woken thread prunes spent rows; it is a second
SQLite writer, and a tip flush can queue behind its deletes. HTTP handlers
that mutate patterns write from their own connections as well.

The following forces bear on the design:

- **Overlapping fetch and processing is bounded by the slower stage.**
  Without overlap, each block costs the time to fetch it plus the time to
  process it; with overlap, the next block is fetched while the current one
  is processed, so each block costs only whichever of the two is slower.
  The gain is therefore at most 2×, reached only when the two costs are
  comparable — overlap adds no capacity to the slower stage.
- **Against a synced local node, processing dominates fetching.** Fetching
  from a local node is a streaming cost: the node reads raw bytes from disk
  and copies them to a local socket, decoding nothing. The client touches
  the same bytes and must additionally CBOR-decode them, match every output
  against the patterns, and write the matches durably. Fetching bounds a
  local client only when the node itself is not ready — still syncing, or
  replaying its ledger before opening the node-to-client socket — which no
  client-side buffering helps.
- **ChainSync is strictly request/response, and a pipelined client's replies
  buffer below the application.** With ~50 requests in flight, replies pile
  up in the network stack (node send buffer, kernel, mux ingress queue), so
  a consumer that is keeping up finds the next block already in its own
  process — fetch overlaps processing with no application code. The node can
  only reply to outstanding requests, so a stalled consumer stops requesting
  and at most depth replies are ever in flight. The mux ingress limit is no
  bound: it is effectively unbounded on node-to-client, and overrunning it
  kills the connection rather than throttling it.
- **Decode timing is a client choice.** Blocks arrive as raw CBOR
  (`Serialised blk`); a client may decode eagerly, lazily, or partially.
- **SQLite writes are cheap until you commit.** Rows inserted inside an
  open transaction only touch memory; the commit is the expensive step,
  because surviving a power failure requires an fsync — a wait on the
  storage hardware itself. Committing durably per block would cost more
  than ten million fsyncs over a mainnet sync, so writes must be batched.
  Kupo today runs `WAL` + `synchronous = NORMAL` instead: no fsync per
  commit, but a power failure can lose the most recent commits.
- **At the tip, blocks arrive ~20 s apart**, and anything not yet committed
  is invisible to queries.
- **Prior art.** `foldBlocks` in `cardano-api` was measured at 1h00m19s with
  a non-pipelined ChainSync client vs 46m23s pipelined
  ([cardano-node#2633](https://github.com/IntersectMBO/cardano-node/pull/2633))
  — single-threaded, no queue. Marconi (`marconi-core`), IOG's chain-indexing
  library, solves the same problem and independently converged on the same
  write policies.

# Decision

We will structure the rewrite as a **single-threaded application loop driven
by a pipelined ChainSync client, batching database writes and flushing on
idle** — no application-level queue, no second effectful thread on the
indexing path. The single thread is the point: one thread sequencing decode,
match, write, and rollback in protocol order cannot race itself, needs no
cross-thread ordering or synchronization, and leaves exactly one database
writer.

- **The stack is the buffer.** `ChainSyncClientPipelined` with a fixed depth
  (~50); replies buffer in the network stack below the application. Depth is
  chosen deliberately: it is the only bound on this path, and it bounds the
  drain a client-initiated rewind must collect before `MsgFindIntersect`.
- **Flush on idle.** After each block, if another message is already
  buffered, keep batching toward a byte cap; if not, flush and wait. This
  preserves the existing drain-batching policy — large transactions during
  bulk sync, per-block freshness at the tip — with the pipeline itself as
  the trigger. Flushes commit durably (`synchronous = FULL`): one fsync per
  flush, affordable because flushes are batched, so a crash — including
  power failure — redoes at most one batch and restarts from a block
  boundary.
- **No decode thread up front.** Decode runs inline, instrumented per stage;
  mitigation is added only as measurement demands, in order: zero-copy
  writes, speculative decode via sparks, custom partial decoders (a future
  option, not committed — they risk drifting from the ledger's decoders),
  and, last, a decode thread.
- **Garbage collection runs on the indexing thread**, in bounded increments
  scheduled into tip idle time, preserving the single writer.
- **API mutations enter through the loop.** Handlers post commands to a
  control variable the loop checks every cycle, and never touch the
  database. A forced rewind first collects the outstanding pipelined
  replies, then renegotiates the intersection; the depth policy has already
  drained the pipeline by the time the client is at the tip, so a rewind is
  never blocked behind a deep one.

**Scope.** The rewrite preserves kupo's functional behaviour (the capability
list in Context); this ADR decides the concurrency and storage-write
architecture. Decided separately: on-disk schema and resumption-format
compatibility, and whether the ogmios and Hydra producers are retained — if
they are, the flush trigger generalises unchanged ("is another message
already buffered?", asked of the WebSocket client's receive queue).

# Alternatives Considered

We rejected the **k-deep stability buffer** used by `foldBlocks` (hold the
last k blocks in memory, commit only what is k-deep, making rollback a pure
in-memory truncate): k is thousands of blocks — hours of wall clock — and
anything not flushed is invisible to queries, which is incompatible with
"did my output land?" latency.

We also rejected the **minimal repair of the current architecture**: keep the
producer thread and mailbox, and fix the ordering problem by moving rollbacks
into the block queue — nothing else would need to change. But what remains is
a thread and a queue that buy only two things: buffering, which the network
stack already provides, and decode on a second core, which the decode policy
adds only if measurement shows it is needed. If they turn out unnecessary,
nothing ever removes them: unneeded threads and queues do not fail, they just
stay.

# Consequences

## Positive

- **Less machinery.** No queue, no second thread, no merge logic. Blocks and
  rollbacks are never split across two channels, so no code is needed to put
  them back in order.
- **Flow control end to end.** A slow consumer stops collecting and
  therefore stops requesting, and the node stops serving; in-flight memory
  is bounded by pipelining depth, with zero application buffering code.
- **Atomicity preserved.** Outputs, spent-marks, datums, scripts, and
  checkpoints continue to commit in one SQLite transaction per flush.
- **Self-tuning writes.** While catching up, writes accumulate into large
  transactions, which is what makes sync fast; once caught up, every block
  is committed the moment it arrives, so queries are never stale. The
  switch between the two happens by itself — the loop just notices whether
  another block is already waiting — and there is nothing to configure or
  tune at runtime: the only fixed numbers anywhere are the pipelining
  depth, the byte cap, and the GC cadence.

## Negative

- **Decode runs on the application thread.** The old design overlapped
  decode with match+write on its producer thread; we trade that away,
  betting that decode is a minor term. If the bet is wrong, the loop runs
  up to ~2× slower until a rung of the decode escalation ladder is applied.
  Write batching makes the bet worse, not better: batching shrinks the
  per-block cost of writes, so whatever decode costs becomes a larger share
  of each loop iteration. The ladder's spark rung also carries a deployment
  requirement: sparks only run in parallel given a second capability
  (`+RTS -N2` or more). On a single capability they do not fail — each
  spark is quietly evaluated by the indexing thread itself when its result
  is demanded — so a misdeployment shows up as zero speedup, not as an
  error.
- **If the loop stalls, arriving blocks pile up in memory, and the only
  limit on the pile is the depth constant.** The client keeps ~50 requests
  outstanding at all times, so even while the loop is stuck the node still
  answers all ~50, and those blocks sit in this process's memory until the
  loop resumes — worst case ~50 × 90 KB ≈ 4.5 MB. Harmless, but only
  because 50 is small: no other layer imposes any limit (the next one down
  is ~4 GiB and closes the connection when reached).
- **Pending batches are invisible to queries during bulk sync.** Accepted:
  during catch-up the index is stale regardless, and at the tip batches are
  ~1 block. A crash redoes up to one batch; both effects are bounded by the
  cap.
- **The database rollback path is tip-critical.** Tip batches are ~1 block,
  so common shallow rollbacks nearly always hit the cascade-delete path; it
  must be covered by first-class tests, not treated as rare.
- **API mutations wait for the loop**, by up to ~one block interval at the
  tip before they are acknowledged.

# References

- [`ouroboros-network` network specification](https://github.com/IntersectMBO/ouroboros-network/tree/main/docs/network-spec)
  — flow-control and buffering in the demultiplexer; pipelining of mini
  protocols.
- [`Ouroboros.Network.Protocol.ChainSync.ClientPipelined`](https://github.com/IntersectMBO/ouroboros-network/blob/main/ouroboros-network/protocols/lib/Ouroboros/Network/Protocol/ChainSync/ClientPipelined.hs)
  — pipelined client machinery.
- [`Cardano.Network.NodeToClient`](https://github.com/IntersectMBO/ouroboros-network/blob/main/cardano-diffusion/lib/Cardano/Network/NodeToClient.hs)
  — node-to-client mini-protocol limits.
- [`Cardano.Api.LedgerState.foldBlocks`](https://github.com/IntersectMBO/cardano-api/blob/master/cardano-api/src/Cardano/Api/LedgerState.hs)
  — pipelined, single-threaded prior art; function and benchmark note
  introduced in
  [cardano-node#2633](https://github.com/IntersectMBO/cardano-node/pull/2633).
- [`input-output-hk/marconi`](https://github.com/input-output-hk/marconi)
  `marconi-core` — `WithCatchup`, `MixedIndexer`, `Coordinator` (used as
  specification, not as a dependency).

# Authors

- Jordan Millar
