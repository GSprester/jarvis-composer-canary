# TASK-09-RETENTION-RISK

This checkout has no event table. The analysis is for a fixed-size rolling store of the kind implied by `TASK-03-REBOOT-SWEEP.md` (bounded row cap, oldest-first eviction).

## Failure mode: aged out before anyone looks

The table keeps at most **N** rows. Insert of row N+1 deletes the oldest row. Audit does not run on insert; it runs when a human or a later seat opens an incident. If the lookback window is longer than the time to wrap the cap, the rows that would reconcile the incident are already gone. Absence is then indistinguishable from “never happened.”

Concrete estate (stated assumptions, not measurements from this repo):

| Parameter | Value |
|---|---|
| Cap **N** | 2,048 rows |
| Mean row size | 2 KiB (status, timestamps, fingerprint, path) |
| Hot-table size | 2,048 × 2 KiB = **4.0 MiB** |
| Write rate | 180 rows/day (claims, receipts, dispositions, watchdog ticks, reboot_interrupted rewrites) |
| Wrap time | 2,048 / 180 ≈ **11.4 days** |
| Typical incident review lag | **30 days** (next monthly pass, or “looked after the weekend” plus backlog) |
| Rows needed for one incident | ~40 (handoff, receipt, matcher pairs, watchdog episode, reboot sweep) spanning **3–14 days** before the page |

At day 30 the reviewer needs rows from day 0–14. Those rows were evicted around day 11–25. The hot table then holds only day 19–30. The incident’s `reboot_interrupted` row, the `re` identity of the receipt, and the first watchdog fingerprint are gone. A date-prefix matcher miss (`TASK-07-DATE-ROLLOVER.md`) cannot be proven; a silent-watchdog UNKNOWN episode (`TASK-08-WATCHDOG-PATTERN.md`) cannot be distinguished from “never probed.”

Peak-day variant: 600 rows/day (retry storms). Wrap time drops to 2,048 / 600 ≈ **3.4 days**. A Friday incident reviewed Monday has already aged out.

## Strategy A — raise the cap

Keep a single hot table. Size it for the audit horizon, not for wrap comfort.

Target: **90 days** at 180 rows/day = 16,200 rows. Round to **16,384**.

| | Today | Raised cap |
|---|---|---|
| N | 2,048 | 16,384 (8×) |
| Disk / backup of the table | 4.0 MiB | **32.0 MiB** |
| Wrap time at 180/day | 11.4 days | **91 days** |
| Wrap time at 600/day | 3.4 days | **27 days** (still short of 90) |
| Peak-safe N for 90 days at 600/day | — | 54,000 ≈ **65,536** → **128 MiB** |

Cost trade-off:

- **Pays:** one store, one query, no archive job. Incident review stays `SELECT` on the live table.
- **Costs:** 8×–32× memory and backup; every full scan (naive matcher, reboot sweep) walks 8×–32× rows; a crash-recovery rewrite (`reboot_interrupted`) still consumes cap and does not free space (`TASK-03-REBOOT-SWEEP.md`).
- **Does not pay** if write rate is bursty: 65,536 at 2 KiB is 128 MiB and still wraps in 90 days only if the 600/day peak is the mean. A week of 2,000 rows/day wraps 65,536 in **33 days**.

Raise-the-cap is the right move when the estate already owns tens of MiB and review is always ad hoc on the live box.

## Strategy B — archive to a monthly store

Leave the hot cap at **2,048** (4.0 MiB, wrap ~11 days). Once per UTC month, copy every row with `event_time` in that month to an append-only file `events-YYYYMM.jsonl` (or a monthly SQLite), then evict only those archived rows that are terminal.

Volume:

| Period | Rows at 180/day | Archive size at 2 KiB/row |
|---|---|---|
| One month (30 d) | 5,400 | **10.5 MiB** |
| One year | 65,700 | **128 MiB** |
| Three years | 197,100 | **385 MiB** |

A 14-day incident in September is in `events-202609.jsonl` even if the hot table wrapped three times since.

Cost trade-off:

- **Pays:** hot path stays 4.0 MiB and 2,048-row sweeps; audit horizon is years, not days; reboot_interrupted and watchdog fingerprints survive eviction.
- **Costs:** a monthly job that must succeed (failed archive + continued eviction **is** the original failure mode); reconcile is two lookups (hot, then `events-YYYYMM*`); UTC month boundaries need the same identity key as `receipt.re` (`TASK-07-DATE-ROLLOVER.md`) — do not name archive files from local date only; restore/grep of 10.5 MiB JSONL is cheap, but you now operate two stores.
- **Does not pay** if the archive job is “best effort.” If September’s copy is skipped, September’s wrap still deletes the incident before October’s review.

Minimum archive SLA: finish copy, fsync, verify row count ≥ month’s inserts, **then** allow eviction of those terminal rows. Incomplete archive must block eviction (fail closed).

## Which cost to take

| If | Prefer |
|---|---|
| Review lag ≤ 14 days and mean ≤ 180 rows/day | A at N = 16,384 (32 MiB) |
| Review lag ≥ 30 days, or peaks ≥ 600 rows/day | B; keep hot N = 2,048 and monthly 10.5 MiB files |
| Need both fast live sweep and 90-day proof | A at 16,384 **plus** B for months 2+ (hot is cache, monthly is source of truth) |

Do not treat “row missing” as “work never existed.” After wrap, missing is the default.
