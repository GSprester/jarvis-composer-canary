# TASK-19-TIMESTAMP-HAZARD

This checkout has no job store. The hazard is mixing **local** and **UTC** instants in the same rows, names, or partitions. A calendar date taken from one domain is not the date of the other. The same class of miss as `TASK-07-DATE-ROLLOVER.md`.

## What goes wrong

A naive `datetime` has no zone. Code that formats it with `UTC` in one path and `local` in another assigns two different `YYYYMMDD` keys to one job. Comparators (`scheduled_at < boot_at`, “jobs for today”, lock paths, receipt names) then disagree. `mtime` has the same weakness (`TASK-16-ONE-LEAD-LOCK.md`): it is a local-looking clock, not an incarnation.

## Worked example — 23:30 local misfiled to the next UTC day

Seat zone: `America/Los_Angeles` (PDT, UTC−7) on **2026-09-10**.

| | Value |
|---|---|
| Operator intent | run at **2026-09-10 23:30** local |
| Instant | 2026-09-10 23:30 −07:00 |
| Same instant in UTC | **2026-09-11 06:30:00Z** |

```python
# Writer A (local date in the job name / partition)
local = "2026-09-10T23:30:00"          # naive, implied PDT
name_a = "20260910-nightly"            # date from local clock

# Writer B (UTC date in the store key)
utc = "2026-09-11T06:30:00Z"
name_b = "20260911-nightly"            # date from UTC clock
```

The job scheduled at 23:30 **local on the 10th** is stored or listed under **20260911**. Effects:

- A “10 September local” queue does not contain it; it looks unscheduled.
- A UTC-day sweep for `20260911` claims it as tomorrow’s work while the operator still thinks it is tonight.
- A receipt minted after 00:00 UTC is named `20260911-*` and a prefix matcher drops the `20260910-*` handoff (`TASK-07-DATE-ROLLOVER.md`).
- `process_started_at` recorded as naive local 23:30 compared to `boot_at` in UTC can look **in the future** or **already before boot**, so reboot sweep (`TASK-03-REBOOT-SWEEP.md`) either skips a dead row or interrupts a live one.

No field is “wrong” in isolation. The **mix** is the bug.

```
2026-09-10 23:30 -07:00  ==  2026-09-11 06:30Z
local YYYYMMDD = 20260910
UTC   YYYYMMDD = 20260911   ← misfile
```

## Single rule that removes the class

**Store every instant as UTC, timezone-aware. Local civil time is display only. Never use a calendar date as a store key.**

Identity is `handoff_id` / `receipt.re` / `pid+start` in UTC (`TASK-04-MATCHER-TESTS.md`, `TASK-15-IDEMPOTENT-RECEIPT.md`, `TASK-16-ONE-LEAD-LOCK.md`). Partitions, lock paths, and receipt bodies do not contain `YYYYMMDD` derived from `now()` in either zone. If a UI needs “tonight at 23:30,” convert **from** the stored UTC instant **at the edge**.

That one rule ends local-vs-UTC double dating. Incremental “fix the matcher for +1 day” does not.
