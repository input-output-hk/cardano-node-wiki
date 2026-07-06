# ADR-020: Kupo Rewrite — Single-Threaded Pipelined Indexing Architecture

## Status

📜 Proposed 2026-07-06

## Context

Kupo is a chain-index: given a set of address patterns, it tracks every UTxO on
Cardano matching those patterns — when it was created, when it was spent, and by
what. We maintain the IntersectMBO fork and are planning a reimplementation. The
functional requirements (chain follow, era-polymorphic block decoding, pattern
matching, rollback, checkpoint-based resumption, REST serving, finality-aware
garbage collection) are fixed; this ADR records the architectural decisions for
the rewrite's concurrency and storage-write design. On-disk schema and
resumption-format compatibility with existing kupo databases is out of scope
and will be decided separately.

Kupo's existing architecture is a two-thread producer/consumer design: a
producer thread receives and decodes blocks into a bounded mailbox (capacity
200), with rollbacks delivered through a separate single-slot `TMVar`. This
forces a manual ordering invariant — the consumer must drain pending blocks
before processing a rollback. Writes are already adaptively batched: each
consumer iteration drains the entire mailbox and commits everything drained
in one transaction, so batches grow toward 200 blocks when the consumer lags
and shrink to single blocks at the tip. The write policy is not the defect;
the machinery delivering it is.

The following forces bear on the design:

- **A decoupling queue only turns *sum* into *max*.** For a two-stage pipeline,
  throughput goes from `1/(t_fetch + t_process)` to `1/max(t_fetch, t_process)`
  — a bounded, at-most-2× win, realised only when fetch and process costs are
  comparable; when either side dominates, a queue cannot help, because it adds
  no capacity to the slow stage. Against a fully-synced local node, processing
  dominates: once pipelining hides round trips, fetch is a streaming cost — the
  node reads raw bytes from disk and copies them to a local socket — while the
  client touches the same bytes and additionally decodes, matches, and writes
  them. Operational experience bears this out: a node syncs mainnet from the
  network (validating every block) in under two days, while kupo and db-sync
  following an already-synced local node take several days, and a db-sync
  rewrite has cut historical sync several-fold through client-side changes
  alone. The cases where fetch does bound a local client are
  node-not-ready states — the node still syncing itself, or replaying its
  ledger before opening the node-to-client socket — which no client-side
  buffering can help with either. Where the stages are comparable, the
  sum→max overlap is already delivered by the pipelined client (next bullet);
  the only marginal value of a queue-plus-producer-thread is decode on a
  second core, addressed without a queue in Decision 3. At the tip the
  question dissolves: one block every ~20 s leaves both stages idle, and
  throughput is not the binding constraint.
- **The network stack already buffers.** A pipelined ChainSync client (~50
  `MsgRequestNext` in flight) causes blocks to accumulate in the node's send
  buffer, our kernel receive buffer, and the ouroboros-network mux ingress
  queue while the application thread is busy. Each reply flows as far toward
  the application as it can before something stalls it: the mux demultiplexer
  runs on its own thread, continuously draining the socket into the ingress
  queue, so a consumer that is keeping up finds the next block already
  resident in its own process — `CollectResponse` is a local queue pop, not a
  network wait. Requests the node has not yet answered cost nothing; they are
  held as protocol state, not bytes. Fetch of block N+1 therefore genuinely
  overlaps processing of block N — the kernel and the mux thread play the
  producer role. If the consumer stalls, the buffers fill from the ingress
  queue back toward the node, which then simply stops serving the remaining
  requests: bounded by pipelining depth and backpressured end to end.
  Note that node-to-client connections set `maximumIngressQueue` effectively
  unbounded, so pipelining depth is the *only* application-controlled bound.
- **Decode timing is a client choice.** Node-to-client ChainSync ships blocks
  as `Serialised blk` (raw CBOR); the node serves them disk-to-wire without
  decoding. A client may decode eagerly, lazily, or partially.
- **Per-block fsync would be the dominant write cost.** SQLite throughput is
  bounded by durable transaction commits (one fsync each), not by row
  inserts; a design committing per block would pay ~11M fsyncs over a
  mainnet sync. Kupo therefore already amortises: one transaction per
  mailbox drain, batching when behind and degenerating to per-block at the
  tip. Any replacement must preserve that policy, which raises the trigger
  question. Mainnet mints a block every ~20 s, so count-based triggers leave
  near-empty batches open for minutes at the tip (stale, unqueryable data),
  and timer-based triggers reintroduce a thread and a tunable — while
  mailbox-drain, the current trigger, requires exactly the queue and thread
  we are removing. Decision 2's flush-on-idle rule obtains drain semantics
  from the pipeline itself.
- **Prior art.** `foldBlocks` in `cardano-api`
  went from 1h00m19s to 46m23s by switching to pipelined ChainSync —
  single-threaded, no queue. Marconi (`marconi-core`) independently arrived at
  the same batching and rollback policies we want, but wrapped in a
  Coordinator/Worker apparatus (`TChan`/`TBQueue`/`MVar`/`QSemN`) that exists
  to fan one event stream out to N heterogeneous indexers. Kupo has N=1 and
  requires outputs, datums, scripts, and checkpoints to commit in one atomic
  SQLite transaction, which a per-worker split cannot provide.

## Decision

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
   `TMVar`, and therefore no drain-before-rollback invariant: a single consumer
   processing an in-order protocol stream cannot race itself. Pipelining depth
   will be chosen deliberately, as it is the only bound on this path.

2. **Batch database writes, flushing on idle.** After processing a block, if
   another response is already buffered (observable via `CollectResponse`'s
   `Maybe` continuation — a non-blocking peek), we keep batching up to a cap;
   if the pipeline is empty, we flush immediately and then wait for the next
   block (an *idle flush*, as opposed to a flush forced by the cap). This
   preserves current kupo's drain-batching policy; only the trigger changes —
   pipeline peek instead of mailbox drain. The cap is
   expressed in bytes of pending writes rather than blocks: block sizes span
   three orders of magnitude, so a byte cap bounds memory where a count cap
   cannot. During bulk sync batches fill to the cap (one large transaction);
   at the tip every block flushes individually (~zero staleness). No runtime
   mode switch and no distance-to-tip function; the only knobs are static
   constants — pipelining depth, the batch cap, and Decision 5's GC cadence.
   Rollbacks that land inside the pending batch truncate an in-memory list; a
   rollback deeper than the batch cascades in the database. At the tip batches
   are ~1 block, so shallow tip rollbacks — the common case — take the database
   path, and we will test that path first-class. Checkpoint density (every
   block point stored) is preserved, and durability follows flushes:
   checkpoints become durable per flush, so a crash redoes at most one batch
   and always restarts from a block boundary; staleness while saturated is
   bounded by the cap.

3. **Treat decode cost by measurement, then escalating mitigation.** The one
   job the old producer thread did that the buffer stack does not is paying
   CBOR deserialization on a second core. Rather than rebuild a thread for it
   speculatively, we will escalate only as far as measurement demands:

   - **Measure before mitigating.** Per-stage costs (decode / match / write)
     must be visible before any mitigation below is built. This does not
     require permanent infrastructure: eventlog markers (`traceEventIO`)
     around each stage, analysed with `ghc-events-analyze`, work on an
     ordinary build and can be retrofitted in an afternoon; alternatively,
     three `ekg` `Distribution`s hung off the Prometheus `/metrics` endpoint
     kupo already serves make the same numbers permanent for ~20 lines. If
     decode is well below match+write, we stop here and build nothing.
   - **Decode less.** CBOR is self-describing, so a decoder can skip a term
     without materializing its interior. Hand-rolled cborg decoders can
     extract only what kupo indexes — outputs, inputs, redeemer pointers,
     metadata labels, the header point — and skip witness sets and the rest,
     at the price of owning era-specific CBOR layouts (adoptable
     incrementally). Independently, and worth doing regardless: ledger
     decoders memoize their input bytes (`MemoBytes`/`originalBytes`), so
     datums and scripts — which kupo stores as raw CBOR anyway — can be
     written as zero-cost slices of the block's own bytes rather than
     round-tripped through rich types and re-serialized. This also removes a
     class of non-canonical-CBOR round-trip bugs.
   - **Speculative decode via sparks.** Collecting a response yields a cheap
     `Serialised blk`; `rpar` its decode thunk into a small lookahead deque.
     An idle capability steals and evaluates the spark, so with `-N2`+ the
     decode of block N+1 runs on a second core while block N is matched. (The
     spark must force past WHNF to the fields actually consumed — trivial
     once the "decode less" rung's strict record exists, another reason the
     rungs order this way.) Forcing an already-evaluated thunk is O(1); a spark that never ran is
     simply evaluated on demand — never slower than baseline. No thread, no
     queue, no synchronization; a sparked decode of a rolled-back block is
     discarded speculative work. Spark conversion rates (`+RTS -s`,
     ThreadScope) tell us whether this is actually helping.
   - **Last resort: one decode thread.** Only if decode rivals match+write
     and sparks do not convert reliably: a single bounded FIFO carrying one
     event sum type, `RollForward blk | RollBackward pt`, fed by a decode
     thread. Because rollbacks travel in-band (marconi's `ProcessedInput`
     shape), FIFO order preserves protocol order by construction — no second
     channel, no drain-first invariant. It is still a thread, a queue, and a
     capacity tunable, which is why it comes last.

4. **Treat marconi as validation of the write policies, not as a design to
   copy.** Marconi (IOG's chain-indexing library) faced the same problems and
   independently converged on the behaviours Decisions 2 and 3 produce, which
   is strong evidence they are the right ones:

   - It batches writes while far from the tip (its `WithCatchup` layer
     buffers events in memory and submits them as bulk inserts) and switches
     to per-event writes once near the tip, so tip data is immediately
     queryable. Our flush-on-idle rule yields both behaviours without
     marconi's distance function and its two tuning knobs.
   - On rollback it first truncates its in-memory buffer of not-yet-written
     events; storage is only touched when the rollback reaches deeper than
     the buffer. Decision 2's pending batch behaves identically.
   - Its worker queues carry rollbacks *in-band* — one sum type where
     rollbacks travel in the same FIFO as blocks, making ordering correct by
     construction. Decision 3's last-resort decode thread copies exactly this
     shape.

   What we do **not** take is marconi's machinery, for reasons stated in
   Context: the Coordinator/Worker concurrency exists to fan one event stream
   out to N independent indexers, and kupo has N=1 with a single-transaction
   atomicity requirement. We also reject its `MixedIndexer` storage split
   (recent rollback-able events held in memory, stable history on disk):
   every query would have to consult both stores and merge the results, and
   kupo's one query shape — SQL over the matches table — does not justify a
   permanent merge layer.

5. **Garbage collection runs on the indexing thread.** With `--prune-utxo`,
   a spent entry is deleted once no rollback can resurrect it. Most
   deletions happen inline: when a spend is already deeper than the
   stability window at the time it is processed — during catch-up, every
   spend — the row is deleted in the same transaction as the batch that
   spends it. What is left for garbage collection is whatever could not be
   deleted inline: rows spent within the unstable window (marked rather
   than deleted, since a rollback could still un-spend them) that have
   since become final, plus orphaned binary data. Deleting all of these in
   one statement would hold the write lock for the whole delete, so kupo
   deletes at most 50,000 rows per transaction and stops; the rest waits
   for the next run. Current kupo triggers these deletes from a dedicated
   thread woken by a timer; that thread is a second SQLite writer, and a
   tip flush can queue behind its deletes. The rewrite changes only when
   the deletes run: after an idle flush, the loop runs at most one such
   delete before it resumes waiting for the next block. At the tip this
   uses idle time — the next block is on average ~20 s away — and the
   50,000-row cap limits how long a newly arrived block can wait if one
   shows up mid-delete. During bulk sync the pipeline is never idle, so
   the delete instead runs every Nth flush (the GC cadence constant) —
   easily keeping pace, since catch-up leaves almost nothing to collect.
   The result is exactly one database writer; the REST server's handlers
   read WAL snapshots and never contend.

We rejected the **k-deep stability buffer** used by `foldBlocks` (hold the last
k blocks in memory, commit only what is k-deep, making rollback a pure in-memory
truncate): k is thousands of blocks — hours of wall clock — and anything not
flushed is invisible to queries, which is incompatible with "did my output
land?" latency.

We also rejected the **minimal repair of the current architecture** (keep the
producer thread and mailbox, move rollbacks in-band to fix the `TMVar`
ordering invariant; the write batching already exists): it is the smallest
diff, but what remains after the repair is a thread and a queue whose only
contribution besides transport — decode on a second core — is exactly what
Decision 3 provides on demand, with evidence, and whose buffering the network
stack already supplies. Starting simple and escalating on measurement
dominates starting complex and hoping to simplify later.

## Consequences

### Positive

- **Less machinery, fewer invariants.** No queue, no second thread, no
  `TMVar`, no manual ordering rule. The rollback-ordering invariant disappears
  structurally rather than being enforced.
- **Backpressure end to end.** A slow consumer stalls the producer with
  bounded memory, from SQLite back through the socket to the node, with zero
  application buffering code.
- **Atomicity preserved.** Outputs, spent-marks, datums, scripts, and
  checkpoints continue to commit in one SQLite transaction per flush.
- **Self-tuning write behaviour.** Large transactions during bulk sync,
  per-block freshness at the tip, and burst absorption fall out of the
  flush-on-idle rule with no runtime tuning; the only constants are the
  pipelining depth, the batch cap, and the GC cadence.

### Negative

- **Decode runs on the application thread.** The buffer stack below us moves
  bytes but cannot run our code: CBOR deserialization is application work,
  and the per-block loop is strictly `t_decode + t_match + t_write`,
  serialized on one core. The old design overlapped decode with match+write
  on its producer thread; we trade that overlap away, betting that decode is
  a minor term — if the bet is wrong, the single thread costs up to ~2× and
  we are committed to Decision 3's escalation. The spark mitigation is also
  a deployment requirement, not just a code change: under `-N1` there is no
  second capability, so sparks are silently evaluated on demand with zero
  speedup — kupo must run with `+RTS -N2` or more, and RTS flags are in
  operators' hands. Write batching actively steers the system toward this
  risk: shrinking one term of a sum grows every other term's share, so
  amortising fsyncs promotes decode's relative weight even though its
  absolute cost is unchanged. (The same masking is why decode was never
  visible in the current design — hidden behind a per-block fsync, and on
  another thread besides.) This is why measurement is Decision 3's first
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

## References

- `ouroboros-network` `docs/network-spec/mux.tex` ("Flow-control and Buffering
  in the Demultiplexer") and `miniprotocols.tex` ("Pipelining of Mini
  Protocols") — buffering and depth-as-capacity semantics.
- `Ouroboros.Network.Protocol.ChainSync.{ClientPipelined,PipelineDecision}` —
  pipelined client machinery.
- `Ouroboros.Consensus.Network.NodeToClient` — codecs over `Serialised blk`.
- `cardano-api` `Cardano.Api.LedgerState.foldBlocks` — pipelined,
  single-threaded prior art and measurement.
- `input-output-hk/marconi` `marconi-core` — `WithCatchup`, `MixedIndexer`,
  `Coordinator` (used as specification, not as a dependency).
- `IntersectMBO/cardano-db-sync` issue #1721 — public record of the
  client-side sync-time investigation behind the db-sync figures.
