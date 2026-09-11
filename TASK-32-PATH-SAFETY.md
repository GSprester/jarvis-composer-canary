# TASK-32-PATH-SAFETY

Validator for a declared artifact path (`TASK-06-ARTIFACT-CHECK.md`, `TASK-24-ARTIFACT-DECLARATION.md`). Reject **traversal**, **absolute paths that escape the root**, and **Windows reserved names**. Tests use both `/` and `\` separators.

**Validate the declared string first, then join to root.** Resolving before validating is the vulnerability (below).

## Usage

```text
python3 path_safety.py --self-test
```

Extract the script from the fence that starts `#!/usr/bin/env python3`.

## Validator

```python
#!/usr/bin/env python3
"""Lexical path safety: reject .., absolute escape, Windows reserved names.

Do not realpath/abspath the user string before these checks. Normalising
first erases the evidence of traversal.

Stdlib only. No network.
"""

from __future__ import annotations

import re
import sys
from pathlib import Path, PureWindowsPath

RESERVED = {
    "CON",
    "PRN",
    "AUX",
    "NUL",
    *(f"COM{i}" for i in range(1, 10)),
    *(f"LPT{i}" for i in range(1, 10)),
}


class PathUnsafe(Exception):
    pass


def split_components(declared: str) -> list[str]:
    normalised = declared.replace("\\", "/")
    return [p for p in normalised.split("/") if p not in ("", ".")]


def is_absolute_declared(declared: str) -> bool:
    s = declared.replace("\\", "/")
    if s.startswith("/"):
        return True
    # Windows drive or UNC
    if re.match(r"^[a-zA-Z]:", declared):
        return True
    if declared.startswith("\\\\") or declared.startswith("//"):
        return True
    return False


def has_reserved(declared: str) -> bool:
    for part in split_components(declared):
        stem = part.split(".")[0].upper()
        # CON, CON.txt, CON:stream
        stem = stem.split(":")[0]
        if stem in RESERVED:
            return True
        if PureWindowsPath(part).stem.upper() in RESERVED:
            return True
    return False


def validate_declared(declared: str, root: Path) -> Path:
    if "\x00" in declared:
        raise PathUnsafe("nul byte")
    if ".." in split_components(declared):
        raise PathUnsafe("traversal")
    if is_absolute_declared(declared):
        raise PathUnsafe("absolute")
    if has_reserved(declared):
        raise PathUnsafe("reserved")
    # Only after lexical reject: join. Then confirm we did not escape.
    joined = (root / declared.replace("\\", "/")).resolve()
    try:
        joined.relative_to(root.resolve())
    except ValueError:
        raise PathUnsafe("escapes_root") from None
    return joined


def self_test() -> int:
    failures = 0

    def check(name: str, ok: bool) -> None:
        nonlocal failures
        print(f"{'PASS' if ok else 'FAIL'}: {name}")
        if not ok:
            failures += 1

    root = Path("/tmp/path-safety-root")
    root.mkdir(parents=True, exist_ok=True)

    def rejects(declared: str, reason: str) -> bool:
        try:
            validate_declared(declared, root)
            return False
        except PathUnsafe as exc:
            return reason in str(exc)

    check("posix traversal", rejects("docs/../../../etc/passwd", "traversal"))
    check("windows traversal", rejects("docs\\..\\..\\windows\\system32", "traversal"))
    check("posix absolute", rejects("/etc/passwd", "absolute"))
    check("windows absolute drive", rejects("C:\\Windows\\System32\\config", "absolute"))
    check("windows reserved CON slash", rejects("out/CON", "reserved"))
    check("windows reserved CON backslash", rejects("out\\CON.txt", "reserved"))
    check("windows reserved NUL both seps", rejects("a/NUL", "reserved") and rejects("a\\NUL", "reserved"))
    check("relative safe posix", validate_declared("docs/changelog.md", root).is_relative_to(root.resolve()) if hasattr(Path, "is_relative_to") else True)
    ok_win = False
    try:
        validate_declared("docs\\changelog.md", root)
        ok_win = True
    except PathUnsafe:
        ok_win = False
    check("relative safe windows seps", ok_win)

    # Resolve-then-validate would miss this: .. is gone after abspath
    sneaky = "docs/foo/../../outside.txt"
    check("traversal still visible before resolve", ".." in split_components(sneaky))

    return 1 if failures else 0


def main(argv: list[str]) -> int:
    if argv[1:] == ["--self-test"]:
        return self_test()
    print("usage: path_safety.py --self-test", file=sys.stderr)
    return 2


if __name__ == "__main__":
    sys.exit(main(sys.argv))
```

## Why resolving before validating is the vulnerability

`abspath` / `realpath` / `Path.resolve()` **erase** `..` and may **follow a symlink** out of the root **before** any policy runs.

```python
# WRONG
resolved = Path(root, user_path).resolve()   # already outside, or .. gone
if ".." in str(resolved):                    # never true after resolve
    reject()
use(resolved)
```

`docs/foo/../../etc/passwd` becomes `/etc/passwd`. A validator that only looks at the resolved string does not see `..`, may only check `startswith(root)` after a symlink swap, or may open the target while “checking.” The filesystem has already been interpreted.

Lexical checks on the **declared** string see `..`, `/`, `\`, `C:`, `CON`. Join and resolve happen **after** reject. A second `relative_to(root)` is a belt, not a substitute for the first pass.

Same class as hash-before-normalise (`TASK-15-IDEMPOTENT-RECEIPT.md`): you computed on a different object than the one you later enforce.
