# TASK-17-SEPARATION-OF-DUTIES

Tests that a worker cannot mark its own work `COMPLETED` without an independent check. This checkout has no worker runtime; the script is the contract (`TASK-10-SELF-REPORT.md`: `COMPLETED` is a view).

The adversarial case is a worker that writes a plausible completion record (`status=completed`, a path, and a sha256). Fabrication is detectable because the **expected digest is on the handoff**, not taken from the worker’s record. The checker hashes the file and compares to that declared digest.

## Usage

```text
python3 separation_of_duties.py --self-test
```

Extract the script from the fence that starts `#!/usr/bin/env python3`.

## Tests

```python
#!/usr/bin/env python3
"""Worker cannot close work; only an independent checker can.

The worker may write bytes and a self-report. It has no set_completed().
COMPLETED(job) iff checker(declared_path, handoff.expected_sha256) == 0.
The worker-supplied sha256 in a completion record is ignored.

Fabrication is detectable: the handoff carries expected_sha256 before the
worker runs. A plausible record that copies or invents a digest still fails
when the file bytes do not match the handoff digest (or the file is missing).

Stdlib only. No network.
"""

from __future__ import annotations

import hashlib
import sys
import tempfile
from pathlib import Path


def sha256_bytes(data: bytes) -> str:
    return hashlib.sha256(data).hexdigest()


class WorkerForbidden(Exception):
    pass


class Job:
    def __init__(self, unit: str, path: Path, expected_sha256: str) -> None:
        self.unit = unit
        self.path = path
        self.expected_sha256 = expected_sha256
        self.worker_record: dict | None = None

    def status(self, repo_root: Path) -> str:
        if check_artifact(self.path, self.expected_sha256, repo_root) == 0:
            return "COMPLETED"
        return "UNVERIFIED"


def check_artifact(declared: Path, expected: str, repo_root: Path) -> int:
    try:
        resolved = declared.resolve()
        resolved.relative_to(repo_root.resolve())
    except (ValueError, OSError):
        return 1
    if not resolved.is_file():
        return 1
    actual = sha256_bytes(resolved.read_bytes())
    return 0 if actual == expected.lower() else 1


class Worker:
    def write_bytes(self, path: Path, data: bytes) -> None:
        path.parent.mkdir(parents=True, exist_ok=True)
        path.write_bytes(data)

    def write_completion_record(self, job: Job, record: dict) -> None:
        job.worker_record = dict(record)

    def set_completed(self, job: Job) -> None:
        raise WorkerForbidden("worker has no write path to COMPLETED")


def self_test() -> int:
    failures = 0

    def check(name: str, ok: bool) -> None:
        nonlocal failures
        print(f"{'PASS' if ok else 'FAIL'}: {name}")
        if not ok:
            failures += 1

    with tempfile.TemporaryDirectory() as tmp:
        root = Path(tmp)
        payload = b"helix-7 changelog canonical\n"
        expected = sha256_bytes(payload)
        artifact = root / "docs" / "changelog-helix7.md"
        job = Job("20260910-alpha", artifact, expected)
        worker = Worker()

        # 1. No API to self-complete
        forbidden = False
        try:
            worker.set_completed(job)
        except WorkerForbidden:
            forbidden = True
        check("worker set_completed is forbidden", forbidden)
        check("no file yet is UNVERIFIED", job.status(root) == "UNVERIFIED")

        # 2. Honest path: worker writes declared bytes; checker closes
        worker.write_bytes(artifact, payload)
        check("independent check of declared bytes is COMPLETED", job.status(root) == "COMPLETED")

        # 3. Adversarial: plausible completion record, wrong or missing bytes
        job2 = Job("20260910-beta", root / "docs" / "notes-beta.md", expected)
        fake_digest = sha256_bytes(b"DONE: published notes-beta.md\n")
        worker.write_completion_record(
            job2,
            {
                "status": "completed",
                "path": str(job2.path),
                "sha256": fake_digest,
                "note": "DONE: published notes-beta.md",
            },
        )
        check("record looks completed", job2.worker_record["status"] == "completed")
        check("no file: still UNVERIFIED", job2.status(root) == "UNVERIFIED")

        # Worker writes different bytes and embeds hash of those bytes
        worker.write_bytes(job2.path, b"DONE: published notes-beta.md\n")
        check("self-hash matches worker file", sha256_bytes(job2.path.read_bytes()) == fake_digest)
        check("handoff digest does not match file", sha256_bytes(job2.path.read_bytes()) != job2.expected_sha256)
        check("fabrication UNVERIFIED despite plausible record", job2.status(root) == "UNVERIFIED")

        # Worker copies the real expected digest into the record but leaves wrong bytes
        worker.write_completion_record(job2, {"status": "completed", "sha256": expected})
        check("copied expected digest still UNVERIFIED if bytes differ", job2.status(root) == "UNVERIFIED")

        # What makes it detectable: view ignores worker sha256; recomputes vs handoff
        view_uses_record = (
            job2.worker_record is not None
            and job2.worker_record.get("sha256") == expected
            and job2.status(root) == "COMPLETED"
        )
        check("worker-supplied sha256 cannot close the view", view_uses_record is False)

        # 4. Only matching bytes at the declared path close the job
        worker.write_bytes(job2.path, payload)
        check("same expected bytes then COMPLETED", job2.status(root) == "COMPLETED")

    return 1 if failures else 0


def main(argv: list[str]) -> int:
    if argv[1:] == ["--self-test"]:
        return self_test()
    print("usage: separation_of_duties.py --self-test", file=sys.stderr)
    return 2


if __name__ == "__main__":
    sys.exit(main(sys.argv))
```

## What the adversarial record looks like

```json
{"status": "completed", "path": "docs/notes-beta.md", "sha256": "<hash of the worker's own file>", "note": "DONE: published notes-beta.md"}
```

That is a completion **record**. It is not a completion **check**.

## What makes the fabrication detectable

The handoff already carries `expected_sha256` (the obligation). The view recomputes `sha256(file bytes)` and compares to **that** value. It does not read `record["sha256"]`.

| Worker action | Record | Checker vs handoff digest | View |
|---|---|---|---|
| Writes `status=completed`, no file | plausible | missing file | UNVERIFIED |
| Writes own bytes and own digest | plausible | digest ≠ handoff | UNVERIFIED |
| Copies handoff digest, wrong bytes | plausible | digest ≠ handoff | UNVERIFIED |
| Writes the declared bytes | irrelevant | match | COMPLETED |

Detectable means a third party who never sat in the seat can rerun the checker and get UNVERIFIED. The lie does not survive the file.
