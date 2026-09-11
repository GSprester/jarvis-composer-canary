# LEGACY-10

Determinism means: the same inputs yield the same outputs. Accuracy means: the output matches the thing you intended to measure. A procedure can be perfectly repeatable and still be wrong every time. Repeatability is a property of the **instrument**. Truth is a property of the **predicate**.

Concrete measurement: “does this receipt close handoff `20260910-alpha`?” A naive matcher answers by comparing the first eight characters of the two **names** (`TASK-07-DATE-ROLLOVER.md`). Handoff `20260910-alpha`, receipt `20260911-alpha-r0` with `re` exactly `20260910-alpha`. Every run, on every machine, the prefix test is false. The pair is dropped. That is **deterministic**. It is not **accurate**. The identity the estate defined is `receipt.re == handoff.name` (`TASK-04-MATCHER-TESTS.md`). The instrument measured a different quantity (calendar prefix) and got a stable zero.

The same split shows up as a **count**. `unmatched_handoffs` is computed as “`find_receipts` returned `[]`.” If the store times out, the function returns `[]` (`LEGACY-03.md`). Run the dashboard twice; you get `0` twice. Deterministic. The receipt is on disk. The measurement was “did this query fail-open,” not “are there zero receipts.” `open_units == 0` after a hot-table wrap is the same: the counter is stable and the work is not gone (`TASK-34-METRIC-THAT-LIES.md`).

A hash can be deterministic and inaccurate if you hash **before** normalisation (`TASK-15-IDEMPOTENT-RECEIPT.md`). Same dict insertion order, same digest, every time — and that digest is not the digest of the file a third party will read. `exists(path)` is a deterministic boolean and not a completion check (`LEGACY-05.md`).

Accuracy needs a predicate that names the obligation (declared digest, `re`, `pid+start`) and a check that **actually ran** (fail closed if it did not, `LEGACY-07.md`). Determinism is what you want **after** that predicate is right. Determinism without the right predicate is a reliable lie. Do not confuse a stable number with a true one.
