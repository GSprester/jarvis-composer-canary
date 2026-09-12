# orphan reclamation

**Orphan:** `qid` left the hot queue (`released`, ack, or drop) and **never launched** (no `promoted`, no live claim, Sig1 false). Re-`offer` of the same `qid` is a **dedup no-op** (`TASK-26-DEDUPE-BY-CONTENT.md`, `enqueue_id` / content hash). Work is gone from `list_next` and cannot be pushed again. **Fail if you mint a new `qid` “to help”:** if launch actually happened, second shell (`TASK-27-RETRY-SEMANTICS.md`). **Fail if you delete the `released` line:** replay and `R` break (`IDEMPOTENT-EVENT-LEDGER.md`, `BLAST-RADIUS-ESTIMATE.md`).

`qid` = `enqueue_id` (stable). Content-hash of the WO body may **equal** `qid` for dedup; **fail if hash includes `at_utc` or seat** — every retry is a new `qid` and this bug hides; **fail if hash is `name[:8]`.**

## Classify (MEASURE)

Canaries first (`POSITIVE-CONTROL.md`). Ledger look ERROR ⇒ stop (`LEGACY-38.md`).

```
orphan  iff  qid ∈ done ∪ (enqueued ∧ not queued ∧ not inflight)
             AND no promoted for qid in [0,hwm)
             AND not same(any claim on unit)
             AND not (Sig1 ∧ Sig2)
```

**Never-tried** (`LEGACY-30.md`): no `route_intent` / no lock rename. **Fail if `released` with empty `delivery` is treated as launched.** **Fail if `exists(path)` ⇒ launched** (`LEGACY-05.md`). **Fail if UNKNOWN lock ⇒ orphan:** steal/reclaim under a live holder (`LIVENESS-PREDICATE.md`). **Fail if Sig1 true, Sig2 false ⇒ orphan:** that is in-flight/crash, not never-launched (`STALE-CLAIM-RECOVERY.md`). **Fail if you classify from “errors in the last hour.”**

`DESIGN` crash_loop (`CRASH-LOOP-BRAKE.md`) is not an orphan. **Fail if you reclaim a DESIGN brake.**

## Reclamation path

Do **not** rewrite history. Append:

```
{type:reclaim, qid, unit, reason:never_launched,
 pred_released_event_id, grantor:wrapper,
 pid, start, at_utc}
```

Flush, HWM. Replay `apply(reclaim)`:

```
if not orphan-predicate (recomputed): UNKNOWN; ignore write
else: done.remove(qid); queued[qid]=original enqueued fields
```

Then `on_launch` may `promoted` the **same** `qid`. Lane: reclaim **re-inserts at head of its frozen lane** only if it was head when falsely released; else append (`ORDERED-RELEASE-QUEUE.md` — **fail if reclaim leapfrogs a real earlier promote**). **Fail if `apply` treats any `reclaim` as queued without recomputing orphan:** forges launch skip the other way.

Wrapper writes `reclaim`, not `W`, not the model (`DUAL-SIGNATURE-COMPLETION.md`). **Fail if `C` local gate writes it** (`DEPENDENCY-INVERSION.md`). **Fail if reclaim runs on a timeout of `G.measure`.**

If a **true** launch is discovered mid-reclaim (late `promoted` / live `same`): abort; `qid` stays done or inflight. **Fail if you continue and double-pop.**

Presence leftovers: archive first (`PROVENANCE-ARCHIVE.md`), then reclaim. **Fail if you reclaim while `C` still sees hot `exists`:** consumer and dispatcher disagree.

Quota: do not decrement `I` on the false `released`; do not decrement again on the real launch until Sig1∧Sig2 (`BUDGET-AWARE-ROUTING.md`).

Burst-review of covers uses the same classify; **fail if `P` ACK is reclaim** (`BURST-REVIEW.md`).

## Guard so it does not recur

1. **`released` only after Sig1∧Sig2** (or `stale_archived` of a **promoted** attempt). **Fail if release-on-promote or release-on-ack.** Tombstone that branch (`LEGACY-16.md`).
2. **`offer(qid)`:** if `qid` in `done` **and** last `released.delivery` is well-formed ⇒ no-op (real dedup). If `qid` in `done` **and** orphan-predicate ⇒ **refuse with `ORPHAN`**, do not no-op. Caller must `reclaim`, not “offer harder.” **Fail if both look like `duplicate`.**
3. Dual-run: dispatcher vs `G.measure` (`DEPENDENCY-INVERSION.md`). Mismatch `released` ∧ never-tried ⇒ `ERROR`, increment `orphan_suspect`. **Fail if mismatch prefers local done.**
4. Cheap decisive (`BLAST-RADIUS-ESTIMATE.md`): one `released` without `evidence_sha256` ⇒ all such `qid`s from that writer version are suspects. **Fail if you fix one file and close.**
5. `list_next` includes `ORPHAN` bag via `order_key` (`LEGACY-35.md`). **Fail if orphans vanish from the UI because they are “done.”**

**Rule:** Same `qid`. Prove never-launched, append `reclaim`, replay into `queued`. Dedup no-op is only for delivered work. Release-without-delivery is the recurrence.
