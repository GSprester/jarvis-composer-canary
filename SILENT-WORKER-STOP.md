# Silent worker stop

**Status:** design only.  
**Depends on:** `LIVENESS-PREDICATE.md`, `TASK-08-WATCHDOG-PATTERN.md`, `SILENT-FAILURE-DETECTION.md`, `TASK-10-SELF-REPORT.md`, `POSITIVE-CONTROL.md`.  
**Does not exist in this tree:** worker pool, heartbeat writer, detector cron.

An automated worker **stops without saying so** when it is still **claimed running** and there is **no wrapper terminal disposition**. Exit 0, `status=completed`, a touched file, and `errors==0` are not a goodbye (`LEGACY-04.md`, `TASK-34-METRIC-THAT-LIES.md`).

Four stops share that shape and need different looks. One heartbeat plus a detector that can be proved to **fire**.

## Claim vs goodbye

```
claimed     =  published lock {pid, start, unit}  OR  last wrapper heartbeat for that incarnation
goodbye     =  wrapper row: verify COMPLETED | failed_both | reboot_interrupted
               | quota_exhausted | quarantined | unverifiable
said        =  goodbye flushed, then HWM (TASK-11)
stopped     =  claimed AND NOT said
```

The worker must not write `goodbye` (`TASK-17-SEPARATION-OF-DUTIES.md`). A worker `status=completed` is **not** `said`.

## Heartbeat

Wrapper (not the worker) appends, every interval `T`, while `same_process` is true:

```
{type:heartbeat, worker_id, unit, pid, start, hwm, at_utc, policy_sha256}
```

| Rule | Hold |
|---|---|
| Author | Wrapper `pid+start` ≠ worker. Worker `touch` / `last_ok` is discarded (`TASK-08`: LKG not on fail). |
| Identity | Same `pid+start` as the lock. A new incarnation is a new series. |
| Progress | `hwm` is the ledger byte offset or last `re` the wrapper admitted, not mtime. |
| Flush | Bytes then HWM. Torn last line is not a beat (`IDEMPOTENT-EVENT-LEDGER.md`). |
| Silence | Missing beat after `T + ε` is a **detector input**, not health. |

`ε` is one clock tick plus scheduler slack, UTC (`TASK-19`). Do not key freshness on file mtime (`LEGACY-14.md`).

A beat proves only: the **wrapper** still looked at that incarnation at `at_utc`. It does not prove COMPLETED. It does not prove quota. It does not bless a worker success bit.

## Four stops

Detector `D` is a different process from the worker and from the heartbeat writer (`LEGACY-13.md`). Each kind requires a **completed** look. Failed look → UNKNOWN, not a stop, not healthy (`TASK-21-WATCHDOG-SILENCE.md`).

### Crash

Process incarnation is gone. No goodbye.

```
crash  iff  claimed
            AND look(same_process) completed AND same == false
            AND NOT said
            AND start < boot_at  OR  pid dead  OR  start_os ≠ record.start
```

Steal/recovery is `STALE-CLAIM-RECOVERY.md`. `D` pages `silent_stop/crash`. Do not wait for mtime.

### Stall

Incarnation is live. Work or beats have stopped.

```
stall  iff  claimed
            AND look(same_process) completed AND same == true
            AND NOT said
            AND ( last_heartbeat.at_utc < now_utc - (T+ε)
                  OR (claimed.unit set AND hwm unchanged for n·T
                      AND canary-hit work still due) )
```

Live pid + stale beat is the hung shell. Live pid + advancing beat + never-moving `hwm` while due work exists is a busy-loop that does no work (`ADVERSARIAL-ONE-PER-TICK.md`). `n` is declared (e.g. 3). Timeout on `/proc` is UNKNOWN — not stall, not crash (`LIVENESS-PREDICATE.md`).

### Quota exhaustion

Entitlement refused. Worker may still be alive and may write success.

```
quota_stop  iff  claimed
                 AND look(quota MEASURE) completed
                 AND class ∈ {402, entitlement_exhausted}   # not 429-as-402
                 AND NOT wrapper goodbye quota_exhausted
```

`D` pages `silent_stop/quota`. The worker heartbeat can still be fresh. **Do not** treat 402 as retry-until-200 (`ADVERSARIAL-QUOTA-RETRY.md`). COMPLETED remains A∧B only. A beat during 402 is not `said`.

### Silent success-reporting

The worker announced done. The checker did not.

```
fake_success  iff  claimed or worker record status=completed or exit 0
                   AND look(verify(path, expected)) completed
                   AND verify != COMPLETED
                   AND NOT said
```

This is LEGACY-04 §4 / `TASK-10`. Heartbeat freshness **does not** clear it. `D` pages `silent_stop/fake_success`. Mapping verify-fail to healthy because the worker exited 0 is the bug.

## Detector output

`D` is a silent watchdog (`TASK-08`): speak only on a new fingerprint `{kind, worker_id, pid, start, unit}`. Healthy is allowed **only** when:

```
D_healthy  iff  every claimed incarnation has a beat within T+ε
                AND same_process consistent with that beat
                AND no fake_success
                AND no unsaid 402
                AND both controls below completed
```

Zero ERROR rows and an empty due list are not `D_healthy` (`SILENT-FAILURE-DETECTION.md`).

## Positive control (proves `D` can fire)

A detector that is always silent is indistinguishable from a working quiet estate (`ACCEPTANCE-RATE.md` A→1). Every `D` interval must complete a **planted stop** the issuer put on the **same** path `D` watches (`LEGACY-39.md`).

```
canary-stop  =  a declared lock + heartbeat series whose correct D verdict is
                silent_stop  (one of: dead pid+start, beat older than T+ε,
                402 without goodbye, status=completed ∧ verify UNVERIFIED)
```

| Look | Hold |
|---|---|
| `D(canary-stop)` speaks `silent_stop` of the planted kind | detector **can fire** |
| `D(canary-stop)` silent or healthy | **instrument dead** — do not trust `D_healthy` on prod workers |
| look of the plant UNKNOWN | UNKNOWN, not “no stops” |
| `D` inserts the plant or an ERROR so it can count a fire | gate mints evidence (`LEGACY-17.md`) |

`canary-hit` for `D`: a declared live worker with current beat, `same==true`, no 402, no fake success → `D` silent. Both required every interval, same as NONE (`POSITIVE-CONTROL.md`).

Do not cache “we fired once in staging.” Resume is a new admit (`TASK-05`). Evicted plant → UNKNOWN, restore, then look (`TASK-09`).

Zero-acceptance: one interval where `canary-stop` does not fire fails the `D` stratum (`CLASSIFIER-ZERO-ACCEPT.md`).

## What this must not do

- Treat worker `touch` / exit 0 / `errors==0` as a heartbeat or a goodbye.
- Call timeout on `/proc` a crash.
- Page only on `errors>0`.
- Retry 402 until the worker reports success.
- Let `D` write the stop it then detects.

**Rule:** Wrapper heartbeats `{pid,start,hwm}` every `T`. `D` pages crash (dead incarnation, no goodbye), stall (live, beat or hwm dead), quota (402 unsaid), fake success (exit 0 / status vs UNVERIFIED). `D` is healthy only if a planted `canary-stop` still fires on the same path.
