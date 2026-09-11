# TASK-24-ARTIFACT-DECLARATION

Declare-then-verify: a unit **declares** `(path, expected_sha256)` **before** it writes. Completion is decided by the **caller** checking that path exists and matches the declared digest (`TASK-06-ARTIFACT-CHECK.md`, `TASK-17-SEPARATION-OF-DUTIES.md`). The worker has no completion bit.

Empty leftover and stale leftover from a previous run both exist as files and both fail verify.

## Usage

```text
python3 artifact_declaration.py --self-test
```

Extract the script from the fence that starts `#!/usr/bin/env python3`.

## Module

```python
#!/usr/bin/env python3
"""Declare output path+digest before work; caller verifies.

COMPLETED iff the declared path exists, is a regular file, and
sha256(bytes) == declaration.expected. Empty files and stale
previous-run bytes fail unless they happen to match the digest
(which an empty file does only if the declaration expected empty).

Stdlib only. No network.
"""

from __future__ import annotations

import hashlib
import sys
import tempfile
from dataclasses import dataclass
from pathlib import Path


def sha256_bytes(data: bytes) -> str:
    return hashlib.sha256(data).hexdigest()


class DeclarationError(Exception):
    pass


@dataclass(frozen=True)
class Declaration:
    path: Path
    expected: str


def declare(path: Path, expected: str) -> Declaration:
    expected = expected.strip().lower()
    if len(expected) != 64 or any(c not in "0123456789abcdef" for c in expected):
        raise DeclarationError("expected digest must be 64 hex chars")
    return Declaration(path=path, expected=expected)


def verify(decl: Declaration, repo_root: Path) -> str:
    """Caller-only verdict. Returns COMPLETED or UNVERIFIED."""
    try:
        resolved = decl.path.resolve()
        resolved.relative_to(repo_root.resolve())
    except (ValueError, OSError):
        return "UNVERIFIED"
    if not resolved.is_file():
        return "UNVERIFIED"
    actual = sha256_bytes(resolved.read_bytes())
    return "COMPLETED" if actual == decl.expected else "UNVERIFIED"


def self_test() -> int:
    failures = 0

    def check(name: str, ok: bool) -> None:
        nonlocal failures
        print(f"{'PASS' if ok else 'FAIL'}: {name}")
        if not ok:
            failures += 1

    payload = b"helix-7 changelog canonical\n"
    expected = sha256_bytes(payload)
    empty = sha256_bytes(b"")
    stale = b"previous run leftover\n"

    with tempfile.TemporaryDirectory() as tmp:
        root = Path(tmp)
        out = root / "docs" / "changelog-helix7.md"
        decl = declare(out, expected)
        check("declare before the file exists", not out.exists())
        check("verify before write is UNVERIFIED", verify(decl, root) == "UNVERIFIED")

        # Empty leftover (crashed after create, before write)
        out.parent.mkdir(parents=True)
        out.write_bytes(b"")
        check("empty file exists", out.is_file() and out.stat().st_size == 0)
        check("empty file hash is empty digest", sha256_bytes(out.read_bytes()) == empty)
        check("empty file does not match declared payload", verify(decl, root) == "UNVERIFIED")

        # Stale previous run at the same path
        out.write_bytes(stale)
        check("stale file exists", out.is_file())
        check("stale hash != declared", sha256_bytes(stale) != expected)
        check("stale previous run is UNVERIFIED", verify(decl, root) == "UNVERIFIED")

        # Honest write after declare
        out.write_bytes(payload)
        check("matching bytes is COMPLETED", verify(decl, root) == "COMPLETED")

        # Caller decides; worker record is irrelevant
        check("declaration still names the same digest", decl.expected == expected)

    return 1 if failures else 0


def main(argv: list[str]) -> int:
    if argv[1:] == ["--self-test"]:
        return self_test()
    print("usage: artifact_declaration.py --self-test", file=sys.stderr)
    return 2


if __name__ == "__main__":
    sys.exit(main(sys.argv))
```

## Cases

| State at `path` | vs declared digest | Caller verdict |
|---|---|---|
| Missing | — | UNVERIFIED |
| Exists, empty | empty digest ≠ payload digest | UNVERIFIED |
| Exists, stale previous-run bytes | stale digest ≠ payload digest | UNVERIFIED |
| Exists, declared bytes | match | COMPLETED |

Existence alone is not completion. The digest was fixed at **declare**, before start, so a leftover file cannot close the unit (`TASK-10-SELF-REPORT.md`).
