# TASK-12-POLICY-PARITY

Stdlib checker: compare two provider entries in a policy document and report which permissions differ. This checkout has no live policy file; the script and worked example are the contract.

## Usage

```text
python3 policy_parity.py --policy POLICY.json --left PROVIDER --right PROVIDER
python3 policy_parity.py --self-test
```

Extract the script from the fence that starts `#!/usr/bin/env python3`.

A permission is a string. Sensitivity class is the permission `class:<name>`. The checker reports symmetric difference and, separately, a class-only delta.

## Checker

```python
#!/usr/bin/env python3
"""Diff permissions on two provider entries in one policy document.

Exit 0 if the permission sets are identical. Exit 1 if they differ.
Exit 2 on usage or missing keys. Stdlib only. No network.
"""

from __future__ import annotations

import argparse
import json
import sys
from pathlib import Path


def permissions_of(entry: dict) -> set[str]:
    perms = set(entry.get("permissions") or [])
    if "class" in entry:
        perms.add("class:" + str(entry["class"]))
    return perms


def diff_providers(policy: dict, left_name: str, right_name: str) -> dict:
    providers = policy.get("providers") or {}
    if left_name not in providers or right_name not in providers:
        missing = [n for n in (left_name, right_name) if n not in providers]
        raise KeyError("missing provider entries: " + ", ".join(missing))
    left = permissions_of(providers[left_name])
    right = permissions_of(providers[right_name])
    only_left = sorted(left - right)
    only_right = sorted(right - left)
    class_left = {p for p in left if p.startswith("class:")}
    class_right = {p for p in right if p.startswith("class:")}
    return {
        "left": left_name,
        "right": right_name,
        "only_left": only_left,
        "only_right": only_right,
        "class_only_delta": sorted((class_left ^ class_right))
        if (left - class_left) == (right - class_right) and class_left != class_right
        else [],
        "equal": left == right,
    }


def main(argv: list[str]) -> int:
    parser = argparse.ArgumentParser()
    parser.add_argument("--policy", type=Path)
    parser.add_argument("--left")
    parser.add_argument("--right")
    parser.add_argument("--self-test", action="store_true")
    args = parser.parse_args(argv[1:])
    if args.self_test:
        return self_test()
    if not args.policy or not args.left or not args.right:
        print("usage: policy_parity.py --policy POLICY.json --left P --right P", file=sys.stderr)
        return 2
    try:
        report = diff_providers(json.loads(args.policy.read_text(encoding="utf-8")), args.left, args.right)
    except KeyError as exc:
        print(f"error: {exc}", file=sys.stderr)
        return 2
    print(json.dumps(report, indent=2, sort_keys=True))
    return 0 if report["equal"] else 1


def self_test() -> int:
    failures = 0

    def check(name: str, ok: bool) -> None:
        nonlocal failures
        print(f"{'PASS' if ok else 'FAIL'}: {name}")
        if not ok:
            failures += 1

    policy = {
        "providers": {
            "alpha": {"permissions": ["read", "write"], "class": "internal"},
            "beta": {"permissions": ["read", "write"], "class": "restricted"},
            "gamma": {"permissions": ["read", "write"], "class": "internal"},
        }
    }
    same = diff_providers(policy, "alpha", "gamma")
    check("identical entries are equal", same["equal"] is True)

    report = diff_providers(policy, "alpha", "beta")
    check("one-class pair is not equal", report["equal"] is False)
    check("only_left is class:internal", report["only_left"] == ["class:internal"])
    check("only_right is class:restricted", report["only_right"] == ["class:restricted"])
    check("class_only_delta names both classes", report["class_only_delta"] == ["class:internal", "class:restricted"])
    check("no non-class permission drift", set(report["only_left"] + report["only_right"]) == {"class:internal", "class:restricted"})
    return 1 if failures else 0


if __name__ == "__main__":
    sys.exit(main(sys.argv))
```

## Worked example — differ by one sensitivity class

```json
{
  "providers": {
    "alpha": { "permissions": ["read", "write"], "class": "internal" },
    "beta":  { "permissions": ["read", "write"], "class": "restricted" }
  }
}
```

```text
python3 policy_parity.py --policy example.json --left alpha --right beta
```

```json
{
  "class_only_delta": ["class:internal", "class:restricted"],
  "equal": false,
  "left": "alpha",
  "only_left": ["class:internal"],
  "only_right": ["class:restricted"],
  "right": "beta"
}
```

`read` and `write` match. The only differing permission is the sensitivity class: `internal` vs `restricted`. That single bump is enough for a failover to exceed a seat that declared `internal` as its ceiling.

## Why keying by provider name lets failover exceed the ceiling

Policy keyed by **provider name** (`alpha`, `beta`) attaches the permission set to the vendor bucket, not to the **model** the seat admitted (`helix-7-small`, `helix-7-large`).

A seat’s declared ceiling is a model-level promise: this admission may handle up to `class:internal`. Failover is usually implemented as “try the other provider.” The lookup key becomes `beta`. `beta` is one class higher. No new writer admission runs (`TASK-05-DOCTRINE-DRAFT.md` says resume/failover must take the same admission as a cold start). The seat keeps its old identity and picks up `restricted` because that is what the name `beta` maps to.

The exceed is silent:

- The seat object still says `ceiling=internal`.
- The provider entry that is now in force says `class:restricted`.
- Nothing in a name-keyed table asks whether `beta`’s class is ≤ the seat’s ceiling.
- A model-keyed table (`helix-7-small` → `internal` on any provider) would deny the path, or require a fresh admission, because the model did not change its ceiling when the vendor did.

Checker output above is the diff a failover wrapper should run **before** switching names. If `class_only_delta` is non-empty and the right-hand class is above the seat ceiling, refuse. Do not treat “same provider family” as “same permissions.”
