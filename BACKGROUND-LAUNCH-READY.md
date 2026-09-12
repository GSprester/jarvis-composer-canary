# Background launch: never success if nothing started

**Status:** design only.  
**Depends on:** `PROCESS-BOUNDARY-EXIT.md`, `LIVENESS-PREDICATE.md`, `TASK-16-ONE-LEAD-LOCK.md`, `CROSS-PLATFORM-LAUNCHER.md`.  
**Does not exist in this tree:** daemonizer, systemd unit, ready probe.

A **background** launcher `L` detaches a worker (double-fork, `CreateProcess` + parent return, `nohup`, systemd `Type=forking`). `L`’s **exit 0** is “`L` finished its spawn sequence.” It is **not** “a worker is running.” If `L` exits 0 after starting **nothing**, cron and `last-run` record success (`LEGACY-04.md`, `PROCESS-BOUNDARY-EXIT.md`).

Ready is a **third-party MEASURE** of the worker incarnation. It must not be `L`’s status bit, stdout “started,” or a pid file `L` wrote before `exec`.

## What “started nothing” is

| Sequence | `L` exit 0? | Worker |
|---|---|---|
| `fork`/`CreateProcess` fails; `L` still returns 0 | yes (bug) | none |
| Parent exits after first fork; child `execve` fails | often yes | none |
| `nohup` / `start /B` returns; image not resolved | yes | none |
| Lock `.tmp` written; rename never happens | yes | no published claim (`TASK-16`) |
| Pool exhausted; `L` “queued” as success | yes | none (`DISPATCH-POOL-LATCH.md`) |
| Pid file with a recycled pid | yes | stranger (`LIVENESS-PREDICATE.md`) |

`L` exit 0 is **orthogonal** to all of these. Supervisors that treat 0 as ready convert “started nothing” into success.

## Readiness proof (independent of `L`’s exit)

Declared before spawn (`TASK-24`): `unit`, image digest, lock path. After `L` returns (any exit), a **different** process `K` (wrapper/checker, not `L`) looks:

```
ready(unit)  iff  look(lock) completed
                  AND lock is the renamed record {pid, start, unit, image_digest}
                  AND look(same_process(pid, start)) completed AND same == true
                  AND start >= boot_at
                  AND image_digest == declared
                  AND (optional) first heartbeat {pid, start} flushed
```

| Signal | Not ready |
|---|---|
| `L` exit 0 | `L` is not the worker |
| `exists(pidfile)` / `exists(lock.tmp)` | unpublished / no `start` |
| pid exists, `start` mismatch | recycle |
| `same` UNKNOWN (EACCES, timeout) | UNKNOWN, not ready (`LIVENESS-PREDICATE.md`) |
| Worker `status=started` | self-report (`TASK-10`) |
| Empty stdout from `L` | not a type (`POSITIVE-CONTROL.md`) |
| systemd `ActiveState=activating` without the lock | supervisor hope |

`ready` is **inferred** at best until `same` is deterministic; it is **never** COMPLETED (`EVIDENCE-TIERING.md`). COMPLETED is still `verify` of work. Ready only means an incarnation holds the lock.

`K` must not be `L` writing the lock and then scoring it (`LEGACY-17.md`). The worker publishes the lock **after** `exec` (`TASK-16`: tmp → fsync → rename). `L` may wait **up to `T_ready`** for `ready` before it chooses **its** exit, but the **proof** is still `K`’s look, not that wait.

```
L_exit =
    2  if spawn failed OR T_ready elapsed AND NOT ready
       OR ready look UNKNOWN
    0  only as “L stopped waiting” — supervisors MUST NOT treat this as ready
```

The **estate** treats the job as started iff `ready(unit)`. `L_exit==0 ∧ NOT ready` is **started nothing** — page (`SILENT-WORKER-STOP.md` / `PROCESS-BOUNDARY-EXIT.md` parent 2 if `K` is the cron child). Prefer: `L` itself exits **2** when not ready, so even a naive supervisor does not record success. The proof remains `ready`, so a buggy `L` that exits 0 is still caught by `K`.

Foreground launch (`CROSS-PLATFORM-LAUNCHER.md` without detach): parent is the worker; `ready` is `same` of that pid after exec, or the parent has not yet reaped (`ABSENT-OUTPUT-STATE.md` running). Do not use this doc’s “`L` exit 0” as ready there either — that parent’s 0 means **finished**.

## Test

| Plant | `L` may exit | `ready` | Estate |
|---|---|---|---|
| `canary-no-exec` — image missing / `exec` fails | 2 (if `L` is honest) | false | **not started**; if `L` exits 0, `K` still false — supervisor that ignores `K` fails the test |
| `canary-live` — worker publishes lock, `same` | 0 or 2 | true | started |
| `canary-pidfile-only` — `L` writes pid, no rename | 2 | false | not started |

```
fail  iff  ready(canary-no-exec) == true
           OR (L_exit(canary-no-exec)==0 AND K skipped)
           OR estate.started == (L_exit==0)
```

Zero-acceptance: one tick that sets `started` from `L_exit==0` without `ready` fails the stratum. `K` must not insert the lock it then accepts.

## What this must not do

- Treat `L` exit 0, `nohup` 0, or `ActiveState=active` as ready.
- Write the lock in `L` before `exec`.
- Use pid-only or mtime.
- Call ready COMPLETED.

**Rule:** Started iff a checker sees a published `{pid,start}` whose `same_process` holds. `L`’s exit is not that look. Exit 0 after starting nothing is success only if the estate ignored `ready`.
