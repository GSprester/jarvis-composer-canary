# LEGACY-31

An **unverified claim** is a sentence about a unit that a third party cannot re-derive from declared bytes (`TASK-05-DOCTRINE-DRAFT.md`). **Settled** means COMPLETED: `verify(path, declared.expected)` after a completed look (`TASK-17-SEPARATION-OF-DUTIES.md`). Reading the first as the second is not a labeling slip. It **stops the work that would have produced the check**.

## Scenario

Handoff `20260910-alpha` declares `out/bundle` and a digest. Drain hits SIGTERM. The worker writes `status=completed` and exits before fsync (`TASK-31-GRACEFUL-SHUTDOWN.md`). The path is empty or absent (`LEGACY-05.md`). No durable receipt is published. A later seat reads the status bit as settled. `find_due` returns `[]` — or times out and returns `[]` (`LEGACY-03.md`). Cron exits 0 (`LEGACY-04.md`). The change-detector hashes that green row and skips the job (`LEGACY-09.md`).

Cost compounds on the next clock. The unit’s name on disk is tomorrow’s UTC date (`TASK-19-TIMESTAMP-HAZARD.md`). A prefix matcher drops the pair (`TASK-07-DATE-ROLLOVER.md`). `open_units` is 0 (`TASK-34-METRIC-THAT-LIES.md`). Failover hops because “this seat finished,” widening class or rotating a key that was only `denied_here` (`LEGACY-23.md`). Retry mints a new id and double-sends (`TASK-27-RETRY-SEMANTICS.md`), or does not send because settled. The lock is released; a second lead writes another empty file (`LEGACY-01.md`). The hot row that named the obligation is evicted without archive (`TASK-22-ROLLING-WINDOW-AUDIT.md`).

The dashboard is **deterministic** and there is **no** artifact (`LEGACY-10.md`). Re-opening means treating COMPLETED as UNVERIFIED — against every green bit the store still emits.

**Rule:** Unverified read as settled retires the checker. The cost is the work that will not run, not the wrong adjective on a row.
