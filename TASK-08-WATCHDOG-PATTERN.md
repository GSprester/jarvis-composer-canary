# TASK-08-WATCHDOG-PATTERN

Reference implementation of a **silent watchdog**: a scheduled script that prints nothing when healthy and speaks only on anomaly. This checkout has no production watchdog; the script below is the pattern.

Rules:

1. **Silence is health.** No stdout/stderr on a successful healthy probe.
2. **One alert per episode.** An episode is identified by a state fingerprint. Repeat probes with the same fingerprint stay silent after the first speak.
3. **Probe failure is UNKNOWN, not healthy.** A failed probe must not take the silent path.
4. **Never overwrite last-known-good on a failed probe.** LKG updates only after a successful healthy probe.

## Usage

```text
python3 silent_watchdog.py --state STATE.json --probe {healthy|anomaly|unknown}
python3 silent_watchdog.py --self-test
```

Extract the script from the fence that starts `#!/usr/bin/env python3`.

## Checker

```python
#!/usr/bin/env python3
"""Silent watchdog: speak only on anomaly, once per fingerprint.

A scheduled probe should be quiet when the estate is healthy. Operators
treat any output as a page. Therefore:

- healthy + successful probe -> print nothing; refresh last-known-good
- anomaly -> print one line if this fingerprint is a new episode
- probe failure -> UNKNOWN (not healthy): speak if new fingerprint;
  do not write last-known-good

Stdlib only. No network.
"""

from __future__ import annotations

import argparse
import hashlib
import json
import sys
from pathlib import Path
from typing import Any


def fingerprint(kind: str, detail: str) -> str:
    payload = f"{kind}\n{detail}".encode("utf-8")
    return hashlib.sha256(payload).hexdigest()


def load_state(path: Path) -> dict[str, Any]:
    if not path.is_file():
        return {"last_alert_fp": None, "last_known_good": None}
    return json.loads(path.read_text(encoding="utf-8"))


def save_state(path: Path, state: dict[str, Any]) -> None:
    path.write_text(json.dumps(state, indent=2, sort_keys=True) + "\n", encoding="utf-8")


def run_probe(probe: str, detail: str) -> tuple[str, str]:
    """Return (kind, detail). kind is healthy | anomaly | unknown."""
    if probe == "healthy":
        return "healthy", detail
    if probe == "anomaly":
        return "anomaly", detail
    # probe failure, timeout, unreadable target, or explicit unknown
    return "unknown", detail or "probe_failed"


def tick(state: dict[str, Any], kind: str, detail: str) -> tuple[dict[str, Any], str | None]:
    """Advance watchdog state. Returns (new_state, alert_or_None).

    Prints are the caller's job: None means stay silent.
    """
    state = {
        "last_alert_fp": state.get("last_alert_fp"),
        "last_known_good": state.get("last_known_good"),
    }

    if kind == "healthy":
        state["last_known_good"] = {"detail": detail}
        state["last_alert_fp"] = None  # episode closed; next anomaly is new
        return state, None

    # anomaly or unknown: both are speakable episodes, never LKG writes
    fp = fingerprint(kind, detail)
    if state["last_alert_fp"] == fp:
        return state, None
    state["last_alert_fp"] = fp
    return state, f"{kind.upper()} {fp[:12]} {detail}"


def main(argv: list[str]) -> int:
    parser = argparse.ArgumentParser(add_help=True)
    parser.add_argument("--state", type=Path, help="JSON state file")
    parser.add_argument(
        "--probe",
        choices=("healthy", "anomaly", "unknown"),
        help="synthetic probe outcome for this reference",
    )
    parser.add_argument("--detail", default="estate", help="fingerprint material")
    parser.add_argument("--self-test", action="store_true")
    args = parser.parse_args(argv[1:])

    if args.self_test:
        return self_test()
    if args.state is None or args.probe is None:
        print("usage: silent_watchdog.py --state STATE.json --probe {healthy|anomaly|unknown}", file=sys.stderr)
        return 2

    kind, detail = run_probe(args.probe, args.detail)
    state, alert = tick(load_state(args.state), kind, detail)
    save_state(args.state, state)
    if alert is not None:
        print(alert)
        return 1 if kind != "healthy" else 0
    return 0


def self_test() -> int:
    failures = 0

    def check(name: str, ok: bool) -> None:
        nonlocal failures
        print(f"{'PASS' if ok else 'FAIL'}: {name}", file=sys.stderr)
        if not ok:
            failures += 1

    # 1. healthy is silent and writes LKG
    s, alert = tick({"last_alert_fp": None, "last_known_good": None}, "healthy", "ok")
    check("healthy is silent", alert is None)
    check("healthy writes LKG", s["last_known_good"] == {"detail": "ok"})

    # 2. same anomaly fingerprints to one episode
    s1, a1 = tick(s, "anomaly", "disk-full")
    s2, a2 = tick(s1, "anomaly", "disk-full")
    check("first anomaly speaks", a1 is not None and a1.startswith("ANOMALY"))
    check("repeat anomaly is silent", a2 is None)
    check("LKG unchanged during anomaly", s2["last_known_good"] == {"detail": "ok"})

    # 3. new fingerprint is a new episode
    s3, a3 = tick(s2, "anomaly", "queue-stuck")
    check("new anomaly fingerprint speaks", a3 is not None and a3 != a1)

    # 4. probe failure is UNKNOWN, not healthy, does not overwrite LKG
    lkg_before = s3["last_known_good"]
    s4, a4 = tick(s3, "unknown", "probe_failed")
    check("probe failure speaks UNKNOWN", a4 is not None and a4.startswith("UNKNOWN"))
    check("probe failure does not overwrite LKG", s4["last_known_good"] == lkg_before)
    s5, a5 = tick(s4, "unknown", "probe_failed")
    check("repeat UNKNOWN same fingerprint is silent", a5 is None)
    check("still no LKG overwrite", s5["last_known_good"] == lkg_before)

    # 5. healthy after unknown: silent, refreshes LKG, next unknown is a new speak
    s6, a6 = tick(s5, "healthy", "recovered")
    check("recovery is silent", a6 is None)
    check("recovery refreshes LKG", s6["last_known_good"] == {"detail": "recovered"})
    s7, a7 = tick(s6, "unknown", "probe_failed")
    check("UNKNOWN after recovery is a new episode", a7 is not None)

    return 1 if failures else 0


if __name__ == "__main__":
    sys.exit(main(sys.argv))
```

## Behaviour table

| Probe | Output | LKG | Episode |
|---|---|---|---|
| healthy | none | overwritten with this reading | cleared |
| anomaly (new fingerprint) | one `ANOMALY` line | unchanged | `last_alert_fp` set |
| anomaly (same fingerprint) | none | unchanged | unchanged |
| unknown / probe failure (new fingerprint) | one `UNKNOWN` line | **unchanged** | `last_alert_fp` set |
| unknown (same fingerprint) | none | **unchanged** | unchanged |

Exit status: `0` when silent-healthy; `1` when an alert line is printed; `2` on usage error. A scheduler can still treat non-zero as “spoke.”
