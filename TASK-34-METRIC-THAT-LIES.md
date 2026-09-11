# TASK-34-METRIC-THAT-LIES

A count of **zero** is often treated as “the condition is gone.” That is the same shortcut as `len == 0` (`TASK-18-COUNT-RECONCILIATION.md`) and `[]` meaning success (`TASK-20-FAIL-LOUD.md`). This checkout has no metrics backend. The lying metric in this estate is **`open_units = 0`** (or `unmatched_handoffs = 0`, `watchdog_alerts = 0`): operators read “quiet” while work is still wrong.

Below, the underlying condition is **unfinished or unverified work**. Zero does not mean it is resolved.

## Three mechanisms

### 1. The rolling table evicted the rows

`open_units` is `COUNT(*)` on the hot job table. Cap N = 2,048, storm R = 10,000/day (`TASK-22-ROLLING-WINDOW-AUDIT.md`). After ~4.9 hours the OPEN rows that were the incident are gone. The count is **0**. The handoffs are still unmatched; the receipts still fail prefix match (`TASK-07-DATE-ROLLOVER.md`). Absence was stored as “never existed.”

**Companion:** `open_units` **and** `archive_open_units` (monthly store, archive-before-evict) **and** `wraps_since_boot`. If hot is 0 and `wraps_since_boot > 0` without a matching archive count, the zero is a wrap, not a close. `oldest_row_age_s` going **down** while work is ongoing is the same tell.

### 2. The search failed and returned empty

`unmatched_handoffs` is “handoffs whose `find_receipts` returned `[]`.” A timeout or missing table returns `[]` (`TASK-20-FAIL-LOUD.md`). The matcher never ran. The dashboard shows **0 unmatched** (or 0 errors) because the failed look was classified as “none.” The receipts are on disk.

**Companion:** `receipt_query_unknown` (raise path / UNKNOWN from `TASK-21-WATCHDOG-SILENCE.md`). Zero unmatched is trustworthy only when `receipt_query_unknown == 0` **and** the query actually completed. A gauge `last_successful_receipt_scan_unix` (UTC, `TASK-19-TIMESTAMP-HAZARD.md`) that is stale means the zero is silence, not health.

### 3. The watchdog stopped speaking for the wrong reason

`watchdog_alerts = 0` looks like healthy. Mechanisms that zero it **without** a healthy probe:

- Same fingerprint, episode already alerted (`TASK-08-WATCHDOG-PATTERN.md`) — still unhealthy, just quiet.
- Probe timeout/raise should be UNKNOWN; a bug maps that to healthy and **overwrites LKG**.
- Log rotator deleted the open alert file or wrapped the only copy (`TASK-30-LOG-ROTATION.md`) so the **scrape** of alerts is 0.

The estate is still on fire; the **counter of pages** is 0.

**Companion:** `last_known_good_age_s` and `last_probe_outcome{healthy,unhealthy,unknown}`. Alerts == 0 is allowed only if the last **successful** outcome is healthy **and** LKG age is within the probe interval. UNKNOWN or stale LKG with zero alerts is a lie. Scrape `rotated_alert_files` so prune cannot hide the episode.

## Summary

| Lying zero | Why it hit 0 | Companion that catches it |
|---|---|---|
| `open_units` | Hot rows evicted; work not closed | Archive count + wrap counter + oldest-row age |
| `unmatched_handoffs` | Failed search returned `[]` | `*_unknown` + last successful scan time |
| `watchdog_alerts` | One-shot fingerprint, false healthy, or rotated-away log | Last probe outcome + LKG age (not the alert counter) |

## Rule

**A zero is a measurement of the instrument, not of the estate.** Pair every “count of bad things” with “the instrument ran and still sees the universe that contained those things.” If the instrument did not run, the disposition is UNKNOWN, not 0 (`TASK-05-DOCTRINE-DRAFT.md`).
