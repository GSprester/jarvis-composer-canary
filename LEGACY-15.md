# LEGACY-15

**Retire** means the superseded tool is **not on any run path**: no import, no cron, no failover hop. **Document** means a note exists (this file, a comment, a gold-set case) while the binary can still run. They are not substitutes. A documented footgun is still a footgun.

Argue for **retire the callable path**. Keep a **tombstone** that names the successor — that is documentation *of the retirement*, not a third scheduler.

If you only document, two predicates stay live. The date-prefix matcher and `receipt.re == handoff.name` (`TASK-07-DATE-ROLLOVER.md`) can both run. One seat follows the README; another imports the old module because it is still `pip`-able. You get split-brain: one “unmatched,” one “matched,” both deterministic (`LEGACY-10.md`). Same for `exists(path)` vs `verify(path, digest)` (`LEGACY-05.md`) and worker `status=completed` vs the checker view (`TASK-17-SEPARATION-OF-DUTIES.md`). Documentation does not take the one-writer lock (`LEGACY-01.md`). The old tool will be the change-detector that skips the new job (`LEGACY-09.md`) or the search that returns `[]` on failure (`LEGACY-03.md`).

If you only retire, with no successor named, the next seat reimplements the prefix test or the empty-is-success helper. SUPERSEDED without a `successor_id` is not superseded in this estate. The tombstone must say: **removed from run**; **use this**; **do not do X**. Archive the old bytes (`TASK-22-ROLLING-WINDOW-AUDIT.md`); do not leave a green cron.

Retire: delete the hook, fail closed if something still calls it. Document: one short pointer, not a parallel tool. That pair is the argument. Documentation alone is fail-open: the old success bit still writes.
