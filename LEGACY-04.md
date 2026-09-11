# LEGACY-04

Five ways a **scheduled job can stop doing work** while the dashboard still says **success**. Success here is a green last-run bit, `exit 0`, or `open_units = 0` — not a third-party check of the declared artifact (`TASK-05-DOCTRINE-DRAFT.md`).

## 1. Failed look reported as empty

The scheduler’s “any due work?” query times out or errors and returns `[]` (`LEGACY-03.md`, `TASK-20-FAIL-LOUD.md`). Zero due jobs → exit 0 → “success.” The jobs are still due.

**Observable:** `last_successful_scan` (UTC) older than the schedule interval while `exit_code == 0`. A `SearchFailed` / UNKNOWN counter > 0. Re-run the same query; if it raises, the green run was a failed look.

## 2. Hot table wrap hid the backlog

Due rows aged out of the N-row table (`TASK-22-ROLLING-WINDOW-AUDIT.md`). The count of due work is 0. The work was evicted, not done.

**Observable:** `wraps_since_boot > 0` and `archive_due_units > 0` (or month file has the handoff) while hot `due_units == 0`. `oldest_row_age_s` much smaller than the incident age.

## 3. Local/UTC misfile: “today” has no rows

Job scheduled 23:30 local is stored under the **next UTC date** (`TASK-19-TIMESTAMP-HAZARD.md`). The daily success check lists `YYYYMMDD` from the other clock and finds nothing due. Scheduler exits 0.

**Observable:** declared `scheduled_at` as UTC-aware vs the partition key. Same instant: local date ≠ UTC date. Receipt/handoff names disagree on `YYYYMMDD` while `re` matches (`TASK-07-DATE-ROLLOVER.md`).

## 4. Completion written before durable work

The job writes `status=success` / sends “done,” then dies before fsync (`TASK-31-GRACEFUL-SHUTDOWN.md`). Supervisor records last run OK. The declared path is missing, empty, or stale (`TASK-24-ARTIFACT-DECLARATION.md`).

**Observable:** worker record says completed; `verify(path, declared_digest)` is UNVERIFIED. File mtime/generation vs receipt `sha256` of canonical body do not match (`TASK-15-IDEMPOTENT-RECEIPT.md`).

## 5. Watchdog silence mistaken for health

Alerts = 0 because the episode fingerprint already fired, the probe timed out and was mapped to healthy, or the open log was rotated away (`TASK-08-WATCHDOG-PATTERN.md`, `TASK-30-LOG-ROTATION.md`, `TASK-34-METRIC-THAT-LIES.md`). Cron still “succeeded.”

**Observable:** `last_probe_outcome == UNKNOWN` or `unhealthy` while `watchdog_alerts == 0`; `last_known_good_age_s` > probe interval; archive `app.log.N` contains the first ANOMALY line the scrape missed.

Zero plus a **completed instrument** can be real success. Zero plus a dead, wrapped, or silent instrument is not.
