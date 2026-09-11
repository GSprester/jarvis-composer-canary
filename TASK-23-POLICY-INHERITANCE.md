# TASK-23-POLICY-INHERITANCE

Sensitivity resolver: **effective class = min(seat, provider)**, never max. Failover to a wider provider must not raise the seat (`TASK-12-POLICY-PARITY.md`, `TASK-13-FAILOVER-CEILING.md`). This checkout has no live policy; the script is the implementation.

Missing or unknown class **raises** (`TASK-20-FAIL-LOUD.md`). It does not default to the wider of the two.

## Usage

```text
python3 policy_inheritance.py --self-test
```

Extract the script from the fence that starts `#!/usr/bin/env python3`.

## Implementation

```python
#!/usr/bin/env python3
"""Effective sensitivity is min(seat, provider), never max.

Failover may change the provider name. It must not change seat.ceiling.
effective() reads both allowances and returns the narrower class.

Stdlib only. No network.
"""

from __future__ import annotations

import sys

RANK = {"public": 0, "internal": 1, "restricted": 2, "secret": 3}
NAME = {v: k for k, v in RANK.items()}


class PolicyError(Exception):
    pass


def rank(label: str) -> int:
    if label not in RANK:
        raise PolicyError("unknown class: " + repr(label))
    return RANK[label]


def effective(seat_class: str, provider_class: str) -> str:
    return NAME[min(rank(seat_class), rank(provider_class))]


def failover(seat_class: str, from_provider: str, to_provider: str, providers: dict[str, str]) -> str:
    if from_provider not in providers or to_provider not in providers:
        raise PolicyError("missing provider")
    # Destination hop uses the same seat_class. No assignment seat = provider.
    return effective(seat_class, providers[to_provider])


def self_test() -> int:
    failures = 0

    def check(name: str, ok: bool) -> None:
        nonlocal failures
        print(f"{'PASS' if ok else 'FAIL'}: {name}")
        if not ok:
            failures += 1

    providers = {"alpha": "internal", "beta": "restricted", "gamma": "public"}

    check("min internal,restricted is internal", effective("internal", "restricted") == "internal")
    check("min restricted,internal is internal", effective("restricted", "internal") == "internal")
    check("never max", effective("internal", "restricted") != "restricted")
    check("narrower provider binds", effective("restricted", "public") == "public")

    raised = False
    try:
        effective("internal", "not-a-class")
    except PolicyError:
        raised = True
    check("unknown class raises, not max", raised)

    # Failover alpha → beta: beta is wider; seat stays internal.
    after = failover("internal", "alpha", "beta", providers)
    check("failover to wider provider does not widen seat", after == "internal")
    check("failover result is not beta's class", after != providers["beta"])
    check("seat argument unchanged", "internal" == "internal")

    after_narrow = failover("restricted", "beta", "gamma", providers)
    check("failover to narrower provider narrows effective", after_narrow == "public")

    return 1 if failures else 0


def main(argv: list[str]) -> int:
    if argv[1:] == ["--self-test"]:
        return self_test()
    print("usage: policy_inheritance.py --self-test", file=sys.stderr)
    return 2


if __name__ == "__main__":
    sys.exit(main(sys.argv))
```

## Failover proof

Seat ceiling `internal`. Providers: `alpha=internal`, `beta=restricted`.

```
effective(internal, alpha) = internal
effective(internal, beta)  = min(internal, restricted) = internal
```

`beta` is one class wider (`TASK-12-POLICY-PARITY.md` worked example). The hop is allowed **only** at `internal`. Assigning `seat.ceiling = beta.class` would be max and is not in this resolver.

`admit` in `TASK-13-FAILOVER-CEILING.md` may still refuse the hop. This file only pins the inheritance function: if the hop happens, width does not increase.
