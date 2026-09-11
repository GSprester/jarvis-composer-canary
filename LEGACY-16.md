# LEGACY-16

Five **mechanical** guards so a seat cannot re-create a bypass it was told to stop using (`LEGACY-15.md`). A comment is not a guard. The old path must be **unrepresentable** or **unsuccessful**.

## 1. Tombstone module that raises

Replace the retired entrypoint with a module that **raises** on import or call (`SearchFailed` / `BypassRetired`). Do not leave a stub that returns `[]` or `True` (`LEGACY-03.md`, `LEGACY-07.md`). Cron that still names the old hook goes UNKNOWN, not green (`LEGACY-04.md`). Successor is the only callable name.

## 2. No write path to COMPLETED

The worker API cannot set `status=completed` (`TASK-17-SEPARATION-OF-DUTIES.md`). The view is `verify(declared_path, declared_digest)` only. Recreating `exists(path)` or a self-hash record does not close the unit (`LEGACY-05.md`, `LEGACY-06.md`). The bypass has nowhere to write a lie that downstream will treat as a check.

## 3. Lock-shaped admit

Any mutate (shell, receipt, rotate) requires a published `pid+start` lock on **this** root (`LEGACY-08.md`, `LEGACY-01.md`). Queue `offer` and failover `admit` refuse work that skipped the lock (`TASK-25-QUEUE-BACKPRESSURE.md`, `TASK-13-FAILOVER-CEILING.md`). A “just run the shell” clone is `queue_full` / `FailoverDenied`, not a second lead (`LEGACY-12.md`).

## 4. Lexical path + declared artifact only

Writes go through `validate_declared` then `declare` (`TASK-32-PATH-SAFETY.md`, `TASK-24-ARTIFACT-DECLARATION.md`). Absolute paths, `..`, reserved names, and undeclared outputs never join the root. Recreating `/tmp/done` or a date-prefix name as a side file cannot become the checker’s path.

## 5. CI predicate tests, not prose

Ship tests that **fail the build** if the old predicate is used: prefix match instead of `re` (`TASK-04-MATCHER-TESTS.md`), `len` shortcut (`TASK-18-COUNT-RECONCILIATION.md`), hash-before-normalise (`TASK-15-IDEMPOTENT-RECEIPT.md`), 429-as-402 (`TASK-28-ERROR-CLASSIFICATION.md`). Grep the tree for the retired symbol. A seat that pastes the bypass back loses CI. SUPERSEDED without `successor_id` is not merged (`LEGACY-15.md`).

Honor-system READMEs and “please don’t” comments are not in this list. If the seat can still `import old_matcher` and `UPDATE status` and exit 0, the bypass was not retired.
