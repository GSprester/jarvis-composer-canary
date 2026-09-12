# substitute review queue

Primary seat `P` is silent for days. Substitutes `S1…Sn` cover **the same `unit`s**, not a parallel id. Covering is a normal claim after `same_process(P)` is **false** or UNKNOWN is **not** treated as dead (`LIVENESS-PREDICATE.md`). **Fail if days-old mtime ⇒ steal:** `P` may be in drain (`LEGACY-14.md`). **Fail if `P` still alive and `S` “just runs the shell”:** two leads (`LEGACY-01.md`). **Fail if cover mints a new `unit`:** duplicate delivery (`TASK-27-RETRY-SEMANTICS.md`).

Each cover produces a **review item**. `P`’s return is a cold admit (`TASK-05-DOCTRINE-DRAFT.md`). In-seat “we handled it” does not travel (`LEGACY-29.md`).

## Item identity

```
review_id = sha256(canonical({unit, enqueue_id, substitute, attempt_start}))
```

| Field | Rule |
|---|---|
| `unit` | WO `re`, not `YYYYMMDD` / filename (`TASK-07-DATE-ROLLOVER.md`) |
| `enqueue_id` | ledger id for this attempt (`IDEMPOTENT-EVENT-LEDGER.md`) |
| `substitute` | covering `seat` token |
| `attempt_start` | OS start of the covering lock |

**Fail if identity is `P`’s name + date:** UTC rollover and two covers one day collide. **Fail if identity is chat/thread id:** not durable, not 1:1 with a claim. **Fail if two covers share `enqueue_id`:** they were one attempt. Same `unit`, new `enqueue_id` after stale-claim archive is a **new** item (`STALE-CLAIM-RECOVERY.md`).

Priority on the item is body `priority` via `order_key` (`LEGACY-36.md`). **Fail if `list_next` sorts by who covered:** `P` reviews the wrong first row (`LEGACY-35.md`).

## Record location

Append-only JSONL **outside** the harness prefix and **outside** the payload (`LEGACY-21.md`, `LEGACY-29.md`):

```
review/<unit>/<review_id>.json     # snapshot after flush
ledger.jsonl                       # type=review_opened|review_closed
archive/…                          # if hot evidence was presence-blocking
```

Hot declared path stays the WO artifact. **Fail if the queue is `P`’s mailbox or model context:** offline days drop it. **Fail if the row is appended onto `out/<unit>`:** digest moves; `C` presence-gates (`PROVENANCE-ARCHIVE.md`). **Fail if location is `/tmp` or the substitute laptop:** `P` never sees it. Failed look at `review/` is ERROR, not “no covers” (`POSITIVE-CONTROL.md`).

## Writer

The **wrapper** of `S` appends `review_opened` **after** lock publish, flush, hwm — not the model (`TASK-05-DOCTRINE-DRAFT.md`). Fields: `review_id`, `unit`, `enqueue_id`, `substitute`, `pid`, `start`, `path`, `expected_sha256`, `at_utc`. **Fail if `S`’s model writes `status=covered`:** self-report (`TASK-17-SEPARATION-OF-DUTIES.md`). **Fail if `P` is the only writer of the queue:** `P` is offline, so no item exists and cover is invisible. **Fail if any `S` can `review_closed`:** they grade themselves (`LEGACY-17.md`).

`review_closed` writer = checker process (not `P`’s memory, not `S`). `P` may **MEASURE** (`GATE-DEADLOCK.md`); closing requires A∧B below.

## Verifiable vs asserted

Asserted: “`S` covered `unit`,” presence of a stub, LKG, or `review_opened` alone.

Verifiable:

```
closed  iff  A  sha256(hot or restored payload)==expected  (≠ empty)
             AND B  checker receipt cites that digest, re==unit
             AND review_id row exists (opened) with same expected
```

(`LEGACY-32.md`). Checker recomputes hashes. **Fail if `review_opened` ⇒ closed:** that is a claim, not a check. **Fail if `P` types ACK:** same. **Fail if fuzzy match of `S`’s notes to the WO** (`RECEIPT-MATCHING.md`). **Fail if `exists(path)` while `P` was away:** leftover (`LEGACY-05.md`).

Canary-hit/miss on the review scan (`LEGACY-39.md`). **Fail if empty `review/` after timeout ⇒ `P` has nothing to do.**

`P` returning does **not** delete items. Unverifiable covers stay UNVERIFIED; archive stubs; same `unit` may be reclaimed if `same(S)` is false. **Fail if `P` “takes back” a live `S` claim.** Crash-loop still applies.

Ceiling: `S` stays `min(S.ceiling, provider)` — **fail if covering inherits `P`’s ceiling** (`TASK-13-FAILOVER-CEILING.md`).

**Rule:** Same `unit`. Record in the estate log. Wrapper opens; checker closes. Covering is a claim until A∧B.
