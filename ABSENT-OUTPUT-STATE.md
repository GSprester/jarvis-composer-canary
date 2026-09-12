# Absent output: running vs finished-quiet

**Status:** design only.  
**Depends on:** `POSITIVE-CONTROL.md`, `LIVENESS-PREDICATE.md`, `SILENT-WORKER-STOP.md`, `TASK-20-FAIL-LOUD.md`.  
**Does not exist in this tree:** supervisor, output tee, reap loop.

The only wire many watchers see is **no new bytes** on the job’s stdout/stderr/log. That observation is compatible with **still running** and with **finished and reported nothing**. It is also compatible with a failed look (`[]`, timeout, unopened fd). Absence of output is not a state. It is a missing MEASURE of output.

## Why the output fd cannot decide

| Estate | New output |
|---|---|
| Running, no line yet (compute, block on IO, quiet-healthy worker) | none |
| Exited 0, printed nothing (typed NONE never flushed; grep-miss; cron success) | none |
| Exited / crashed before flush | none |
| Supervisor has not opened the fd / read timed out | “none” |

`if not stdout: done` and `if not stdout: still going` are the same bug with opposite defaults (`POSITIVE-CONTROL.md`: empty is not NONE). A deadline on silence that flips running → finished **without a reap** invents a goodbye (`SILENT-WORKER-STOP.md`).

## Second channel (required)

Distinguish with a look that is **not** the output stream. Declare it on the handoff before start.

```
reap      =  waitpid / equivalent completed: exit_code known, pid reaped
same      =  pid_exists AND start_os == record.start AND start >= boot_at
beat      =  wrapper heartbeat {pid, start, hwm} within T+ε     # optional, not a reap
output    =  bytes on the declared fd/log after HWM, or empty after a completed read
```

`output` empty after a **failed** read is UNKNOWN, not empty (`TASK-20-FAIL-LOUD.md`). Do not treat “no poll event” as a completed empty read.

```
running          iff  reap is false
                      AND look(same) completed AND same == true

finished_quiet   iff  reap is true
                      AND look(output) completed AND output is empty
                      AND (typed NONE with both controls
                           OR declared contract is “silence allowed after exit”)

unknown          iff  look(same) or look(output) or wait did not complete
                      OR (reap is false AND same == false)  # crash; see silent_stop/crash
                      OR reap is false AND same UNKNOWN

not_this_job     iff  no lock / never started (REMOVAL-FIX never_present)
```

`finished_quiet` is **not** COMPLETED. COMPLETED is still `verify(path, expected)` or a typed FOUND (`TASK-17`). Quiet exit with an expected artifact is `fake_success` (`SILENT-WORKER-STOP.md`).

`running` does **not** require new output. A live incarnation with an empty fd is the normal quiet worker (`TASK-08` silence-as-health is the **watchdog**, not the job).

Do not use `beat` as a substitute for `reap`. A beat says the wrapper looked. A dead worker can leave a stale beat; a live worker can miss a beat (stall). `beat` refines stall vs healthy-running; it does not flip `finished_quiet`.

## Protocol

Wrapper owns the child (`TASK-16` lock `{pid,start}`):

1. **While `same`:** classify `running`. Record `output_hwm`. Absence of new bytes updates a gauge `silent_for_s`, not a verdict.
2. **On reap:** completed `wait`. Then completed read of the output object (or explicit empty). Then:
   - typed `{kind:NONE, control.hit, control.miss}` → `finished_quiet` / valid NONE
   - empty stdout + exit 0 + no type → `finished_quiet` / **untyped** (unhealthy unless the handoff declared silence)
   - empty + exit ≠ 0 → finished, failed; not running
3. **On `same == false` without reap:** `unknown` until wait, then `silent_stop/crash`. Do not call it `finished_quiet` (no completed empty-output look).
4. **Deadline `T_silent`:** if `silent_for_s > T_silent` **and** `running`, that is **stall** (still running), not finish (`SILENT-WORKER-STOP.md` stall). Page stall. Do not reap-by-fiction.

Never: `silent_for_s > T` ⇒ `finished_quiet`. That is the collapse.

## Positive controls (same path as the watcher)

Two plants, every interval (`LEGACY-39.md`). `D` is the classifier, not the job.

| Plant | Correct `D` |
|---|---|
| `canary-live-quiet` — live `{pid,start}`, no bytes, no reap | `running` |
| `canary-exit-quiet` — reaped, exit 0, empty fd, typed NONE + controls (or declared silence) | `finished_quiet` |

If `canary-live-quiet` is `finished_quiet`, the watcher uses silence as done.  
If `canary-exit-quiet` is `running`, the watcher uses silence as liveness.  
Either miss → do not publish prod classifications. Timeout on a plant → UNKNOWN (`ACCEPTANCE-RATE.md`).

`D` must not start or kill the plants so the verdict becomes true (`LEGACY-17.md`).

## What this must not do

- Infer finish from empty stdout, idle CPU, or stale mtime.
- Infer running from “we haven’t seen an exit line” without `same` / wait.
- Map read-timeout of the log to empty output.
- Call `finished_quiet` COMPLETED.
- Flip running → finished when a silence deadline elapses.

**Rule:** No new output does not distinguish the two. `running` iff not reaped and `same`; `finished_quiet` iff reaped and a completed empty-output look (typed NONE or declared silence). Silence past a deadline is stall, not a goodbye.
