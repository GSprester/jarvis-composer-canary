# TASK-22-ROLLING-WINDOW-AUDIT

Fixed-capacity event table, oldest-first eviction (same store class as `TASK-09-RETENTION-RISK.md`). This checkout has no table; numbers are a stated estate, not measurements here.

## Fraction of a day retained

Let **N** = cap in rows, **R** = mean events per UTC day. The hot table holds

```
hours_retained = 24 × (N / R)
fraction_of_day = N / R
```

When `R > N`, less than one day remains.

| N | R (events/day) | N / R | Hours retained | Fraction of a day |
|---|---:|---:|---:|---:|
| 2,048 | 180 | 11.38 d | 273 | **11.38** (more than a day) |
| 2,048 | 600 | 3.41 d | 82 | **3.41** |
| 2,048 | 2,048 | 1.00 d | 24 | **1.00** |
| 2,048 | 10,000 | 0.2048 d | 4.9 | **0.205** |

At **10,000 events/day** (retry storm, watchdog ticks, failed-search loops from `TASK-20-FAIL-LOUD.md`) the table keeps **20.5% of one day** — about **4 hours 55 minutes**. Row size 2 KiB → hot footprint still **4.0 MiB**; rate, not bytes, is what shrinks the window.

Peak hour: 2,000 events in 60 minutes. Window = 2,048 / 2,000 ≈ **1.02 hours**. Almost the entire UTC day is already gone by the next hour.

## When an incident becomes unreconstructable

An incident needs a span of **S** days of rows (handoff, receipt, lock identity, watchdog UNKNOWN, reboot_interrupted). Review happens **L** days after the first event.

```
unreconstructable  iff  S + L > N / R
```

The oldest needed row is evicted when more than **N** later rows have been written. After that, “row missing” equals “never happened” (`TASK-09-RETENTION-RISK.md`).

Concrete:

| R | N/R | S = 0.5 d, L = 2 d (weekend) | Reconstructable? |
|---|---:|---|---|
| 180 | 11.38 d | 2.5 < 11.38 | yes |
| 600 | 3.41 d | 2.5 < 3.41 | yes, barely |
| 2,048 | 1.00 d | 2.5 > 1.00 | **no** — gone ~1 day after start |
| 10,000 | 0.205 d | 2.5 > 0.205 | **no** — gone in **4.9 h** |

Monthly review (L = 30, S = 3): unreconstructable unless `N/R > 33`. At R = 180 that needs N > 5,940. The 2,048 cap fails by day **11**.

The **point** is the first insert that makes `rows_written_since_incident > N`. At R = 10,000 that insert is number 2,049, about **4.9 hours** after the incident starts. After that, the receipt `re`, the lock `pid+start`, and the first UNKNOWN fingerprint cannot be checked.

## Archive-before-evict

**Rule:** do not delete a hot row until it is in an append-only monthly store **and** that write is durable.

```
archive(row) → fsync → verify count/digest → then evict(row)
```

Failed archive **blocks eviction** (`TASK-20-FAIL-LOUD.md`: do not treat archive failure as empty/success). If the cap is full and archive has not succeeded, refuse the new insert (or spill to a side file). Never evict to make room for a live write without a copy.

Cost (same 2 KiB rows, R = 180):

| Item | Amount |
|---|---|
| Extra write per evicted row | 1 × 2 KiB to `events-YYYYMM.jsonl` (UTC month, `TASK-19-TIMESTAMP-HAZARD.md`) |
| Amplification on the evict path | **2×** (hot already wrote; archive writes again) |
| Monthly archive volume | 180 × 30 × 2 KiB ≈ **10.5 MiB** |
| Year | ≈ **128 MiB** |
| Verify | sha256 of the appended batch, or row count ≥ month inserts; fail → no evict |
| Latency | one fsync per batch (e.g. 64 rows / 128 KiB) on the insert that would wrap |
| Operator | two lookups on incident (hot, then month file); failed monthly job stalls inserts at cap |

At R = 10,000, monthly archive is 10,000 × 30 × 2 KiB ≈ **586 MiB/month**. That is the cost of keeping the 4.9-hour hot window from being the entire audit trail.

Archive-before-evict does not raise N. It makes wrap **safe**: the fraction of a day in RAM can stay 0.205; the incident remains reconstructable from the month file.
