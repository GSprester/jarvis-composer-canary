# Deferral (wait for a dependency)

**Status:** design only.  
**Depends on:** `DISPATCH-POOL-LATCH.md`, `TASK-27-RETRY-SEMANTICS.md`, `STALE-CLAIM-RECOVERY.md`, `ARTIFACT-DERIVED-LEDGER.md`.  
**Does not exist in this tree:** deferral table, wake probe, cancel API.

Work that **cannot run now** but **must run when a named dependency returns** is **deferred**. It is not failed, not COMPLETED, not a new `unit`. Deferral is a wrapper row. The worker does not write it (`TASK-17`). Sleep, “try later,” and unmarked `queue_full` are not this mechanism (`TASK-25`, `ABSENT-OUTPUT-STATE.md`).

## Trigger

A completed look: the unit is still **admissible** (`form(C)` holds, `MECHANICAL-ACCEPT-SPLIT.md`) and a **named** dependency `dep` is **not ready**. Failed look of `dep` is UNKNOWN — do not defer, do not fail (`TASK-20-FAIL-LOUD.md`).

```
may_defer(unit, dep)  iff  look(unit) completed AND form(C_unit)
                           AND look(dep) completed AND ready(dep) == false
                           AND NOT said(unit)          # no goodbye
                           AND NOT cancelled(unit)
```

| `dep` (examples) | `ready` |
|---|---|
| Worker slot / pool `G` | `free≥1` and not `shared_429` (`DISPATCH-POOL-LATCH.md`, `SHARED-RATE-CEILING.md`) |
| Upstream `unit_u` | `derive` yields `released` / verify COMPLETED |
| Grant / token | unexpired row, `would_deny` still true (`APPROVAL-TOKEN-TTL.md`) |
| Artifact / key bind | `verify` or `bound` (`LEGACY-24.md`) |

```
{v:1, type:deferred, unit, enqueue_id,     # same ids (TASK-27)
 dep_id, dep_digest,                       # snapshot for staleness
 reason, policy_sha256,
 pid, start, at_utc, expires_at_utc}       # null expiry = standing; forbidden here
```

`enqueue_id` is unchanged. A new id is a second effect. `dep_digest` is `sha256` of the **declared** obligation we are waiting on (handoff `expected` of `dep`, latch fingerprint, grant `policy_sha256`) — not mtime. Class the row **reserved** in the hot table (`MEMORY-RESERVE.md`) so wrap does not drop the wake.

Do not defer on 402 of **this** unit’s entitlement (that is escalate, `TASK-28`). Do not defer because the worker thinks it is blocked (`T`).

One deferral per `enqueue_id`. Repeat ticks are the same `event_id` minus clock (`IDEMPOTENT-EVENT-LEDGER.md`).

## Staleness check (on return)

Wake is a **MEASURE of `dep` then of the unit**, not a timer. `Retry-After` / latch reset may **schedule** the look; the look still decides.

```
wake(unit)  iff  look(dep) completed AND ready(dep)
                 AND look(deferral row) completed
                 AND NOT cancelled
                 AND now_utc < expires_at_utc
                 AND form(C_unit) still holds
                 AND handoff.expected == expected at defer
                 AND current dep_digest == deferred.dep_digest
                     OR dep is “any completion of unit_u”
                        AND successor_id names the new digest
                 AND same enqueue_id not already released
```

| Fail | Reading |
|---|---|
| `expected` changed / `form(C)` fails | Obligation moved; old deferral is stale |
| `dep_digest` ≠ snapshot and no `successor_id` | Dependency was replaced, not “returned” |
| Expired TTL | Standing wait is a missing cancel (`APPROVAL-TOKEN-TTL.md`) |
| Row evicted, no archive | UNKNOWN — not “dep returned, run now” (`TASK-09`) |
| `ready` UNKNOWN | Do not launch |
| `cancelled` row after the deferral | See below |

Stale ⇒ do **not** offer. Append `type=deferral_stale` `{unit, reason}`. Unit stays UNVERIFIED. Issuer re-declares or cancels. Do not launch the old payload against a new dep (two leads / wrong `expected`).

After `wake`: offer the **same** `enqueue_id` (head rules still apply, `ORDERED-RELEASE-QUEUE.md`). One offer. If `dep` is again not ready, defer again (same ids, new `dep_digest` if the snapshot legally advanced via `successor_id`).

Canary-hit: planted `dep` becomes ready, snapshot matches → one launch.  
Canary-miss: `dep` ready but `expected` drifted → must **not** launch.

## Cancellation

Cancel is an append, not an unlink (delete looks like never deferred).

```
{v:1, type:deferral_cancelled, unit, enqueue_id,
 pred_event_id,              # the deferred row
 reason,                     # issuer_abort | dep_retired | unverifiable | grant_expired
 grantor, pid, start}        # wrapper; not the worker
```

| Who | When |
|---|---|
| Issuer / grantor | Work must not run even if `dep` returns |
| Wrapper | `dep` tombstoned / retired (`REMOVAL-FIX-EVIDENCE.md`); `form(C)` failed; 402 on this unit; `expires_at` passed |
| Never the seat | A model “never mind” is `model_guess` |

After cancel: `wake` is false **even if** `ready(dep)`. Do not fail-close siblings. Do not mark COMPLETED. Archive any hot leftover (`STALE-CLAIM-RECOVERY.md`). Same `unit` may be **re-admitted** only with a new `enqueue_id` after `released.kind=stale_archived` of the deferred attempt — not by deleting `deferral_cancelled`.

Cancel of `dep` (quarantine, retire) must cancel or stale every deferral that named that `dep_id` (bag walk, not “hope the wake fails”). Failed walk → UNKNOWN; do not launch those units.

## What this must not do

- Fail the unit because `dep` is not ready.
- Wake on the clock alone, or on empty stdout.
- Mint a new `unit` when `dep` returns.
- Launch when `expected` or `dep_digest` drifted.
- Unlink the deferral row to cancel.

**Rule:** Defer when `form(C)` holds and a named `dep` is not ready (completed looks). Wake only if `dep` is ready **and** `expected` / `dep_digest` still match. Cancel is an append that blocks wake; it is not COMPLETED and not a delete.
