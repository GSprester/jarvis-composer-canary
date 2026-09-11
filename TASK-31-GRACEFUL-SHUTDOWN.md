# TASK-31-GRACEFUL-SHUTDOWN

Design only. This checkout has no worker supervisor. Shutdown must **not abandon in-flight work** as if it had never started, and must **not claim COMPLETED** before the artifact is durable (`TASK-11-IDEMPOTENCE.md`, `TASK-24-ARTIFACT-DECLARATION.md`).

## Sequence

On SIGTERM / seat revoke / queue drain (`TASK-25-QUEUE-BACKPRESSURE.md`):

1. **Stop admitting.** `enqueue` returns `queue_full` / refuse. No new handoffs. The one-lead lock is not released yet (`TASK-16-ONE-LEAD-LOCK.md`).
2. **Drain.** In-flight shells keep running. The worker waits up to **drain deadline T = 30 s** (stated estate) for each in-flight unit to reach **durable** state: bytes fsynced, then HWM / rename, then caller `verify` (`TASK-24-ARTIFACT-DECLARATION.md`).
3. **Then** write a receipt / disposition. Receipt is idempotent (`TASK-15-IDEMPOTENT-RECEIPT.md`). `COMPLETED` is only the verify view, not a worker bit (`TASK-17-SEPARATION-OF-DUTIES.md`).

```
stop_admit → wait_durable(T) → verify → receipt/disposition → release lock → exit
```

Do not: write `status=completed`, then flush, then exit.

## Drain deadline exceeded

If `verify` has not returned COMPLETED when T elapses:

| Action | Value |
|---|---|
| Kill the shell | Yes, after T — it is no longer “graceful” for that unit |
| Job status | **UNVERIFIED** (or `reboot_interrupted` if the process start is about to die, `TASK-03-REBOOT-SWEEP.md`) |
| Wrapper record | Disposition `drain_deadline_exceeded` — wrapper writes it, not the model (`TASK-05-DOCTRINE-DRAFT.md`) |
| Watchdog | UNKNOWN, not healthy (`TASK-21-WATCHDOG-SILENCE.md`) |
| Lock | Release only after identity is still probeable or the record is rewritten stale; do not leave an empty lock |
| Retry | Same handoff id; **read** first (`TASK-27-RETRY-SEMANTICS.md`). Do not send a second unit |

Exceeded deadline is **abandon of the process**, not abandon of the **unit**. The unit stays open/UNVERIFIED so a later seat can finish or a third party can see there is no matching digest.

If the work **did** become durable in the race after kill, `verify` on retry finds COMPLETED and must not run the shell again.

## Why completion-before-durable inverts the guarantee

The guarantee is: **no completion claim without an artifact a third party can check** (`TASK-05-DOCTRINE-DRAFT.md`, `TASK-10-SELF-REPORT.md`).

Inverted order:

```
t0  write status=completed / send "done"
t1  crash or SIGKILL
t2  bytes never fsynced; HWM not advanced; path empty or stale
```

A later checker sees missing/empty/stale (`TASK-24-ARTIFACT-DECLARATION.md`) → UNVERIFIED, but the store or a downstream queue already treated t0 as success. That is over-claim. Restart may skip the unit (thinks it is done) or retry send and **duplicate** (`TASK-27-RETRY-SEMANTICS.md`) if a peer also saw t0.

Correct order is the log rule: **append bytes → flush → then mark** (`TASK-11-IDEMPOTENCE.md`). The “mark” here is the caller’s verify + receipt, not a worker status field.

```
t0  write payload
t1  fsync / rename
t2  verify(path, declared_digest) == COMPLETED
t3  receipt for re=handoff_id
t4  unlock, exit
```

If death is before t2, there is no completion record to lie. If death is after t3, retry is a no-op (same receipt bytes).

## Rule

**Drain up to T; then disposition, never COMPLETED. Never write a completion record until verify succeeds on durable bytes. Deadline exceeded leaves the unit UNVERIFIED and the same id retryable.**
