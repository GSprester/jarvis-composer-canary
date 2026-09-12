# TASK-07-DATE-ROLLOVER

This checkout has no matcher implementation. The bug is: a handoff named with a **local** calendar date is never paired to a receipt named with the **next UTC** date when the matcher joins on `YYYYMMDD` name prefixes instead of `receipt.re == handoff.name`.

## Minimal reproducing example

Seat zone `America/Los_Angeles` (PDT, UTC−7) on **2026-09-10**. The writer opens a handoff at **23:30 local**. That instant is already **2026-09-11 06:30Z**. The handoff **name** still uses the local date. The receipt is minted after 00:00 UTC, so its **name** uses the next UTC date. `receipt.re` still names the handoff.

```
local  2026-09-10 23:30 -07:00
UTC    2026-09-11 06:30Z
handoff.name  = 20260910-alpha     # local YYYYMMDD
receipt.name  = 20260911-alpha-r0  # next UTC YYYYMMDD
receipt.re    = 20260910-alpha     # identity (correct)
```

```python
handoff_name = "20260910-alpha"          # local date on the writer
receipt_name = "20260911-alpha-r0"       # next UTC date on the receipt
receipt_re   = "20260910-alpha"          # identity pointer (correct)

# Failing predicate: same YYYYMMDD prefix
assert handoff_name[:8] == receipt_name[:8]   # False; pair dropped

# Corrected predicate: receipt.re names this handoff
assert receipt_re == handoff_name             # True; pair kept
```

Naive join (`handoff.name[:8] == receipt.name[:8]`) returns no pair. The work looks unmatched even though `re` is exact. Do not parse, increment, or “fix up” dates. Date prefixes are display, not keys (`TASK-04-MATCHER-TESTS.md`, `TASK-19-TIMESTAMP-HAZARD.md`).

## Predicates

| | Predicate | Result on the repro |
|---|---|---|
| Failing | `handoff.name[:8] == receipt.name[:8]` | no match (`20260910` ≠ `20260911`) |
| Corrected | `receipt.re == handoff.name` | match (`20260910-alpha`) |

## Edge cases

Handoff name stays `20260910-alpha`. Receipt `re` stays `20260910-alpha`. Only the receipt **name** date varies.

| Case | Receipt name | Failing prefix predicate | Corrected `re` predicate |
|---|---|---|---|
| Same day | `20260910-alpha-r0` | match (accidental) | match |
| +1 day (UTC rollover) | `20260911-alpha-r0` | **no match** | match |
| −1 day | `20260909-alpha-r0` | **no match** | match |

Same-day names hide the bug: the prefix join succeeds for the wrong reason. +1 day is the UTC rollover that drops a valid receipt. −1 day is the opposite skew (receipt named with yesterday relative to the handoff’s local date) and is also dropped by the prefix predicate.

A matcher that uses the failing predicate fails `TASK-04-MATCHER-TESTS.md` (`test_utc_date_rollover_receipt_next_day_still_binds`).
