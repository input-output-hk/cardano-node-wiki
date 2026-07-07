# ADR-020: Kupo Rewrite - Single-Threaded Pipelined Indexing Architecture

# Status

📜 Proposed 2026-07-06

# Context

Kupo is a chain-index: given a set of patterns, it tracks every UTxO on
Cardano matching those patterns — when it was created, when it was spent, and
by what. We maintain the IntersectMBO fork and are planning a reimplementation.

Today kupo:

- **follows the chain** over the node-to-client ChainSync protocol, decoding
  blocks from every era (Byron through Conway) from their era-specific CBOR;
- **matches transaction outputs** against user-configured patterns — full
  addresses, payment/delegation credentials, `policy_id.asset_name`,
  `tx_id@output_index`, and wildcards — and records the spend when a
  previously matched output is consumed;
- **rewinds the index on chain rollbacks**: ChainSync can announce that the
  node has switched forks, so every indexed row within the rollback window
  must be revertible;
- **resumes from checkpoints**: it stores the point of each processed block so
  a restart re-enters ChainSync at the last indexed point rather than
  resyncing;
- **serves queries over a REST API** backed by the match table;
- **garbage-collects rows whose retention has expired** (with `--prune-utxo`):
  spent entries are kept only so a rollback can un-spend them, and once a
  spend is deeper than the stability window (3k/f = 129,600 slots ≈ 36 hours
  on mainnet) it is deleted — finality-aware in that deletion is gated on the
  spend being immutable (mechanics in Decision 4).

Kupo's existing architecture is a two-thread producer/consumer design: a
producer thread receives and decodes blocks into a bounded mailbox (capacity
200), with rollbacks delivered through a separate single-slot `TMVar`. The
producer's ChainSync client is already pipelined (depth 100/5/1 by distance
to the tip). Because blocks and rollbacks arrive on two channels, the
mailbox has to put them back in order: a rollback is only handed to the
consumer once the block queue is empty, and the producer blocks while a
rollback is pending. This works, but it is machinery to restore an ordering
the protocol stream already had. Writes are batched by mailbox drain: each
consumer iteration takes everything pending — however much that is — and
commits it in one transaction; at the tip that is a single block. A third
thread — a timer-woken garbage collector — prunes spent rows when
`--prune-utxo` is set; it is a second SQLite writer, and a tip flush can
queue behind its deletes. The write policy is not the defect; the machinery
delivering it is.

The following forces bear on the design:

- **Overlapping fetch and processing is bounded by the slower stage.**
  Without overlap, each block costs the time to fetch it plus the time to
  process it; with overlap, the next block is fetched while the current one
  is processed, so each block costs only whichever of the two is slower. The
  gain is therefore at most 2×, reached only when the two costs are
  comparable — overlap adds no capacity to the slower stage.
- **Against a synced local node, processing dominates fetching.** The node
  serves blocks disk-to-wire without decoding them; the client touches the
  same bytes and additionally decodes, matches, and writes them. Fetching
  bounds a local client only when the node itself is not ready — still
  syncing, or replaying its ledger before opening the node-to-client socket
  — which no client-side buffering helps.
- **A pipelined client's replies buffer below the application.** With ~50
  `MsgRequestNext` in flight, replies accumulate in the node's send buffer,
  the kernel receive buffer, and the ouroboros-network mux ingress queue,
  which the mux demultiplexer drains on its own thread. A consumer that is
  keeping up finds the next block already in its own process —
  `CollectResponse` is a local queue pop, not a network wait — so fetching
  block N+1 overlaps processing block N with no application code. Requests
  the node has not yet answered are held as protocol state, not bytes.
  ChainSync is strictly request/response — the node sends one reply per
  `MsgRequestNext` and otherwise cannot send at all — so if the consumer
  stalls, it stops requesting, and at most `depth` uncollected replies exist
  between the node and the application. The mux ingress queue provides no
  pushback of its own: its node-to-client limit (`maximumIngressQueue`) is
  effectively unbounded (~4 GiB), and overrunning it kills the connection
  rather than throttling it. Pipelining depth is therefore the only bound on
  this path.
- **Decode timing is a client choice.** Node-to-client ChainSync ships blocks
  as `Serialised blk` (raw CBOR); the node serves them disk-to-wire without
  decoding. A client may decode eagerly, lazily, or partially.
- **SQLite write cost concentrates in commits, not inserts.** Inserting rows
  inside an open transaction is cheap; committing is the slow part. How slow
  depends on durability: a durable commit (`synchronous = FULL`) forces an
  fsync — more than ten million of them over a full mainnet sync if
  committing per block — while
  kupo's current `WAL` + `synchronous = NORMAL` settings defer fsync to WAL
  checkpoints, at the price that a power failure can lose the most recent
  commits.
- **At the tip, blocks arrive ~20 s apart.** A client that has caught up is
  idle most of the time, and anything not yet committed is invisible to
  queries.
- **Prior art.** `foldBlocks` in `cardano-api` was measured at 1h00m19s with
  a non-pipelined ChainSync client vs 46m23s pipelined
  ([cardano-node#2633](https://github.com/IntersectMBO/cardano-node/pull/2633),
  recorded in the note above `foldBlocks` in `Cardano.Api.LedgerState`) —
  single-threaded, no queue. Marconi
  (`marconi-core`) independently converged on the same batching and rollback
  policies this ADR adopts, but wrapped in a Coordinator/Worker apparatus
  (`TChan`/`TBQueue`/`MVar`/`QSemN`) that exists to fan one event stream out
  to N heterogeneous indexers. Kupo has N=1 and requires outputs, datums,
  scripts, and checkpoints to commit in one atomic SQLite transaction, which
  a per-worker split cannot provide.

# Decision

**Scope.** The rewrite preserves kupo's existing functional behaviour (the
capability list in Context); this ADR decides only the concurrency and
storage-write architecture. On-disk schema and resumption-format compatibility
with existing kupo databases will be decided in a separate ADR.

We will structure the rewrite as a **single-threaded application loop driven by
a pipelined ChainSync client**, with no application-level queue.

"Single-threaded" means one locus of effects on the indexing path: a single
thread of control sequences decode, match, write, and rollback in protocol
order. The process still contains the mux demultiplexer, the REST server's
handler threads, and RTS service threads; Decision 3's spark mitigation adds
parallel evaluation of *pure* decode without adding a second effectful agent.

1. **Drop the mailbox.** We will use `ChainSyncClientPipelined` with
   `pipelineDecisionMax` (initial depth ~50) and let the stack below the
   application act as the buffer. There is no consumer thread, no rollback
   `TMVar`, and no reordering to manage: a single consumer processing an
   in-order protocol stream cannot race itself. Pipelining depth
   will be chosen deliberately, as it is the only bound on this path — and
   it also bounds the drain a client-initiated rewind must collect before
   `MsgFindIntersect` (Decision 5).

2. **Batch database writes, flushing on idle.** After processing a block, if
   another response is already buffered (observable via `CollectResponse`'s
   `Maybe` continuation — a non-blocking peek), we keep batching up to a cap;
   if the pipeline is empty, we flush immediately and then wait for the next
   block (an *idle flush*, as opposed to a flush forced by the cap). This
   preserves current kupo's drain-batching policy; only the trigger changes —
   pipeline peek instead of mailbox drain. The other candidate triggers were
   rejected: a count cap leaves a near-empty batch open for minutes at the
   tip (stale, unqueryable data), a timer reintroduces a thread and a
   tunable, and mailbox drain — the current trigger — requires exactly the
   queue and thread being removed. Flushing on idle obtains drain semantics
   from the pipeline itself. The cap is
   expressed in bytes of pending writes rather than blocks: block sizes span
   three orders of magnitude, so a byte cap bounds memory where a count cap
   cannot. During bulk sync batches fill to the cap (one large transaction);
   at the tip every block flushes individually (~zero staleness). No runtime
   mode switch and no distance-to-tip function; the only knobs are static
   constants — pipelining depth, the batch cap, and Decision 4's GC cadence.
   Rollbacks that land inside the pending batch truncate an in-memory list; a
   rollback deeper than the batch cascades in the database. At the tip batches
   are ~1 block, so shallow tip rollbacks — the common case — take the database
   path, and we will test that path first-class. Checkpoint density (every
   block point stored) is preserved. Flushes commit durably
   (`synchronous = FULL`): one fsync per flush, affordable precisely because
   flushes are batched, so a crash — including power failure — redoes at
   most one batch and always restarts from a block boundary. This is a
   deliberate strengthening over current kupo, whose `WAL` +
   `synchronous = NORMAL` settings can lose the most recent commits on power
   failure. Staleness while saturated is bounded by the cap.

3. **No decode thread up front; mitigation must be earned by measurement.**
   The one job the old producer thread did that the buffer stack does not is
   paying CBOR deserialization on a second core. Rather than rebuild that
   thread speculatively, the rewrite starts with decode inline on the
   application loop, instrumented so the cost of each stage — decode, match,
   write — is visible. If decode stays well below match + write, nothing
   further is built. If it does not, mitigation escalates rung by rung,
   stopping at the first that restores headroom:

   1. **Zero-copy writes.** Ledger decoders memoize their input bytes
      (`MemoBytes`), so datums and scripts — which kupo stores as raw CBOR
      anyway — are written as slices of the block's own bytes rather than
      re-serialized. Slices are copied (`BS.copy`) as they enter the
      pending batch: a retained slice would otherwise pin its entire source
      block, silently breaking Decision 2's byte cap. The copy touches only
      matched data, tiny relative to blocks. Ledger-owned machinery with no
      drift surface; worth doing regardless.
   2. **Speculative decode via sparks.** `rpar` the decode of
      already-buffered responses onto an idle capability, so block N+1
      decodes in parallel with matching block N — no thread, no queue, no
      synchronization, never slower than baseline. This parallelises the
      unmodified ledger decode, so it also has no drift surface.
   3. **Custom partial decoders (future option, not committed).** Skipping
      parts of the block kupo does not need could shave further cost, but
      kupo consumes nearly the whole block — the ceiling is roughly the
      witness signatures — and custom decoders risk drifting from the
      ledger's. This option is recorded for future evaluation only; the
      delegation design and ledger-sync plan it would require are kept in
      the notes page, and it is considered only if measurement shows rungs
      1–2 insufficient and the remaining gain is judged worth the drift
      risk.
   4. **Last resort: one decode thread.** A single bounded FIFO carrying
      `RollForward blk | RollBackward pt` in-band (marconi's
      `ProcessedInput` shape), so queue order preserves protocol order. It
      reintroduces a thread, a queue, and a capacity tunable, which is why
      it comes last.

   Measurement methodology and each rung's mechanics (instrumentation
   options, era-specific decoder trade-offs, spark forcing and diagnostics)
   are recorded in [Kupo rewrite: decode mitigation
   notes](./Kupo-rewrite-decode-mitigation-notes.md).

4. **Garbage collection runs on the indexing thread.** With `--prune-utxo`,
   a spent entry is deleted once no rollback can resurrect it. Most
   deletions happen inline: when a spend is already deeper than the
   stability window at the time it is processed — during catch-up, every
   spend — the row is deleted in the same transaction as the batch that
   spends it. What is left for garbage collection is whatever could not be
   deleted inline: rows spent within the unstable window (marked rather
   than deleted, since a rollback could still un-spend them) that have
   since become final, plus orphaned binary data. Deleting all of these in
   one statement would hold the write lock for the whole delete, so each
   run deletes at most 50,000 rows in one transaction and stops; the rest
   waits for the next run. The deletes run on the indexing loop: after an
   idle flush, the loop runs at most one delete before it resumes waiting
   for the next block. At the tip this
   uses idle time — the next block is on average ~20 s away — and the
   50,000-row cap limits how long a newly arrived block can wait if one
   shows up mid-delete. During bulk sync the pipeline is never idle, so
   the delete instead runs every Nth flush (the GC cadence constant) —
   easily keeping pace, since catch-up leaves almost nothing to collect.
   The result is exactly one database writer; the REST server's handlers
   read WAL snapshots and never contend.

5. **API mutations enter through the loop.** The HTTP API can mutate the
   index: `PUT /patterns` inserts patterns together with a mandatory
   rollback point (a new pattern's history requires re-seeing the blocks),
   and `DELETE /patterns` removes a pattern and its matches. REST handlers
   never touch the database: a mutation is posted as a command into a
   shared control variable, and the handler blocks until the loop
   acknowledges the outcome. The loop checks the variable twice per cycle
   — at the flush point, and again just before waiting for the next reply
   — so only a command that arrives while the loop is already waiting is
   delayed, by at most one block interval at the tip. A deletion is
   applied in the loop's next transaction. An insertion first awaits the
   outstanding pipelined replies — `MsgFindIntersect` cannot be sent while
   any are in flight — then renegotiates the intersection at the requested
   point and rewinds the index in one transaction. Once the client reaches
   the server's tip, the depth policy stops sending new requests and
   awaits the outstanding ones, so at the tip — where operators act — at
   most one reply is in flight and a rewind is never blocked behind a deep
   pipeline; during bulk sync the outstanding replies arrive back-to-back
   and are discarded. Decision 4's single-writer property is preserved.

# Alternatives Considered

We rejected the **k-deep stability buffer** used by `foldBlocks` (hold the last
k blocks in memory, commit only what is k-deep, making rollback a pure in-memory
truncate): k is thousands of blocks — hours of wall clock — and anything not
flushed is invisible to queries, which is incompatible with "did my output
land?" latency.

We also rejected the **minimal repair of the current architecture**: keep the
producer thread and mailbox, and fix the ordering problem by moving rollbacks
into the block queue — nothing else would need to change. But what remains is
a thread and a queue that buy only two things: buffering, which the network
stack already provides, and decode on a second core, which Decision 3 adds
only if measurement shows it is needed. If they turn out unnecessary, nothing
ever removes them: unneeded threads and queues do not fail, they just stay.

We considered **marconi's architecture** and took its write policies but not
its machinery. Marconi (IOG's chain-indexing library) faced the same
problems and independently converged on the behaviours Decisions 2 and 3
produce, which is strong evidence they are the right ones:

- It batches writes while far from the tip (its `WithCatchup` layer buffers
  events in memory and submits them as bulk inserts) and switches to
  per-event writes once near the tip, so tip data is immediately queryable.
  The flush-on-idle rule yields both behaviours without marconi's distance
  function and its two tuning knobs.
- On rollback it first truncates its in-memory buffer of not-yet-written
  events; storage is only touched when the rollback reaches deeper than the
  buffer. Decision 2's pending batch behaves identically.
- Its worker queues carry rollbacks *in-band* — one sum type where rollbacks
  travel in the same FIFO as blocks, making ordering correct by
  construction. Decision 3's last-resort decode thread copies exactly this
  shape.

What we do **not** take is the machinery: the Coordinator/Worker concurrency
exists to fan one event stream out to N independent indexers, and kupo has
N=1 with a single-transaction atomicity requirement. We also reject its
`MixedIndexer` storage split (recent rollback-able events held in memory,
stable history on disk): every query would have to consult both stores and
merge the results, and kupo's one query shape — SQL over the matches table —
does not justify a permanent merge layer.

# Consequences

## Positive

- **Less machinery.** No queue, no second thread, no `TMVar`. Blocks and
  rollbacks are never split across two channels, so no code is needed to put
  them back in order.
- **Flow control end to end.** A slow consumer stops collecting and
  therefore stops requesting, and the node stops serving; in-flight memory
  is bounded by pipelining depth, with zero application buffering code.
- **Atomicity preserved.** Outputs, spent-marks, datums, scripts, and
  checkpoints continue to commit in one SQLite transaction per flush.
- **Self-tuning write behaviour.** Large transactions during bulk sync,
  per-block freshness at the tip, and burst absorption fall out of the
  flush-on-idle rule with no runtime tuning; the only constants are the
  pipelining depth, the batch cap, and the GC cadence.

## Negative

- **Decode runs on the application thread.** The buffer stack below us moves
  bytes but cannot run our code: CBOR deserialization is application work,
  and each block pays decode, then match, then write, in sequence on one
  core. The old design overlapped decode with match+write
  on its producer thread; we trade that overlap away, betting that decode is
  a minor term — if the bet is wrong, the single thread costs up to ~2× and
  we are committed to Decision 3's escalation. The spark mitigation is also
  a deployment requirement, not just a code change: under `-N1` there is no
  second capability, so sparks are silently evaluated on demand with zero
  speedup — kupo must run with `+RTS -N2` or more, and RTS flags are in
  operators' hands. Write batching actively steers the system toward this
  risk: shrinking one term of a sum grows every other term's share, so
  amortising fsyncs promotes decode's relative weight even though its
  absolute cost is unchanged. (Decode was never visible in the current
  design for a similar reason — it ran on a separate thread.) This is why
  measurement is Decision 3's first
  step: once batching lands, decode is the most probable next bottleneck.
- **Pipelining depth is load-bearing.** With `maximumIngressQueue` effectively
  unbounded on node-to-client (`0xffffffff` in current `ouroboros-network`),
  a stalled consumer holds up to depth × max-block-size bytes in the mux
  ingress queue — bounded, but linear in a constant we choose, with no
  independent backstop below it.
- **Pending batches are invisible to queries during bulk sync.** Accepted:
  during catch-up the index is stale regardless, and at the tip batches are
  bypassed/empty (marconi accepts the same trade). A crash during bulk sync
  additionally redoes up to one batch; both effects are bounded by the cap.
- **The database rollback path is tip-critical.** Because tip batches are ~1
  block, common shallow rollbacks nearly always hit the cascade-delete path;
  it must be covered by first-class tests, not treated as rare.
- **API mutations wait for the loop.** Mutations are applied only by the
  indexing thread, so a command posted while the loop is waiting for the
  next block is delayed by up to ~one block interval at the tip before it
  is acknowledged.

# References

- `ouroboros-network` `docs/network-spec/mux.tex` ("Flow-control and Buffering
  in the Demultiplexer") and `miniprotocols.tex` ("Pipelining of Mini
  Protocols") — buffering and depth-as-capacity semantics.
- `Ouroboros.Network.Protocol.ChainSync.{ClientPipelined,PipelineDecision}` —
  pipelined client machinery.
- `Ouroboros.Consensus.Network.NodeToClient` — codecs over `Serialised blk`.
- `cardano-api` `Cardano.Api.LedgerState.foldBlocks` — pipelined,
  single-threaded prior art; function and benchmark note introduced in
  [cardano-node#2633](https://github.com/IntersectMBO/cardano-node/pull/2633).
- `input-output-hk/marconi` `marconi-core` — `WithCatchup`, `MixedIndexer`,
  `Coordinator` (used as specification, not as a dependency).

# Authors

- Jordan Millar
