# LEGACY-12

Five ways two processes **disagree about who is lead**, and how to resolve each. Leader means the published lock identity (`LEGACY-08.md`), not a status string (`LEGACY-06.md`).

## 1. Empty-lock identity gap

A created `writer-<unit>` with no `pid`/`start`. Process A was killed after `O_EXCL`. Process B sees “held” and no incarnation. A (if still dying slowly) later writes identity. Each believes it is lead (`LEGACY-01.md`).

**Resolve:** Never publish an empty file. fsync-then-rename a complete record. Missing fields ⇒ not alive (fail closed). Compare-and-unlink only that digest, then acquire.

## 2. mtime steal vs live blocked holder

B treats A’s lock as stale because mtime is old (A is in a 30s drain, `TASK-31-GRACEFUL-SHUTDOWN.md`). B unlinks and publishes. A still has the tree open. Two leads.

**Resolve:** Alive = `pid` live **and** OS `start(pid) == record.start` **and** `start >= boot_at`. Do not use mtime (`TASK-16-ONE-LEAD-LOCK.md`).

## 3. Pid reuse without start time

A died; the kernel reused `pid`. B reads lock `{pid: 4412}` with no `start`, sees 4412 live, waits forever **or** A’s successor is a different binary that thinks the lock is its own. Disagreement: “4412 is me” vs “4412 is the old writer.”

**Resolve:** Require `start`. Mismatch ⇒ stale. Same as reboot: `start < boot_at` ⇒ not alive (`TASK-03-REBOOT-SWEEP.md`).

## 4. Two clocks / two roots

A and B each have a lock path that **string-equals** differently: `/` vs `\`, `P0-…` vs `p0-…` (`LEGACY-02.md`), or local `YYYYMMDD` vs UTC (`LEGACY-11.md`, `TASK-19-TIMESTAMP-HAZARD.md`). Each published `writer-<what-it-thinks-is-unit>`. Both run.

**Resolve:** One unit id in the **body**, lock path `.lock/writer-<unit>` only. Lexical validate then join (`TASK-32-PATH-SAFETY.md`). Date and priority are not the lock key.

## 5. Side-channel / two stores

A holds the lock in clone 1. B “just runs the shell” or writes in clone 2 because the queue was full (`TASK-25-QUEUE-BACKPRESSURE.md`) or the lock file was not on that volume. Each store says it has the only writer.

**Resolve:** Work that did not acquire **this** repo’s published lock is refuse/`queue_full`, not a second lead. One working tree (`LEGACY-01.md`). Verify is on this root’s declared path (`TASK-24-ARTIFACT-DECLARATION.md`).

Timeout or unreadable lock is UNKNOWN, not “I am lead” (`LEGACY-03.md`, `LEGACY-07.md`).
