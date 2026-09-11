# TASK-29-CONFIG-PRECEDENCE

Config resolution: **explicit argument > environment variable > file value > default**. An **empty string is set**, not absent — it must not fall through to the next rung (`TASK-20-FAIL-LOUD.md`: empty is not “I did not look”). This checkout has no live config loader; the script is the resolver.

## Usage

```text
python3 config_precedence.py --self-test
```

Extract the script from the fence that starts `#!/usr/bin/env python3`.

## Resolver

```python
#!/usr/bin/env python3
"""Resolve a key: argv, then env, then file, then default.

None / missing key means absent. "" is present and wins.

Stdlib only. No network.
"""

from __future__ import annotations

import sys
from typing import Any


ABSENT = object()


def _present(value: Any) -> bool:
    return value is not ABSENT and value is not None


def resolve(
    key: str,
    *,
    arg: Any = ABSENT,
    env: dict[str, str] | None = None,
    file_map: dict[str, Any] | None = None,
    default: Any = ABSENT,
) -> Any:
    env = env or {}
    file_map = file_map or {}
    if _present(arg):
        return arg
    if key in env:
        return env[key]
    if key in file_map:
        return file_map[key]
    if _present(default):
        return default
    raise KeyError("unresolved: " + key)


def self_test() -> int:
    failures = 0

    def check(name: str, ok: bool) -> None:
        nonlocal failures
        print(f"{'PASS' if ok else 'FAIL'}: {name}")
        if not ok:
            failures += 1

    key = "seat_ceiling"
    env = {key: "internal"}
    file_map = {key: "restricted"}
    default = "public"

    check(
        "argument beats env, file, default",
        resolve(key, arg="secret", env=env, file_map=file_map, default=default) == "secret",
    )
    check(
        "env beats file and default",
        resolve(key, env=env, file_map=file_map, default=default) == "internal",
    )
    check(
        "file beats default",
        resolve(key, file_map=file_map, default=default) == "restricted",
    )
    check(
        "default when nothing else set",
        resolve(key, default=default) == "public",
    )

    check(
        "empty arg is set, not absent",
        resolve(key, arg="", env=env, file_map=file_map, default=default) == "",
    )
    check(
        "empty env is set, not file",
        resolve(key, env={key: ""}, file_map=file_map, default=default) == "",
    )
    check(
        "empty file is set, not default",
        resolve(key, file_map={key: ""}, default=default) == "",
    )

    missing = False
    try:
        resolve("no_such_key")
    except KeyError:
        missing = True
    check("all absent raises", missing)

    return 1 if failures else 0


def main(argv: list[str]) -> int:
    if argv[1:] == ["--self-test"]:
        return self_test()
    print("usage: config_precedence.py --self-test", file=sys.stderr)
    return 2


if __name__ == "__main__":
    sys.exit(main(sys.argv))
```

## Rungs

| Winner | Why |
|---|---|
| Argument | Operator said it on this invocation |
| Env | Process admission (`TASK-05-DOCTRINE-DRAFT.md`) |
| File | Checked-in or declared artifact, not a worker status bit |
| Default | Last resort; missing required key still **raises** |

`""` on any winning rung is a deliberate empty (clear the ceiling, empty queue name, etc.). Treating it as absent would silently inherit a wider file/default class (`TASK-23-POLICY-INHERITANCE.md`).
