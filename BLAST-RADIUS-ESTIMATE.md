# blast radius estimate

Defect: a path that emits **confident success** (`status=completed`, `exit 0`, `exists`, prefix match, `released` without delivery, empty stdout as NONE) while A∧B is false (`LEGACY-32.md`, `LEGACY-31.md`). Radius `R` = set of `{unit, enqueue_id}` that took **that path**, not the set that looks red now.

**Fail if `open_units=0` ⇒ R=∅** (`TASK-34-METRIC-THAT-LIES.md`). **Fail if you only rescan “failed” rows:** the defect hid them as success. **Fail if you mint new ids to “recheck”:** duplicates (`TASK-27-RETRY-SEMANTICS.md`).

## Cheapest decisive check

Take **one** row the estate still calls success in the incident window. Run MEASURE only (`GATE-DEADLOCK.md`):

```
decisive = verify(declared_path, handoff.expected) AND checker-receipt.re==unit
```

If `decisive` is false or ERROR: **R is at least every success emitted by the same predicate since `last_successful_canary`** (not since “this unit”). Stop claiming “isolated.”

**Fail if you pick a unit the canary already blesses and ignore the rest.** **Fail if `exists(path)` is the check:** stubs pass. **Fail if timeout ⇒ “looks fine”:** UNKNOWN is not decisive-ok (`POSITIVE-CONTROL.md`). **Fail if you fix the one row and close the incident:** that is R=1 by assertion.

Cheaper still, when the log has the field: **any** `released` with missing `delivery.evidence_sha256` (`IDEMPOTENT-EVENT-LEDGER.md`), or `kind=sent`. One such line ⇒ all `released` from that writer version are in R until proven otherwise. **Fail if you treat “no such field” as old-schema healthy** (`TASK-33-SCHEMA-EVOLUTION.md` — missing required ⇒ not a success).

If `canary-miss` ever appeared in FOUND (`LEGACY-39.md`): R includes **all** joins from that matcher build. No sample needed. **Fail if you then fuzzy-exclude “the obvious ones”** (`RECEIPT-MATCHING.md`).

## Measurement ladder

Each rung **widens** R. Do not skip to a dashboard count. Canary-hit must be visible on every look or the rung is ERROR, not 0 (`LEGACY-33.md`).

| # | Measure | Adds to R | Failure if used alone |
|---|---|---|---|
| 1 | Identify the success **predicate** (code path / `rung` / hook) | the defect name | Blaming “the seat” — substitutes and cron share it (`SUBSTITUTE-REVIEW-QUEUE.md`) |
| 2 | Count hot+**archive** rows with that success bit in `[t0,t1]` UTC | candidates | Hot wrap drops the incident (`TASK-22-ROLLING-WINDOW-AUDIT.md`) |
| 3 | Controls: miss matched? hit missing? | all scans in window | Skipping controls ⇒ empty look looks like R=0 |
| 4 | Cheap decisive on 1 row (above) | whole predicate×window if fail | Stopping at 1 WO |
| 5 | Recompute A∧B on **all** candidates (MEASURE, declared paths) | exact R | Mutating to “repair” mid-count (`LEGACY-17.md`) |
| 6 | Downstream that **read** those bits as settled (skip, failover, `review_closed`, LKG) | consumers | Counting only writers — detectors skipped jobs (`LEGACY-09.md`) |

Window `[t0,t1]`: `t0` = last `canary-hit` + `canary-miss` both ok **before** first bad success; `t1` = now or deploy of the fix. **Fail if window is “last 24h local”:** UTC/date-prefix hole (`TASK-19-TIMESTAMP-HAZARD.md`). **Fail if t0 is first user report:** silent greens already shipped.

Bag-reconcile candidates vs A∧B results (`TASK-18-COUNT-RECONCILIATION.md`). **Fail if `len(success)==len(files)` ⇒ all good.**

List order = `order_key` (`LEGACY-35.md`). **Fail if `ls` newest-first is the sample frame:** you miss the 494 unquoted receipts.

## After the bound

Every `unit` in R → UNVERIFIED, same id, archive stubs that block presence (`PROVENANCE-ARCHIVE.md`). **Fail if you delete the lying success rows:** you cannot recompute R tomorrow. **Fail if `P` ACKs the set:** asserted close. Steal only on `same=false` (`LIVENESS-PREDICATE.md`).

**Rule:** One failed A∧B on a green row implicates the predicate, not the row. Count the predicate’s successes in the canary-bounded window. That bag is R.
