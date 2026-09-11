# TASK-21-WATCHDOG-SILENCE

Health probe with three outcomes: **healthy**, **unhealthy**, **UNKNOWN**. A probe that cannot run (timeout, raise, missing callable) reports **UNKNOWN**, never healthy (`TASK-08-WATCHDOG-PATTERN.md`, `TASK-20-FAIL-LOUD.md`). This checkout has no production probe; the script is the implementation.

## Usage

```text
python3 health_probe.py --self-test
```

Extract the script from the fence that starts `#!/usr/bin/env python3`.

## Implementation

```python
#!/usr/bin/env python3
"""Three-way health probe: healthy | unhealthy | UNKNOWN.

UNKNOWN is for 'I did not obtain a reading': timeout, exception,
or missing callable. That path must not report healthy and must
not overwrite last-known-good.

Stdlib only. No network.
"""

from __future__ import annotations

import sys
import threading
from typing import Callable


Outcome = str  # healthy | unhealthy | UNKNOWN


def run_probe(fn: Callable[[], str] | None, timeout_s: float = 0.2) -> tuple[Outcome, str]:
    if fn is None or not callable(fn):
        return "UNKNOWN", "probe_cannot_run"

    box: dict[str, tuple[str, object]] = {}

    def wrap() -> None:
        try:
            box["r"] = ("ok", fn())
        except Exception as exc:
            box["r"] = ("raise", exc)

    thread = threading.Thread(target=wrap, daemon=True)
    thread.start()
    thread.join(timeout_s)
    if thread.is_alive():
        return "UNKNOWN", "timeout"
    if "r" not in box:
        return "UNKNOWN", "probe_cannot_run"

    kind, val = box["r"]
    if kind == "raise":
        return "UNKNOWN", type(val).__name__
    if val == "unhealthy":
        return "unhealthy", "probe_reported_unhealthy"
    if val == "healthy":
        return "healthy", "probe_reported_healthy"
    return "UNKNOWN", "probe_bad_reading"


def self_test() -> int:
    failures = 0

    def check(name: str, ok: bool) -> None:
        nonlocal failures
        print(f"{'PASS' if ok else 'FAIL'}: {name}")
        if not ok:
            failures += 1

    outcome, detail = run_probe(lambda: "healthy")
    check("healthy reading", outcome == "healthy" and detail == "probe_reported_healthy")

    outcome, detail = run_probe(lambda: "unhealthy")
    check("unhealthy reading", outcome == "unhealthy")

    def hang() -> str:
        threading.Event().wait(5)
        return "healthy"

    outcome, detail = run_probe(hang, timeout_s=0.05)
    check("timeout is UNKNOWN not healthy", outcome == "UNKNOWN" and detail == "timeout")

    def boom() -> str:
        raise RuntimeError("store unreachable")

    outcome, detail = run_probe(boom)
    check("raise is UNKNOWN not healthy", outcome == "UNKNOWN" and detail == "RuntimeError")

    outcome, detail = run_probe(None)
    check("missing callable is UNKNOWN", outcome == "UNKNOWN" and detail == "probe_cannot_run")

    outcome, _ = run_probe(lambda: "healthy")
    check("successful healthy is not UNKNOWN", outcome != "UNKNOWN")

    return 1 if failures else 0


def main(argv: list[str]) -> int:
    if argv[1:] == ["--self-test"]:
        return self_test()
    print("usage: health_probe.py --self-test", file=sys.stderr)
    return 2


if __name__ == "__main__":
    sys.exit(main(sys.argv))
```

## Outcomes

| Probe | Outcome | Last-known-good |
|---|---|---|
| Returns `"healthy"` | healthy | may update |
| Returns `"unhealthy"` | unhealthy | unchanged |
| Exceeds timeout | UNKNOWN | unchanged |
| Raises | UNKNOWN | unchanged |
| Missing / not callable | UNKNOWN | unchanged |

Silence (no page) is allowed only for **healthy**. Timeout and raise speak as UNKNOWN (`TASK-08-WATCHDOG-PATTERN.md`).
