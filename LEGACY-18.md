# LEGACY-18

**Polling** re-measures durable state on a clock. **Event-driven** status waits for the job (or a broker) to say the world changed. For a long-running unit the question is not “which is cheaper.” It is whether the **gate** can be a third-party view (`LEGACY-17.md`, `TASK-05-DOCTRINE-DRAFT.md`) after the writer dies, lies, or goes silent.

## Comparison

| | Poll | Event |
|---|---|---|
| What is read | State that already exists: published lock, OS `pid+start`, declared path+digest | A message the job or bus **authored** (`status=running`, “done”, webhook) |
| Who can re-run it | Any observer (`LEGACY-13.md` excludes self) | Only if the message is still on the bus **and** you trust the author |
| Silence | Next tick still looks. Failed look → UNKNOWN, not healthy (`TASK-08-WATCHDOG-PATTERN.md`, `LEGACY-03.md`) | No message. Easy to treat as “still running” or as “nothing due” (`LEGACY-04.md`) |
| Crash / reboot | Cold-start admission: sweep `claimed`/`running` whose `start < boot_at` (`TASK-03-REBOOT-SWEEP.md`, `TASK-05-DOCTRINE-DRAFT.md`) | In-flight callbacks die with the process. Resume memory is not a claim |
| Latency | Bounded by the interval (worst-case detection is one period) | Fast when the message arrives; **unbounded** when it does not |
| Duplicate / reorder | Idempotent view: `verify` and alive are functions of current bytes (`TASK-15-IDEMPOTENT-RECEIPT.md`) | At-least-once “done” plus a later crash looks COMPLETED (`TASK-31-GRACEFUL-SHUTDOWN.md`) |
| Split-brain | One published lock; poll the OS (`LEGACY-08.md`, `LEGACY-12.md`) | Two seats can both emit “I am lead” |

An event is a **hint**. It may wake a poller early. It must not be the predicate that closes the unit or declares the holder alive. That would be the gate arbitrating the command that satisfies it (`LEGACY-17.md`) and a self-report (`LEGACY-06.md`).

## When polling is correct

Poll when the answer must survive a **non-cooperative** seat and a **lost channel**:

1. **Liveness of a long-running writer.** Alive is OS `pid` + `start` + `boot_at` (`LEGACY-08.md`, `LEGACY-14.md`). The kernel does not push. Heartbeat text is a sentence, not a probe.
2. **Completion of a declared artifact.** Status is `verify(path, digest)` over bytes already flushed (`TASK-17-SEPARATION-OF-DUTIES.md`, `TASK-24-ARTIFACT-DECLARATION.md`). A “done” event is the worker writing the verdict.
3. **Observer start, resume, or host reboot.** There is no trusted subscription backlog. Restart is a fresh admission (`TASK-05-DOCTRINE-DRAFT.md`). The first act is a completed scan of published claims, not “wait for the next callback.”
4. **Silence must not mean healthy.** Watchdog, drain, and “any due work?” queries fail closed on timeout (`TASK-08-WATCHDOG-PATTERN.md`, `TASK-31-GRACEFUL-SHUTDOWN.md`). Poll + UNKNOWN is the only way “no news” stays distinct from success.
5. **Stale vs merely old vs never started.** Those labels need a **completed read now** of the lock and the OS (`LEGACY-14.md`). Last callback age is a clock, not an incarnation.
6. **You need a bound on how long a lie can live.** The poll period is the maximum time a dead holder or a missing digest can look live. Event-only designs have no such bound.

Do not poll as a way to **mint** the thing you then accept (`LEGACY-17.md`): a tick that creates an empty file, inserts a dummy receipt, or writes `seat.ceiling = provider.class` is not a status check.

**Rule:** Events notify. Polls decide. For long-running jobs, polling is correct whenever the predicate is over state the job is not allowed to author — lock identity, process incarnation, and declared bytes.
