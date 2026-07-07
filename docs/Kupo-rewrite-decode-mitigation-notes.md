# Kupo Rewrite: Decode Mitigation Notes

Companion notes to
[ADR-020](./ADR-020-Kupo-rewrite-indexing-architecture.md), Decision 3 ("no
decode thread up front; mitigation must be earned by measurement"). The ADR
records the decision and the escalation ladder; this page records the
measurement methodology and the mechanics of each rung. It is implementation
guidance, not a decision record.

## Measurement

Per-stage costs (decode / match / write) must be visible before any
mitigation is built. This does not require permanent infrastructure:

- **Eventlog markers.** `traceEventIO` around each stage, analysed with
  `ghc-events-analyze`, works on an ordinary build and can be retrofitted
  quickly.
- **Prometheus metrics.** Three `ekg` `Distribution`s hung off the
  `/metrics` endpoint kupo already serves make the same numbers permanent
  for a few dozen lines of code.

If decode is well below match + write, stop: build nothing.

## Rung 1: zero-copy writes

Ledger decoders memoize their input bytes (`MemoBytes`/`originalBytes`), so
datums and scripts — which kupo stores as raw CBOR anyway — can be written
as slices of the block's own bytes rather than round-tripped through rich
types and re-serialized. This also removes a class of non-canonical-CBOR
round-trip bugs. The transaction id comes free the same way: it is the
ledger's `SafeHash` of the body's original bytes.

One retention trap to design around: a `ByteString` slice keeps its parent
buffer alive, so a 100-byte datum slice held in the pending write batch
would pin its entire source block (often tens of KB) until the next flush —
and Decision 2's cap, which counts bytes of pending writes, would
undercount actual retained memory by up to three orders of magnitude.
Slices are therefore `BS.copy`'d as they enter the pending batch: the copy
touches only matched data, tiny relative to blocks, and no slice outlives
the block that backs it. Zero-copy through decode and match; one small copy
at batch admission.

Entirely ledger-owned machinery, available from the full decode kupo
already performs: no drift surface.

## Rung 2: speculative decode via sparks

Collecting a response yields a cheap `Serialised blk`; `rpar` its decode
thunk into a small lookahead deque. An idle capability steals and evaluates
the spark, so with `-N2` or more the decode of block N+1 runs on a second
core while block N is matched.

- The spark must force past WHNF to the fields actually consumed — a small
  forcing function over the typed block.
- Forcing an already-evaluated thunk is O(1), and a spark that never ran is
  simply evaluated on demand — never slower than baseline.
- No thread, no queue, no synchronization; a sparked decode of a rolled-back
  block is discarded speculative work.
- Spark conversion rates (`+RTS -s`, ThreadScope) tell us whether this is
  actually helping.
- Deployment requirement: under `-N1` there is no second capability and
  sparks are silently evaluated on demand with zero speedup; kupo must run
  with `+RTS -N2` or more (recorded in ADR-020's negative consequences).

This parallelises the unmodified ledger decode: no drift surface.

## Rung 3 (future option, not committed): custom partial decoders

Not part of the committed design. Recorded here so a future evaluation
starts from this analysis rather than from scratch.

- **The ceiling is low.** Kupo's patterns and stores consume nearly the
  whole block: transaction bodies (inputs, outputs, collateral, mint),
  witness-set datums, redeemers and scripts, auxiliary-data scripts and
  metadata labels. What a partial decoder can actually skip is roughly the
  vkey and bootstrap witnesses.
- **If ever built, maximise ledger delegation.** Hand-roll only the block
  skeleton (one list arity and the component order per era) and the
  witness-set key table; decode every kept component with the ledger's own
  `Annotator`-based `decCBOR` — whole `TxBody`s included, so future body
  fields flow through automatically — and skip unknown keys structurally.
  The owned drift surface is then the era dispatch, the skeleton constants,
  and one small add-only key table.
- **The sync plan it would require.** Decode-equivalence property tests
  (full ledger decode vs the partial path, asserted equal over the ledger's
  era generators and sampled mainnet blocks); loud runtime fallback to the
  full decoder on any structural surprise; and, as the preferred long-term
  home, upstreaming the projection decoders into `cardano-ledger` itself so
  drift is caught where changes are made.

## Rung 4 (last resort): one decode thread

Only if decode rivals match + write and the rungs above do not close the
gap: a single bounded FIFO carrying one event sum type,
`RollForward blk | RollBackward pt`, fed by a decode thread. Because
rollbacks travel in the same queue as blocks (marconi's `ProcessedInput`
shape), FIFO order preserves protocol order by construction — no second
channel, no reordering. It is still a thread, a queue, and a capacity
tunable, which is why it comes last.
