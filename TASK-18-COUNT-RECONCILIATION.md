# TASK-18-COUNT-RECONCILIATION

Function: given two lists that may differ in **order** and **duplicates**, return **added**, **removed**, and **unchanged** as bags (counts matter). This checkout has no production reconciler; the script is the contract.

Items are compared as exact UTF-8 strings. Whitespace-only differences are different items (`"ok"` ≠ `"ok "`). Order is ignored.

## Usage

```text
python3 count_reconciliation.py --self-test
```

Extract the script from the fence that starts `#!/usr/bin/env python3`.

## Why comparing lengths first is a common and wrong shortcut

`len(old) == len(new)` is used as “nothing changed” (or `len` delta as “how many added”). That is a net count, not a reconciliation.

- **Same length, different bags:** `["a", "a", "b"]` vs `["a", "b", "b"]` — length 3 both. One `"a"` removed, one `"b"` added. A length-equal short-circuit reports unchanged and drops the swap.
- **Same length, whitespace-only drift:** `["ok"]` vs `["ok "]` — length 1 both. If you return early, you never see one removed and one added. The artifact checker (`TASK-06-ARTIFACT-CHECK.md`) and canonical receipts (`TASK-15-IDEMPOTENT-RECEIPT.md`) treat those bytes as distinct.
- **Different length, mostly unchanged:** `["a"]` vs `["a", "a"]` — length says “+1” but not that one `"a"` is unchanged. Downstream that skips the bag walk cannot attach the extra copy to the existing id.

Length is a filter on **cardinality**, not on **identity**. Use multiset difference. Length may be an assertion *after* the bags are computed (`len(added) - len(removed) == len(new) - len(old)`), not a substitute for the walk.

## Function and tests

```python
#!/usr/bin/env python3
"""Reconcile two lists as bags: added, removed, unchanged.

Order does not matter. Duplicates do. Whitespace is significant.
Do not short-circuit on len(old) == len(new).

Stdlib only. No network.
"""

from __future__ import annotations

import sys
from collections import Counter
from typing import Iterable


def reconcile(old: Iterable[str], new: Iterable[str]) -> dict[str, list[str]]:
    left = Counter(old)
    right = Counter(new)
    unchanged: list[str] = []
    added: list[str] = []
    removed: list[str] = []
    for key in sorted(set(left) | set(right)):
        keep = min(left[key], right[key])
        unchanged.extend([key] * keep)
        if right[key] > left[key]:
            added.extend([key] * (right[key] - left[key]))
        if left[key] > right[key]:
            removed.extend([key] * (left[key] - right[key]))
    return {"added": added, "removed": removed, "unchanged": unchanged}


def self_test() -> int:
    failures = 0

    def check(name: str, ok: bool) -> None:
        nonlocal failures
        print(f"{'PASS' if ok else 'FAIL'}: {name}")
        if not ok:
            failures += 1

    r = reconcile(["b", "a", "a"], ["a", "b", "a"])
    check("order ignored, all unchanged", r == {"added": [], "removed": [], "unchanged": ["a", "a", "b"]})

    r = reconcile(["a", "a", "b"], ["a", "b", "b"])
    check(
        "same length, duplicate swap",
        r == {"added": ["b"], "removed": ["a"], "unchanged": ["a", "b"]},
    )
    check("length shortcut would be wrong here", len(["a", "a", "b"]) == len(["a", "b", "b"]) and r["added"] != [])

    r = reconcile(["ok"], ["ok "])
    check(
        "whitespace-only is add+remove, not unchanged",
        r == {"added": ["ok "], "removed": ["ok"], "unchanged": []},
    )
    check("whitespace pair has equal length", len(["ok"]) == len(["ok "]))

    r = reconcile(["x", "x", "x"], ["x"])
    check("duplicate extras removed", r == {"added": [], "removed": ["x", "x"], "unchanged": ["x"]})

    r = reconcile(["x"], ["x", "x"])
    check("duplicate extras added", r == {"added": ["x"], "removed": [], "unchanged": ["x"]})

    r = reconcile([], ["a", "a"])
    check("empty old", r == {"added": ["a", "a"], "removed": [], "unchanged": []})

    r = reconcile(["  ", ""], ["", " "])
    check(
        "empty vs space vs two-spaces are distinct",
        r == {"added": [" "], "removed": ["  "], "unchanged": [""]},
    )

    return 1 if failures else 0


def main(argv: list[str]) -> int:
    if argv[1:] == ["--self-test"]:
        return self_test()
    print("usage: count_reconciliation.py --self-test", file=sys.stderr)
    return 2


if __name__ == "__main__":
    sys.exit(main(sys.argv))
```

## Worked same-length trap

```
old = [a, a, b]   new = [a, b, b]   len 3 == 3
unchanged = [a, b]
removed   = [a]
added     = [b]
```

A length-first return of “unchanged” would hide the swapped duplicate and lie to any later receipt or lock keyed by identity (`TASK-15-IDEMPOTENT-RECEIPT.md`, `TASK-16-ONE-LEAD-LOCK.md`).
