# TASK-33-SCHEMA-EVOLUTION

Append-only event log that **adds fields** without breaking old readers. This checkout has no production log; the script is the contract (`TASK-11-IDEMPOTENCE.md` framing still applies: flush, then HWM).

Rules:

1. **Reader ignores unknown fields.** Extra keys are not errors.
2. **Writer never removes a field** once it has been written in any shipped version. Deprecation is add-only.
3. **Rename** is not an in-place swap. Add the new name, write both, then (later) stop *requiring* the old name — do not delete it from new events until all readers are gone, and never rewrite history.

## Usage

```text
python3 schema_evolution.py --self-test
```

Extract the script from the fence that starts `#!/usr/bin/env python3`.

## Reader and writer

```python
#!/usr/bin/env python3
"""Append-only events: ignore unknown keys; never drop known keys.

Stdlib only. No network.
"""

from __future__ import annotations

import json
import sys
from typing import Any

# v1 required. v2 added "seat". Writer still emits v1 keys forever.
KNOWN_V1 = ("type", "re", "outcome")
KNOWN_V2 = KNOWN_V1 + ("seat",)


class SchemaError(Exception):
    pass


def write_event(fields: dict[str, Any], version: int = 2) -> bytes:
    required = KNOWN_V2 if version >= 2 else KNOWN_V1
    missing = [k for k in required if k not in fields]
    if missing:
        raise SchemaError("writer must not omit " + ",".join(missing))
    # Never strip unknown-to-us keys the caller already set (forward compat
    # if a newer writer ran). Never delete a known key.
    payload = dict(fields)
    for key in required:
        payload[key] = fields[key]
    return (json.dumps(payload, ensure_ascii=True, separators=(",", ":"), sort_keys=True) + "\n").encode("utf-8")


def read_event(blob: bytes, version: int = 1) -> dict[str, Any]:
    obj = json.loads(blob.decode("utf-8"))
    if not isinstance(obj, dict):
        raise SchemaError("event must be an object")
    required = KNOWN_V1 if version < 2 else KNOWN_V2
    missing = [k for k in required if k not in obj]
    if missing:
        raise SchemaError("missing required " + ",".join(missing))
    # Ignore unknown fields: only project what this reader understands.
    return {k: obj[k] for k in required}


def self_test() -> int:
    failures = 0

    def check(name: str, ok: bool) -> None:
        nonlocal failures
        print(f"{'PASS' if ok else 'FAIL'}: {name}")
        if not ok:
            failures += 1

    v2 = write_event({"type": "receipt", "re": "20260910-alpha", "outcome": "ok", "seat": "seat-a"})
    old_reader = read_event(v2, version=1)
    check("v1 reader ignores seat", set(old_reader) == {"type", "re", "outcome"})
    check("v1 reader still sees re", old_reader["re"] == "20260910-alpha")

    v1 = write_event({"type": "receipt", "re": "20260910-alpha", "outcome": "ok"}, version=1)
    check("v1 writer bytes lack seat", b"seat" not in v1)

    omitted = False
    try:
        write_event({"type": "receipt", "re": "20260910-alpha", "seat": "seat-a"}, version=2)
    except SchemaError:
        omitted = True
    check("v2 writer cannot drop outcome", omitted)

    # Rename migration: add outcome_v2, keep outcome
    renamed = write_event(
        {
            "type": "receipt",
            "re": "20260910-alpha",
            "outcome": "ok",
            "seat": "seat-a",
            "outcome_v2": "ok",
        }
    )
    dual = json.loads(renamed)
    check("rename writes both names", dual["outcome"] == dual["outcome_v2"] == "ok")
    check("old reader still uses outcome", read_event(renamed, version=1)["outcome"] == "ok")

    return 1 if failures else 0


def main(argv: list[str]) -> int:
    if argv[1:] == ["--self-test"]:
        return self_test()
    print("usage: schema_evolution.py --self-test", file=sys.stderr)
    return 2


if __name__ == "__main__":
    sys.exit(main(sys.argv))
```

## Rename migration

Do **not** replace `"outcome"` with `"result"` in new writes and leave old readers looking for `"outcome"`.

| Step | Writer | Reader |
|---|---|---|
| 0 | `outcome` only | require `outcome` |
| 1 | write **`outcome` and `result`** (same meaning) | v1 uses `outcome`; v2 prefers `result` if present else `outcome` |
| 2 | still write both | all shipped readers accept either |
| 3 (optional, years later) | may stop *documenting* `outcome` as the preferred name | never rewrite archived months (`TASK-22-ROLLING-WINDOW-AUDIT.md`) to delete `outcome` |

Removing a field from **new** events while old files still exist is allowed only after step 2 is universal. Removing it from **old** events is a history rewrite and breaks HWM/idempotent replay (`TASK-11-IDEMPOTENCE.md`).

## Rule

**Add fields. Readers project a known subset. Writers emit every field they have ever emitted. Rename = dual-write, never swap.**
