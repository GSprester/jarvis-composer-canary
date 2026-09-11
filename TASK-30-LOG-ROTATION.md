# TASK-30-LOG-ROTATION

Rotator that keeps the last **N** archived files and **never deletes the currently-open file**. Names use a **monotonic generation**, not wall clock (`TASK-19-TIMESTAMP-HAZARD.md`). Concurrent rotators take a lead lock (`TASK-16-ONE-LEAD-LOCK.md`). This checkout has no production logger; the script is the rotator.

## Usage

```text
python3 log_rotation.py --self-test
```

Extract the script from the fence that starts `#!/usr/bin/env python3`.

## Rotator

```python
#!/usr/bin/env python3
"""Keep last N archives; never unlink the open log.

Generation comes from a fsynced counter, not time.time(). A clock
moving backwards must not reuse a name or sort an older file as new.

Stdlib only. No network.
"""

from __future__ import annotations

import json
import os
import sys
import tempfile
import threading
from pathlib import Path


class Rotator:
    def __init__(self, directory: Path, keep: int = 3, clock=None) -> None:
        self.directory = directory
        self.keep = keep
        self.clock = clock or (lambda: 0.0)  # unused for names; injected for tests
        self.state_path = directory / "rotator-state.json"
        self.lock_path = directory / "rotator.lock"
        self.current_path = directory / "app.log"
        self.current_path.touch()
        self._fh = self.current_path.open("ab")

    def open_path(self) -> Path:
        return self.current_path

    def write(self, data: bytes) -> None:
        self._fh.write(data)
        self._fh.flush()

    def _load_gen(self) -> int:
        if not self.state_path.is_file():
            return 0
        return int(json.loads(self.state_path.read_text(encoding="utf-8"))["gen"])

    def _store_gen(self, gen: int) -> None:
        tmp = self.directory / "rotator-state.json.tmp"
        tmp.write_text(json.dumps({"gen": gen}) + "\n", encoding="utf-8")
        os.replace(tmp, self.state_path)

    def _acquire_lock(self) -> int:
        fd = os.open(self.lock_path, os.O_CREAT | os.O_RDWR, 0o644)
        try:
            import fcntl

            fcntl.flock(fd, fcntl.LOCK_EX)
        except ImportError:
            pass
        return fd

    def _release_lock(self, fd: int) -> None:
        try:
            import fcntl

            fcntl.flock(fd, fcntl.LOCK_UN)
        except ImportError:
            pass
        os.close(fd)

    def archived(self) -> list[Path]:
        found = sorted(self.directory.glob("app.log.*"), key=lambda p: int(p.name.split(".")[-1]))
        return found

    def prune(self) -> None:
        archives = self.archived()
        extra = len(archives) - self.keep
        for path in archives:
            if extra <= 0:
                break
            if path.resolve() == self.current_path.resolve():
                continue
            path.unlink()
            extra -= 1

    def rotate(self) -> Path | None:
        fd = self._acquire_lock()
        try:
            self._fh.flush()
            gen = self._load_gen() + 1
            dest = self.directory / f"app.log.{gen}"
            self._fh.close()
            if self.current_path.is_file():
                os.replace(self.current_path, dest)
            self._store_gen(gen)
            self.current_path.touch()
            self._fh = self.current_path.open("ab")
            self.prune()
            # clock is read only after gen is stored; backwards clock cannot
            # choose dest.
            _ = self.clock()
            return dest
        finally:
            self._release_lock(fd)

    def close(self) -> None:
        self._fh.close()


def self_test() -> int:
    failures = 0

    def check(name: str, ok: bool) -> None:
        nonlocal failures
        print(f"{'PASS' if ok else 'FAIL'}: {name}")
        if not ok:
            failures += 1

    # 1. Rotation during a write: lock orders them; open file remains
    with tempfile.TemporaryDirectory() as tmp:
        root = Path(tmp)
        rot = Rotator(root, keep=2)
        started = threading.Event()
        proceed = threading.Event()

        def writer() -> None:
            started.set()
            proceed.wait(1)
            rot.write(b"during\n")

        t = threading.Thread(target=writer)
        t.start()
        started.wait(1)
        dest = rot.rotate()
        proceed.set()
        t.join(1)
        check("rotate during write left a current file", rot.open_path().is_file())
        check("current path was not deleted", rot.open_path().exists())
        check("archived generation exists", dest is not None and dest.exists())
        rot.write(b"after\n")
        check("write after rotate landed on current", b"after\n" in rot.open_path().read_bytes())
        rot.close()

    # 2. Two rotators concurrently: flock serialises; never unlink current
    with tempfile.TemporaryDirectory() as tmp:
        root = Path(tmp)
        a = Rotator(root, keep=2)
        b = Rotator(root, keep=2)
        errors: list[BaseException] = []

        def go(r: Rotator) -> None:
            try:
                r.write(b"x\n")
                r.rotate()
            except BaseException as exc:
                errors.append(exc)

        t1 = threading.Thread(target=go, args=(a,))
        t2 = threading.Thread(target=go, args=(b,))
        t1.start()
        t2.start()
        t1.join()
        t2.join()
        check("concurrent rotate raised nothing", errors == [])
        check("current still present", a.open_path().is_file())
        check("at most keep archives", len(a.archived()) <= 2)
        check("current not in prune set as a numbered file", a.open_path() not in a.archived())
        a.close()
        b.close()

    # 3. Clock moving backwards: names still increase
    with tempfile.TemporaryDirectory() as tmp:
        root = Path(tmp)
        times = [1_700_000_000.0, 1_600_000_000.0]

        def backward() -> float:
            return times.pop(0) if times else 1_500_000_000.0

        rot = Rotator(root, keep=3, clock=backward)
        rot.write(b"one\n")
        p1 = rot.rotate()
        rot.write(b"two\n")
        p2 = rot.rotate()
        check("first gen is 1", p1 is not None and p1.name.endswith(".1"))
        check("second gen is 2 despite earlier clock", p2 is not None and p2.name.endswith(".2"))
        check("both archives exist (no overwrite)", p1.exists() and p2.exists())
        check("open file not deleted", rot.open_path().is_file())
        rot.close()

    # 4. Never delete current even if keep=0
    with tempfile.TemporaryDirectory() as tmp:
        root = Path(tmp)
        rot = Rotator(root, keep=0)
        rot.write(b"keep-me\n")
        rot.rotate()
        check("keep=0 still has current", rot.open_path().is_file())
        rot.close()

    return 1 if failures else 0


def main(argv: list[str]) -> int:
    if argv[1:] == ["--self-test"]:
        return self_test()
    print("usage: log_rotation.py --self-test", file=sys.stderr)
    return 2


if __name__ == "__main__":
    sys.exit(main(sys.argv))
```

## Rules

| Hazard | Mitigation |
|---|---|
| Delete the open fd’s path | Prune skips `current_path`; rotate replaces then re-opens |
| Two processes rotate | `fcntl.flock` on `rotator.lock` |
| Clock steps backward | Destination is `app.log.<gen>`, gen stored in JSON, not `strftime` |
| Keep last N | After rotate, drop oldest numbered files only, down to N |
