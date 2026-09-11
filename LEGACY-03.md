# LEGACY-03

A check that returns **empty** on both “I looked and found nothing” and “I did not look” is dangerous because the caller cannot tell **absence of a finding** from **absence of a search**. Downstream then takes the empty path: mark unmatched, skip the checker, or treat the estate as healthy (`TASK-20-FAIL-LOUD.md`, `TASK-34-METRIC-THAT-LIES.md`).

## Concrete example

`find_receipts(handoff_id)` is supposed to list receipts whose `re` is that handoff (`TASK-04-MATCHER-TESTS.md`). Implementation:

```python
def find_receipts(store, handoff_id):
    try:
        return store.query("re = ?", handoff_id)
    except TimeoutError:
        return []   # failed search
```

Handoff `20260910-alpha` **has** a receipt on disk. The store times out. The function returns `[]` — the same value as a new handoff with no receipt.

`close_if_unmatched` sees empty and either leaves work OPEN forever (false hole) or records COMPLETED / “nothing to verify” (over-claim, `TASK-10-SELF-REPORT.md`). A third party cannot rerun a check that never ran. `unmatched_handoffs = 0` on the dashboard is a lying zero (`TASK-34-METRIC-THAT-LIES.md`).

Empty and finding-none are **one wire type**. The finding “zero receipts after a completed scan” is a real result. The empty from timeout is not a finding.

## Mechanical guard

**Empty is only legal after a successful scan.** Failure must be a different type (raise or `UNKNOWN`), never `[]`.

```python
def find_receipts(store, handoff_id):
    try:
        return store.query("re = ?", handoff_id)  # [] only if the query finished
    except TimeoutError as exc:
        raise SearchFailed("query failed") from exc
```

Callers may treat `[]` as “no receipts.” They must not treat `SearchFailed` as that. Pair the empty count with `last_successful_scan` (UTC). If the instrument did not finish, disposition is UNKNOWN, not 0 (`TASK-21-WATCHDOG-SILENCE.md`).
