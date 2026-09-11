# TASK-06-ARTIFACT-CHECK

Dependency-free checker: exit 0 only when the declared path is inside this repository, the file exists, and its SHA-256 matches the expected digest passed beside it. Anything else is non-zero. This checkout has no job store; the script below is the checker.

## Usage

```text
python3 artifact_check.py <declared-path> <expected-sha256>
python3 artifact_check.py --self-test
```

Extract the script from the fence that starts `#!/usr/bin/env python3`.

## Cases covered by `--self-test`

| Case | Result |
|---|---|
| Missing file | non-zero |
| Empty file, expected digest is SHA-256 of empty bytes | 0 |
| Empty file, expected digest is some other hash | non-zero |
| Correct hash of a non-empty file | 0 |
| Wrong hash of an existing file | non-zero |
| Path outside the repo (`/tmp/...` or `../...` escape) | non-zero (refuse; do not hash) |

## Checker

```python
#!/usr/bin/env python3
"""Check a declared artifact path against an expected SHA-256.

Exit 0 only if all of the following hold:
  - the path resolves inside the repository root
  - the path is an existing regular file
  - sha256(file bytes) equals the expected hex digest

Why artifact existence (plus digest) is a stronger completion signal than a
worker-written status field: a status bit is authored by the same process that
wants to be seen as done, so it cannot be audited after the worker is gone or
if the worker lies. A file at a declared path with a matching sha256 can be
re-read by a third party who never sat in that seat. Missing, empty-when-not-
expected, wrong-hash, or out-of-repo paths fail closed (non-zero), which is
UNVERIFIED rather than a completion claim.

Stdlib only. No network.
"""

from __future__ import annotations

import hashlib
import os
import sys
import tempfile
from pathlib import Path

EMPTY_SHA256 = hashlib.sha256(b"").hexdigest()  # e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855


def find_repo_root(start: Path) -> Path:
    cur = start.resolve()
    for candidate in (cur, *cur.parents):
        if (candidate / ".git").exists():
            return candidate
    raise FileNotFoundError("no .git directory found from " + str(start))


def is_inside(root: Path, path: Path) -> bool:
    try:
        path.resolve().relative_to(root.resolve())
        return True
    except (ValueError, OSError):
        return False


def check_artifact(declared: str, expected: str, repo_root: Path) -> int:
    expected = expected.strip().lower()
    if len(expected) != 64 or any(c not in "0123456789abcdef" for c in expected):
        print("error: expected digest must be 64 lowercase hex characters", file=sys.stderr)
        return 2

    raw = Path(declared)
    # Refuse absolute paths that are not under the repo, and refuse
    # resolution that escapes via .. or symlinks, before hashing.
    candidate = raw if raw.is_absolute() else (repo_root / raw)
    try:
        resolved = candidate.resolve(strict=False)
    except OSError as exc:
        print(f"error: cannot resolve path: {exc}", file=sys.stderr)
        return 1

    if not is_inside(repo_root, resolved):
        print("error: path is outside the repository; refuse", file=sys.stderr)
        return 1

    if not resolved.exists():
        print("error: artifact file does not exist", file=sys.stderr)
        return 1
    if not resolved.is_file():
        print("error: artifact path is not a regular file", file=sys.stderr)
        return 1

    digest = hashlib.sha256(resolved.read_bytes()).hexdigest()
    if digest != expected:
        print(f"error: sha256 mismatch actual={digest} expected={expected}", file=sys.stderr)
        return 1
    return 0


def self_test() -> int:
    repo_root = find_repo_root(Path.cwd())
    failures = 0

    def expect(name: str, path: str, digest: str, want: int) -> None:
        nonlocal failures
        got = check_artifact(path, digest, repo_root)
        # Map all failures to boolean non-zero vs zero.
        ok = (got == 0) if want == 0 else (got != 0)
        status = "PASS" if ok else "FAIL"
        print(f"{status}: {name} (exit {got}, want {'0' if want == 0 else 'non-zero'})")
        if not ok:
            failures += 1

    with tempfile.TemporaryDirectory(dir=repo_root) as tmp:
        tmp_path = Path(tmp)
        missing = tmp_path / "missing.bin"
        empty = tmp_path / "empty.bin"
        empty.write_bytes(b"")
        payload = tmp_path / "payload.bin"
        payload.write_bytes(b"helix-7-canary\n")
        good = hashlib.sha256(b"helix-7-canary\n").hexdigest()
        wrong = hashlib.sha256(b"not-the-bytes\n").hexdigest()

        expect("missing file", str(missing.relative_to(repo_root)), good, 1)
        expect("empty file correct hash", str(empty.relative_to(repo_root)), EMPTY_SHA256, 0)
        expect("empty file wrong hash", str(empty.relative_to(repo_root)), good, 1)
        expect("correct hash", str(payload.relative_to(repo_root)), good, 0)
        expect("wrong hash", str(payload.relative_to(repo_root)), wrong, 1)

    outside = Path("/tmp/task-06-outside-artifact.bin")
    outside.write_bytes(b"outside\n")
    try:
        expect("path outside repo (absolute)", str(outside), hashlib.sha256(b"outside\n").hexdigest(), 1)
    finally:
        outside.unlink(missing_ok=True)

    escape = os.path.join("..", "etc", "passwd") if Path("/etc/passwd").exists() else os.path.join("..", "..", "tmp")
    expect("path outside repo (relative escape)", escape, EMPTY_SHA256, 1)

    return 1 if failures else 0


def main(argv: list[str]) -> int:
    if len(argv) == 2 and argv[1] == "--self-test":
        return self_test()
    if len(argv) != 3:
        print("usage: artifact_check.py <declared-path> <expected-sha256>", file=sys.stderr)
        print("       artifact_check.py --self-test", file=sys.stderr)
        return 2
    repo_root = find_repo_root(Path.cwd())
    return check_artifact(argv[1], argv[2], repo_root)


if __name__ == "__main__":
    sys.exit(main(sys.argv))
```
