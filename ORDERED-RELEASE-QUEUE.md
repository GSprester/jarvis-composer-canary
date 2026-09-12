# ordered release queue

A **lane** is `effective_priority` **frozen at `promoted`** (`LEGACY-34.md`). Aging must not reorder in-flight work. **Fail if you recompute `effective_priority` at release:** a waiting `p3` ages to `p0` and leapfrogs a live `p1`. **Fail if the lane is filename `p{n}`** (`LEGACY-11.md`).

**Earliest-promote wins** inside a lane: the first flushed `promoted` event in `[0,hwm)` is first to be allowed `released` (`IDEMPOTENT-EVENT-LEDGER.md`). File order, not `at_utc` sort. **Fail if you sort by clock:** skew applies `released` before an earlier promote. **Fail if you use `scheduled_utc` or enqueue time:** never-popped work blocks (or loses to) a later pop (`LEGACY-30.md`). **Fail if you use `ls` / mtime** (`LEGACY-35.md`).

`released` still requires dual-signature completion first (`DUAL-SIGNATURE-COMPLETION.md`). **Fail if order is enforced only on `verify` but `released` can skip:** consumers that watch the ledger leapfrog. **Fail if `COMPLETED` is published before `released`:** same leak.

## Ordering rule

Replay builds, per lane `L`:

```
lane[L] = [enqueue_id, …]  # order of promoted events, first = head
```

`may_release(id)` iff

```
id is head of lane[id.L]
AND Sig1 ∧ Sig2 for id
AND same(holder) is false or holder is checker   # not required live
```

Append `released` only then; drop `id` from `lane[L]`; next head may release. **Fail if you release any Sig1∧Sig2 in the lane:** earliest-promote is dead. **Fail if head is Sig1∧Sig2-failed and you skip it silently:** the rest stuck; must stale-archive the head (`STALE-CLAIM-RECOVERY.md`) then pop it out of the lane with `released.kind=stale_archived`. **Fail if stale-archive of the head does not go through `released`:** successors still see it as head.

Tie: two `promoted` in one flush — **fail if undefined.** Require one event per append; HWM between. **Fail if in-memory pop order ≠ ledger promote order.**

`list_next` for release = heads of lanes in `order_key` across lanes (lane 0 first), then remaining live queue. **Fail if UI shows “whichever finished.”**

Canaries on every `may_release` look (`LEGACY-39.md`). **Fail if empty ledger look ⇒ all may release** (`POSITIVE-CONTROL.md`).

## Leapfrog prevention

A later `enqueue_id` in the same frozen lane **must not** `released` while an earlier one is still in `lane[L]`. Checker `K` refuses Sig2/`released` write if `may_release` is false — MEASURE can still compute Sig1 (do not block hashing; **fail if you deadlock `verify`** — `GATE-DEADLOCK.md`).

**Fail if `W` of a later unit writes a `DONE` filename:** that is not `released` (`LEGACY-05.md`). **Fail if burst `review_closed` implies `released` out of order** (`BURST-REVIEW.md`). **Fail if a new cover `enqueue_id` is prepended:** append on promote (`SUBSTITUTE-REVIEW-QUEUE.md`). **Fail if routing rewrites promote sequence** (`BUDGET-AWARE-ROUTING.md`).

Crash: later unit finishes bytes first. Bytes may sit; `released` waits. **Fail if presence-gated consumer takes the later file** (`PROVENANCE-ARCHIVE.md`): `C` must wait on ledger head, not `exists`.

## Owner override

Owner `P` (or grantor) cannot rename `p0-…` or edit `priority`. Override is an exemption row (`LEGACY-22.md`):

```
{type:release_override, unit, enqueue_id, lane, predecessor_id,
 grantor, pid, start, policy_sha256, would_deny:true,
 deny_reason:leapfrog, expires_at_utc}
```

`would_deny` is recomputed: `may_release` was false. Wrapper writes it (not `W`, not the model). Flush, then `released` of **this** `enqueue_id` only. **Fail if override is a bool on the WO.** **Fail if one override releases the whole lane.** **Fail if `P` ACK in chat is the override** (`EVIDENCE-TIERING.md`). **Fail if override omits `predecessor_id`:** cannot audit who was skipped. **Fail if `G` timeout ⇒ local override** (`DEPENDENCY-INVERSION.md`).

Override still needs Sig1∧Sig2. **Fail if override skips the MAC.** Count overrides in `R` if abused (`BLAST-RADIUS-ESTIMATE.md`).

**Rule:** Freeze the lane at promote. Release the head only, after both signatures. Override is a granted, single-id exemption that still cites the skipped predecessor.
