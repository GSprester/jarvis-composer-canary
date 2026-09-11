# LEGACY-14

**Old** is a clock. **Stale** is an incarnation that **cannot be alive**. Prove stale with `pid` + `start` + `boot_at`, not with mtime (`LEGACY-08.md`, `TASK-16-ONE-LEAD-LOCK.md`). **Abandoned** means a holder existed and is gone. **Never started** means no holder was ever published.

## Merely old (do not steal)

The claim/lock is well-formed. `pid` is live. OS `start(pid) == record.start`. `start >= boot_at`. The seat may be silent in a drain (`TASK-31-GRACEFUL-SHUTDOWN.md`). mtime can be ancient. That is **old and live**. Observer excludes itself (`LEGACY-13.md`) and **waits**.

## Prove stale

Need a **completed** read of a published record (rename visible, JSON parses). Failure to read is UNKNOWN, not stale (`LEGACY-03.md`, `LEGACY-07.md`).

```
stale  iff  published record well-formed
       AND  (
              pid not live
              OR OS start(pid) ≠ record.start
              OR record.start < host boot_at
            )
```

Then compare-and-unlink **those bytes** and rewrite `reboot_interrupted` / disposition — wrapper, not the model (`TASK-03-REBOOT-SWEEP.md`, `TASK-05-DOCTRINE-DRAFT.md`).

Unparseable or empty lock is **not a live claim**. Treat as interrupted identity (fail closed), not as “old holder.” That is the empty-gap (`LEGACY-01.md`), not “merely old.”

## Abandoned vs never started

| | Abandoned | Never started |
|---|---|---|
| Published lock / claim row | Yes: `pid`, `start`, `unit` | No complete publish (no row, or empty file, or missing `start`) |
| Artifact | May be missing, empty, or stale leftover | Same — existence proves nothing (`LEGACY-05.md`) |
| Proof | Incarnation **was** named and **fails** the alive predicate | No incarnation to test; `find` completed and returned no record |
| Disposition | `abandoned` / `reboot_interrupted` (had a holder) | `never_started` / still OPEN (no holder) |
| Retry | Same `unit`; verify first (`TASK-27-RETRY-SEMANTICS.md`) | Same; do not send a second id |

A job that never acquired the lock is **never started**, even if a scheduler row is “old.” A job whose `pid` died after rename is **abandoned**, even if the file is one second old.

Do not use “no output file” to pick the label: both can lack bytes. Do not use “row age > T”: that is merely old if the pid still matches.

## Rule

**Stale = named incarnation that cannot exist. Abandoned = stale after a real publish. Never started = completed scan, no publish. Old + alive = leave it.**
