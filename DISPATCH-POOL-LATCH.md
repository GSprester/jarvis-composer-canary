# Dispatcher pool-exhaustion latch

**Status:** design only.  
**Depends on:** `TASK-25-QUEUE-BACKPRESSURE.md`, `CRASH-LOOP-BRAKE.md`, `TASK-08-WATCHDOG-PATTERN.md`, `SILENT-WORKER-STOP.md`.  
**Does not exist in this tree:** dispatcher, worker pool, latch store.

A dispatcher `D` offers units to a pool of `K` workers. When the pool cannot take another unit, the remaining bag is **not failed**. Those units did not miss `C`. The pool is exhausted. `D` **latches**: stop offering, leave the bag queued, speak **one** line.

Failing each remainder is a false `failed_both` storm (`TWO-LANE-SPLIT.md`), wraps the ledger (`TASK-22`), and invites retry of work that never ran (`TASK-27`).

## Exhaustion (completed look)

```
free     =  count of worker slots with no live {pid, start}
busy     =  count with same_process true
dead     =  K − busy − free   # claimed but same == false; or K == 0
exhausted iff  look(pool) completed AND (free == 0 OR entitlement 402)
```

| Kind | Look | Not this |
|---|---|---|
| `busy` | `free==0` and `busy==K` | Slow consumers; backpressure |
| `dead` | `busy==0` and no free incarnation | Silent worker stop of the **pool** |
| `quota` | 402 / entitlement on the pool hop | 429 — backoff, do not latch as dead (`TASK-28`) |

Look timeout / unreadable `/proc` / unknown `K` → **UNKNOWN**. Do not latch. Do not fail remainders. Do not offer (`TASK-20-FAIL-LOUD.md`).

`len(queue)==0` is not capacity. `errors==0` is not capacity (`SILENT-FAILURE-DETECTION.md`).

## The latch

After the first completed `exhausted` in an episode, the wrapper (not a worker, not `D`’s “fail” branch) appends then flush:

```
{v:1, type:pool_latch, pool_id, kind, K, free, busy,
 n_remaining,                          # bag size still queued, not failed
 would_deny:true,                      # recomputed: exhausted still held at write
 dispatcher_pid, start, at_utc, policy_sha256}
```

`would_deny` is “an offer now would not get a worker.” If a free slot exists, this is not a latch.

**While the latch row is the live episode** (same fingerprint `{pool_id, kind, K}`):

```
on_tick:
    if latch live AND look(pool) completed:
        do not dequeue
        do not write failed / UNVERIFIED / COMPLETED on remaining re
        return
```

Remaining units stay `open` / queued with the same `re`. New enqueue still hits the bound `N` (`TASK-25`): `queue_full`, not a side channel.

One latch per episode (`TASK-08`). Repeat ticks stay quiet after the log line below. A second dispatcher must see the same row (`TASK-16` — one lead on `pool_id`).

## Reset condition

Clear the episode only after a **completed** look that the offer would now succeed, plus a wrapper write. Not a timeout. Not deleting the row (`CRASH-LOOP-BRAKE.md`).

```
may_reset  iff  look(pool) completed
                AND n_unknown == 0
                AND free >= 1
                AND at least one worker same_process
                    (kind=dead: a **new** incarnation, start ≠ the dead set)
                AND (kind=quota: unexpired exemption_grant on the pool
                     — 402 does not self-reset)
                AND would_deny recomputed false
```

Then: append `type=pool_latch_reset` `{pool_id, free, n_remaining, pid, start}`. Flush. **Then** one offer of the **head** only (`ORDERED-RELEASE-QUEUE.md` — do not spray the bag). If that offer sees `free==0` again → new latch immediately (do not require a rebuild of three failures).

Window expiry of the latch without `may_reset` is not a reset. `n_remaining` dropping because of wrap is not a reset.

`canary-hit`: a declared free slot the enumerator must see. Missing → UNKNOWN, stay latched.  
`canary-miss`: plant `free==0` with `n_remaining≥1`. If any remaining `re` gets a fail disposition, `D` is broken.

## The log line

One object on stdout when the latch **sets**. Nothing on healthy offers. Nothing on repeat ticks of the same fingerprint. Reset is a **second** line, different `type`.

```
{"v":1,"type":"pool_latch","kind":"busy","pool_id":"shell-k4","K":4,"free":0,"busy":4,"n_remaining":47,"action":"stop_dispatch"}
```

| Field | Why |
|---|---|
| `kind` | busy ≠ dead ≠ quota (recovery differs) |
| `n_remaining` | bag still queued; not “47 failures” |
| `action=stop_dispatch` | operators must not read this as unit fail |
| no per-`re` fail lines | that is the storm this latch exists to prevent |

Exit 0 on latch is allowed (dispatcher **did** the right stop). Do not exit 0 with empty stdout — that looks like “no work / all done” (`POSITIVE-CONTROL.md`, `ABSENT-OUTPUT-STATE.md`). The line **is** the speak. A watchdog that pages on any stdout will page once per episode; that is intended.

Reset line:

```
{"v":1,"type":"pool_latch_reset","pool_id":"shell-k4","free":1,"n_remaining":47,"action":"offer_head"}
```

## What this must not do

- Walk the remaining bag and write `failed` / `UNVERIFIED` / `status=completed`.
- Latch on UNKNOWN.
- Reset on sleep, empty queue, or deleting `pool_latch`.
- Auto-reset `quota` without a grant.
- Offer the whole bag on reset.
- Omit `n_remaining` so the line looks like a generic error.

**Rule:** First completed exhaustion latches: stop offering, remainders stay queued. Reset only when `free≥1` (and a grant if quota), then offer the head. One log line: `pool_latch` with `kind`, `n_remaining`, `action=stop_dispatch` — not 47 unit failures.
