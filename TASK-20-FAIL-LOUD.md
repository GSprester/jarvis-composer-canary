# TASK-20-FAIL-LOUD

A search or validation function must **raise** (or return a distinct error) on failure. Returning `[]` / `{}` / `None`-as-empty makes “found nothing” and “could not look” the same value. Callers then take the healthy/empty path (`TASK-08-WATCHDOG-PATTERN.md`: probe failure is UNKNOWN, not healthy). That is fail-open (`TASK-02-POLICY-AUDIT.md`).

## The indistinguishability

```python
def find_receipts(store, handoff_id):
    try:
        rows = store.query("re = ?", handoff_id)
    except TimeoutError:
        return []          # WRONG: failed search looks like no receipts
    return rows


def close_if_unmatched(store, handoff_id):
    receipts = find_receipts(store, handoff_id)
    if not receipts:
        # Caller cannot tell "zero receipts" from "query died".
        mark_open(handoff_id)          # or worse: mark_completed("nothing left")
        return "empty"
    return match(handoff_id, receipts)
```

Concrete case: handoff `20260910-alpha` **has** a receipt (`re` exact, `TASK-04-MATCHER-TESTS.md`). The store times out or the path is unreadable. `find_receipts` returns `[]`.

Same `[]` as a brand-new handoff that truly has no receipt.

## Downstream decision that goes wrong

`close_if_unmatched` treats `[]` as “no evidence.” Two wrong closings, depending on the product default:

| Caller policy on empty | What happens on a **failed** search | Correct outcome |
|---|---|---|
| “No receipts → still OPEN” | Live matched work is left OPEN; operators re-run; lock held (`TASK-16-ONE-LEAD-LOCK.md`) | Should be UNKNOWN / retry, not a new OPEN episode |
| “No receipts → nothing to do / COMPLETED” | Work is closed **without** an artifact check (`TASK-10-SELF-REPORT.md`, `TASK-17-SEPARATION-OF-DUTIES.md`) | Should stay UNVERIFIED; checker never ran |

The second is the over-claim: empty and failed are indistinguishable, so the seat records completion because the search was silent. A third party cannot tell the receipt was never read.

Validation has the same shape:

```python
def load_expected_digest(handoff):
    try:
        return [handoff["expected_sha256"]]
    except KeyError:
        return []          # WRONG: missing required field == "no digests"


def check_job(handoff, path):
    expected = load_expected_digest(handoff)
    if not expected:
        return 0           # empty means skip — file never hashed
    return checker(path, expected[0])
```

A malformed handoff (no `expected_sha256`) returns `[]`, the checker is skipped, exit 0. Failed validation and “no constraint” are the same. The artifact can be anything, including missing.

## Raise instead

```python
class SearchFailed(Exception):
    pass


def find_receipts(store, handoff_id):
    try:
        return store.query("re = ?", handoff_id)
    except TimeoutError as exc:
        raise SearchFailed("query failed") from exc
    # Missing table / bad schema: also raise. Only a successful
    # query that matched zero rows returns [].


def close_if_unmatched(store, handoff_id):
    receipts = find_receipts(store, handoff_id)   # may raise
    if not receipts:
        return "open"       # now this really means zero rows
    return match(handoff_id, receipts)
```

`[]` is reserved for a **completed** search with zero hits. Failure is a different type. Watchdog maps that to UNKNOWN and does not overwrite last-known-good. Job status stays UNVERIFIED until a check actually runs.

Rule: **empty means I looked and there were none; if I did not look, raise.**
