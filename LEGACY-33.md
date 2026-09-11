# LEGACY-33

A reader that treats **`[]` as a negative finding** (“no receipts,” “nothing due,” “no exemptions,” “key unknown”) has decided that **empty means the look completed and the bag is zero**. That decision is only sound if the instrument can still see a row that **must** be there. Without that, empty is also “I did not look,” “I looked at yesterday’s UTC partition,” or “the hot table wrapped” (`LEGACY-03.md`, `TASK-19-TIMESTAMP-HAZARD.md`, `TASK-22-ROLLING-WINDOW-AUDIT.md`). Raising on timeout is necessary and not sufficient: a **finished** query of the wrong index is a completed `[]` and a lying zero (`LEGACY-10.md`, `TASK-34-METRIC-THAT-LIES.md`).

A **positive control** is that known-present row. If the control is missing, the empty is UNKNOWN, not a finding. The reader must not close units, skip jobs, or rotate keys (`LEGACY-31.md`, `LEGACY-23.md`).

`last_successful_scan` only proves the function returned. It does not prove the function could retrieve live estate. Length-zero is a cardinality shortcut (`TASK-18-COUNT-RECONCILIATION.md`). The control is identity, not `len > 0` of some other bag.

## What the control should look like

One **canary** object, declared before this reader runs (`TASK-24-ARTIFACT-DECLARATION.md`):

| Property | Rule |
|---|---|
| Identity | Fixed `re` / `unit` (e.g. `canary-control`), never a local `YYYYMMDD` |
| Store | **Same** table, partition scheme, and matcher as the query under test (`receipt.re == name`, `TASK-04-MATCHER-TESTS.md`) |
| Proof | Two-factor: `verify(path, expected)` **and** a checker receipt citing that digest (`LEGACY-32.md`). Existence of a stub canary must not bless empty |
| Author | Not this scan, not this observer (`LEGACY-13.md`, `LEGACY-17.md`). Do not insert the canary so the look “succeeds” |
| Retention | Not evicted by the hot-window cap, or copied into every partition the reader might hit. Missing canary after wrap → UNKNOWN, not 0 due |

```
scan(store, want):
    hits = query(store, re=want)          # may be []
    ctrl = query(store, re=CANARY_RE)
    if query failed: raise SearchFailed   # truncated, not negative
    if not (A_and_B(ctrl)):               # canary not two-factor visible
        return UNKNOWN                    # empty is not a finding
    return hits                           # [] is now a negative finding
```

The canary’s region is this store. A control that only lives in another endpoint is a wrong-region bind (`LEGACY-24.md`). Successor harnesses keep the same `CANARY_RE` or name a `successor_id` (`LEGACY-29.md`, `LEGACY-15.md`).

**Rule:** Empty may mean none only after a completed look that **found the canary**. No canary, no negative.
