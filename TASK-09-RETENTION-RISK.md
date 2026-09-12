# TASK-09-RETENTION-RISK

This checkout has no event table. The analysis is for a fixed-size rolling store of the kind implied by `TASK-03-REBOOT-SWEEP.md` (bounded row cap, oldest-first eviction). Numbers are a stated estate, not measurements here. Same N, R, and 2 KiB rows as `TASK-22-ROLLING-WINDOW-AUDIT.md`.

## Failure mode: aged out before anyone looks

The table keeps at most **N** rows. Insert of row N+1 deletes the oldest row. Audit does not run on insert; it runs when a human or a later seat opens an incident. If the lookback needed to reconcile that incident is longer than the time to wrap the cap, the rows are already gone. Absence is then indistinguishable from “never happened.”

```
unreconstructable  iff  S + L > N / R
```

**S** = span of days the incident needs (handoff, receipt, lock `pid+start`, watchdog UNKNOWN, `reboot_interrupted`). **L** = days until someone looks. **R** = mean rows per UTC day. The first insert that makes `rows_written_since_incident > N` is the point of no reconstruction.

Concrete estate:

| Parameter | Value |
|---|---|
| Cap **N** | 2,048 rows |
| Mean row size | 2 KiB (status, timestamps, fingerprint, path) |
| Hot-table size | 2,048 × 2 KiB = **4.0 MiB** |
| Write rate **R** | 180 rows/day (claims, receipts, dispositions, watchdog ticks, `reboot_interrupted` rewrites) |
| Wrap time **N/R** | 2,048 / 180 ≈ **11.4 days** |
| Typical review lag **L** | **30 days** (monthly pass, or “looked after the weekend” plus backlog) |
| Incident span **S** | ~40 rows over **3–14 days** before the page |

Worked look at L = 30, S = 14: the reviewer needs day 0–14. Those rows were evicted around day 11–25. The hot table then holds only day 19–30. The `reboot_interrupted` row, the receipt `re`, and the first watchdog fingerprint are gone. A date-prefix matcher miss (`TASK-07-DATE-ROLLOVER.md`) cannot be proven; a silent-watchdog UNKNOWN (`TASK-08-WATCHDOG-PATTERN.md`) cannot be distinguished from “never probed.” `open_units = 0` looks clean (`TASK-34-METRIC-THAT-LIES.md`).

| R (rows/day) | N/R | Weekend look (S=0.5, L=2) | Monthly look (S=3, L=30) |
|---:|---:|---|---|
| 180 | 11.4 d | reconstructable | **gone by day 11** (need N/R > 33 ⇒ N > 5,940) |
| 600 (retry storm) | 3.4 d | barely | gone; Friday incident reviewed Monday is already out |
| 10,000 (watchdog/retry loop) | 0.205 d (**4.9 h**) | **gone the same afternoon** | gone |

A bounded cap is not freed by rewriting live rows to `reboot_interrupted` (`TASK-03-REBOOT-SWEEP.md`). Those terminal rows still count toward N until eviction.

## Strategy A — raise the cap

Keep a single hot table. Size it for the audit horizon, not for wrap comfort.

Target: **90 days** at 180 rows/day = 16,200 rows. Round to **16,384**.

| | Today | Raised cap | Peak-safe (90 d at 600/day) |
|---|---:|---:|---:|
| N | 2,048 | 16,384 (8×) | 65,536 (32×; 54,000 rounded up) |
| Disk / backup | 4.0 MiB | **32.0 MiB** | **128 MiB** |
| Wrap at 180/day | 11.4 d | **91 d** | 364 d |
| Wrap at 600/day | 3.4 d | **27 d** (still short of 90) | **109 d** |
| Wrap at 10,000/day | 4.9 h | 1.6 d | 6.6 d |

Cost trade-off:

- **Pays:** one store, one query, no archive job. Incident review stays a read of the live table.
- **Costs:** 8×–32× memory and backup; every full scan (naive matcher, reboot sweep) walks 8×–32× rows; a crash-recovery rewrite still consumes cap and does not free a slot.
- **Does not pay** if write rate is bursty: 65,536 × 2 KiB is 128 MiB and still wraps in 90 days only if 600/day is the **mean**. A week of 2,000 rows/day wraps 65,536 in **33 days**. At 10,000/day even 65,536 is gone in **6.6 days**.

Raise-the-cap is the right move when the estate already owns tens of MiB and review is always ad hoc on the live box.

## Strategy B — archive to a monthly store

Leave the hot cap at **2,048** (4.0 MiB, wrap ~11 days). Copy every row with `event_time` in that UTC month to an append-only file `events-YYYYMM.jsonl` (or a monthly SQLite), then evict only archived **terminal** rows. Do not evict live `claimed`/`running`.

Volume at R = 180, 2 KiB/row:

| Period | Rows | Archive size |
|---|---:|---:|
| One month (30 d) | 5,400 | **10.5 MiB** |
| One year (365 d) | 65,700 | **128 MiB** |
| Three years | 197,100 | **385 MiB** |

At R = 10,000 the monthly file is 10,000 × 30 × 2 KiB ≈ **586 MiB**. That is the cost of keeping the 4.9-hour hot window from being the entire audit trail.

A 14-day incident in September is in `events-202609.jsonl` even if the hot table wrapped three times since.

Cost trade-off:

- **Pays:** hot path stays 4.0 MiB and 2,048-row sweeps; audit horizon is years, not days; `reboot_interrupted` and watchdog fingerprints survive eviction.
- **Costs:** a monthly job that must succeed (failed archive + continued eviction **is** the original failure mode); reconcile is two lookups (hot, then `events-YYYYMM*`); UTC month names must not be local dates (`TASK-07-DATE-ROLLOVER.md`); 2× write amplification on the evict path (hot already wrote; archive writes again); one fsync per batch (e.g. 64 rows / 128 KiB) on the insert that would wrap.
- **Does not pay** if the archive job is “best effort.” If September’s copy is skipped, September’s wrap still deletes the incident before October’s review.

Minimum archive SLA (`TASK-22-ROLLING-WINDOW-AUDIT.md`):

```
archive(row) → fsync → verify count/digest → then evict(row)
```

Incomplete archive **blocks eviction**. If the cap is full and archive has not succeeded, refuse the new insert (or spill). Never evict to make room for a live write without a copy.

## Which cost to take

| If | Prefer |
|---|---|
| Review lag ≤ 14 days and mean ≤ 180 rows/day | A at N = 16,384 (32 MiB) |
| Review lag ≥ 30 days, or peaks ≥ 600 rows/day | B; keep hot N = 2,048 and monthly 10.5 MiB files (586 MiB/month at R = 10,000) |
| Need both fast live sweep and 90-day proof | A at 16,384 **plus** B for months 2+ (hot is cache, monthly is source of truth) |

Do not treat “row missing” as “work never existed.” After wrap, missing is the default.
