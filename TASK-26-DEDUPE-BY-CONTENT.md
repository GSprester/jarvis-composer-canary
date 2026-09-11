# TASK-26-DEDUPE-BY-CONTENT

Deduplicate records by **content hash**, not by identifier. Two rows with different ids and the same canonical body collapse to one. Two rows that share an id but differ in body **do not** collapse (`TASK-15-IDEMPOTENT-RECEIPT.md`: hash after normalisation; `TASK-18-COUNT-RECONCILIATION.md`: identity is not length or label).

This checkout has no record store; the script is the function.

## Usage

```text
python3 dedupe_by_content.py --self-test
```

Extract the script from the fence that starts `#!/usr/bin/env python3`.

## Function

```python
#!/usr/bin/env python3
"""Dedupe records by sha256 of canonical content, not by id.

Content is the record minus the id field, JSON-sorted. Hash after
that step. Same id + different body => two survivors.

Stdlib only. No network.
"""

from __future__ import annotations

import hashlib
import json
import sys
from typing import Any


def canonical_content(record: dict[str, Any]) -> bytes:
    body = {k: v for k, v in record.items() if k != "id"}
    return json.dumps(body, ensure_ascii=True, separators=(",", ":"), sort_keys=True).encode("utf-8")


def content_hash(record: dict[str, Any]) -> str:
    return hashlib.sha256(canonical_content(record)).hexdigest()


def dedupe_by_content(records: list[dict[str, Any]]) -> list[dict[str, Any]]:
    seen: set[str] = set()
    out: list[dict[str, Any]] = []
    for rec in records:
        digest = content_hash(rec)
        if digest in seen:
            continue
        seen.add(digest)
        out.append(rec)
    return out


def self_test() -> int:
    failures = 0

    def check(name: str, ok: bool) -> None:
        nonlocal failures
        print(f"{'PASS' if ok else 'FAIL'}: {name}")
        if not ok:
            failures += 1

    a = {"id": "rec-1", "re": "20260910-alpha", "outcome": "ok"}
    b = {"id": "rec-2", "re": "20260910-alpha", "outcome": "ok"}
    collapsed = dedupe_by_content([a, b])
    check("different ids, identical content collapse", len(collapsed) == 1)
    check("survivor is the first occurrence", collapsed[0]["id"] == "rec-1")
    check("content hashes match", content_hash(a) == content_hash(b))

    c = {"id": "rec-9", "re": "20260910-alpha", "outcome": "ok"}
    d = {"id": "rec-9", "re": "20260910-alpha", "outcome": "reboot_interrupted"}
    kept = dedupe_by_content([c, d])
    check("same id, different content do not collapse", len(kept) == 2)
    check("hashes differ", content_hash(c) != content_hash(d))
    check("both outcomes present", {x["outcome"] for x in kept} == {"ok", "reboot_interrupted"})

    e = {"id": "z", "outcome": "ok", "re": "20260910-alpha"}
    check("key order does not split content", content_hash(a) == content_hash(e))
    check("id-only pair still collapses after reorder", len(dedupe_by_content([a, e])) == 1)

    return 1 if failures else 0


def main(argv: list[str]) -> int:
    if argv[1:] == ["--self-test"]:
        return self_test()
    print("usage: dedupe_by_content.py --self-test", file=sys.stderr)
    return 2


if __name__ == "__main__":
    sys.exit(main(sys.argv))
```

## Why not key on `id`

An id is a writer-chosen label (receipt name, date prefix, seat session). Content is `re` + `outcome` (and any other payload fields). Hashing `id` would keep two byte-identical receipts that differ only in `20260910-*` vs `20260911-*` names (`TASK-07-DATE-ROLLOVER.md`) and would drop a second row that reused an id after a stale write (`TASK-24-ARTIFACT-DECLARATION.md`).
