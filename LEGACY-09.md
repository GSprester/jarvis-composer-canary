# LEGACY-09

A **change-detector that suppresses a scheduled job** deadlocks when the **only writer** of the detector’s input is that job. The detector says “no change” (or “nothing due”) → cron exits 0 (`LEGACY-04.md`) → the job never updates the input → the detector never flips. Three concrete loops.

## 1. Existence skip vs stale artifact

Detector: `if output.exists(): skip`. The job’s effect is **replace** the file with declared bytes (`TASK-24-ARTIFACT-DECLARATION.md`). A leftover empty or previous-run file already **exists** (`LEGACY-05.md`). Skip forever. The digest never becomes the handoff’s `expected_sha256` because only the job would write it.

**Break:** suppress on `verify(path, declared) == COMPLETED`, not on exists. Missing/empty/stale must **run** the job.

## 2. Empty-due skip vs failed search

Detector: `if find_due() == []: skip`. `find_due` returns `[]` on timeout (`LEGACY-03.md`). The job’s effect is the query path working (store flush, receipt rows, lock identity). Those rows are never written because the job did not run. Next tick: timeout or empty hot table (`TASK-22-ROLLING-WINDOW-AUDIT.md`) → `[]` again → skip.

**Break:** `SearchFailed` / UNKNOWN **must run** or escalate; `[]` is skip only after a **completed** scan (`TASK-20-FAIL-LOUD.md`).

## 3. Self-hash / LKG skip vs the job that refreshes it

Detector: `if content_hash(state) == last_hash: skip` (or watchdog `last_alert_fp` / LKG unchanged, `TASK-08-WATCHDOG-PATTERN.md`). `state` includes a worker `status=success` or LKG the **job itself** wrote last time — including after a failed probe mapped to healthy (`TASK-34-METRIC-THAT-LIES.md`). The job’s effect is a new probe, new receipt, or a corrected status. Skip means LKG and `last_hash` never move. Estate drifts; detector still says “same.”

**Break:** hash only **inputs the job does not write** (declared digest, `re`, `pid+start`). LKG updates only after a successful **healthy** probe. Status is a view, not a detector input (`TASK-17-SEPARATION-OF-DUTIES.md`).

In all three, the detector is watching a **side effect of the job**, then using “no delta” to cancel the job. Point the detector at **obligations** (declared path+digest, completed search, incarnation), not at the job’s own leftovers.
