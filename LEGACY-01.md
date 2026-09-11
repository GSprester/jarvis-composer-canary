# LEGACY-01

**One writer per repository** means at most one seat may mutate the working tree, job rows, receipts, and lock files at a time. It protects against **split-brain writes**: two agents each believe they are lead, both `git add` / rewrite the same handoff, and the store can no longer tell which bytes are the artifact. A third party then sees conflicting receipts, a torn event log, or a `COMPLETED` view that does not match any file (`TASK-10-SELF-REPORT.md`). The invariant is not “be polite.” It is “one published identity holds the tree.”

It typically fails in three concrete ways.

## 1. Empty-lock identity gap

Acquire is `O_EXCL` create of an empty lock file; pid and process start are written **later**. A hard kill in the gap leaves a file that means “held” and names **no incarnation** (`TASK-16-ONE-LEAD-LOCK.md`). Waiters deadlock. Stealers take the lock while the first seat is still alive and then write. Two leads, one repo.

## 2. mtime-as-liveness steal

A recoverer treats an “old” lock mtime as dead and unlinks it. mtime is not a process (`TASK-16-ONE-LEAD-LOCK.md`). A live holder blocked in a drain (`TASK-31-GRACEFUL-SHUTDOWN.md`) looks stale; a dead holder can look fresh after `touch` or backup. The second seat starts writing while the first still has the tree open. Same split-brain, justified by a clock.

## 3. Side-channel enqueue

The queue is full (`TASK-25-QUEUE-BACKPRESSURE.md`) or failover hops providers (`TASK-13-FAILOVER-CEILING.md`). A seat “just runs the shell” or opens a second clone, skipping the lock. The lock file is honest; the **path around it** is not. Two checkouts, two `COMPLETED` records, one declared path. Content-hash dedupe (`TASK-26-DEDUPE-BY-CONTENT.md`) may collapse logs after the fact. It does not un-run the second writer.

Publish a complete `pid+start` record via fsync-then-rename, probe that incarnation (not mtime), and refuse work that did not take the lock. Timeout is UNKNOWN, not “no writer.”
