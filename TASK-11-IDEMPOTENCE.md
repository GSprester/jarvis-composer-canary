# TASK-11-IDEMPOTENCE

Stdlib tests for an append-only event log. This checkout has no production log; the script below is the contract.

Guarantees under test:

1. Replay of a flushed prefix is **idempotent** (no duplicated derived state).
2. A crash **between write and flush** leaves **no partial record visible**.
3. That holds because the **high-water mark advances only after flush**.

## Usage

```text
python3 event_log_idempotence.py --self-test
```

Extract the script from the fence that starts `#!/usr/bin/env python3`.

## Tests

```python
#!/usr/bin/env python3
"""Append-only log: idempotent replay, no visible torn writes.

ORDERING RULE: write the record bytes, then flush (fsync) those bytes,
then advance the high-water mark to the new end. Never advance the
high-water mark before flush. Readers must expose only offsets in
[0, high_water_mark). Visibility is the mark, not the file length.

A crash between write and flush may leave junk or a torn tail past the
mark. Because the mark did not move, a reader ignores that tail. A crash
after flush but before the mark moves also hides the record (same rule);
the next append may overwrite the un-marked tail. Either way, no partial
record is visible.

Stdlib only. No network.
"""

from __future__ import annotations

import json
import struct
import sys
from dataclasses import dataclass, field


# Framing: 4-byte big-endian length + UTF-8 JSON payload. A torn write is
# a length prefix without a full payload, or bytes past the high-water mark.
HEADER = struct.Struct(">I")


@dataclass
class AppendLog:
    buf: bytearray = field(default_factory=bytearray)
    hwm: int = 0
    flushed: int = 0
    fail_before_flush: bool = False
    fail_before_hwm: bool = False

    def append(self, event: dict) -> None:
        payload = json.dumps(event, separators=(",", ":"), sort_keys=True).encode("utf-8")
        record = HEADER.pack(len(payload)) + payload

        # ORDERING RULE (write → flush → high-water mark):
        # 1. write the framed record into the buffer
        # 2. flush (durable) those bytes
        # 3. only then advance hwm
        # Readers see buf[:hwm]. File length / flushed length is not visibility.
        self.buf.extend(record)
        if self.fail_before_flush:
            raise Crash("crash between write and flush")
        self.flushed = len(self.buf)
        if self.fail_before_hwm:
            raise Crash("crash between flush and high-water mark")
        self.hwm = self.flushed

    def visible_bytes(self) -> bytes:
        return bytes(self.buf[: self.hwm])

    def replay(self) -> list[dict]:
        data = self.visible_bytes()
        events: list[dict] = []
        offset = 0
        while offset + HEADER.size <= len(data):
            (n,) = HEADER.unpack_from(data, offset)
            offset += HEADER.size
            if offset + n > len(data):
                raise TornRecord("length prefix past high-water mark")
            events.append(json.loads(data[offset : offset + n].decode("utf-8")))
            offset += n
        if offset != len(data):
            raise TornRecord("trailing bytes inside high-water mark")
        return events


class Crash(Exception):
    pass


class TornRecord(Exception):
    pass


def fold(events: list[dict]) -> dict[str, dict]:
    """Idempotent fold: last write per event id wins; replay is a set."""
    out: dict[str, dict] = {}
    for event in events:
        out[event["id"]] = event
    return out


def self_test() -> int:
    failures = 0

    def check(name: str, ok: bool) -> None:
        nonlocal failures
        print(f"{'PASS' if ok else 'FAIL'}: {name}")
        if not ok:
            failures += 1

    e1 = {"id": "evt-1", "body": "claimed"}
    e2 = {"id": "evt-2", "body": "receipt"}

    # 1. Replay without duplication
    log = AppendLog()
    log.append(e1)
    log.append(e2)
    first = log.replay()
    second = log.replay()
    check("replay is list-identical", first == second)
    once = fold(first)
    twice = fold(first + second)
    check("replay twice yields same id set", once == twice)
    check("two distinct ids after double replay", set(twice) == {"evt-1", "evt-2"})
    check("fold is not a bag (len 2 not 4)", len(twice) == 2)

    # 2. Crash between write and flush: no partial record visible
    log2 = AppendLog()
    log2.append(e1)
    hwm_after_first = log2.hwm
    log2.fail_before_flush = True
    crashed = False
    try:
        log2.append(e2)
    except Crash:
        crashed = True
    check("crash between write and flush raised", crashed)
    check("hwm unchanged after unflushed write", log2.hwm == hwm_after_first)
    visible = log2.replay()
    check("unflushed record not visible", [e["id"] for e in visible] == ["evt-1"])
    check("file tail may exist but is past hwm", len(log2.buf) > log2.hwm)

    # 3. HWM-after-flush is what makes the guarantee hold
    log3 = AppendLog()
    log3.append(e1)
    log3.fail_before_hwm = True
    try:
        log3.append(e2)
    except Crash:
        pass
    check("flushed-but-unmarked record not visible", [e["id"] for e in log3.replay()] == ["evt-1"])
    check("hwm did not advance before flush+mark", log3.hwm < log3.flushed)

    # Counterfactual: if HWM moved before flush, a torn tail would be visible.
    torn = AppendLog()
    torn.append(e1)
    payload = json.dumps(e2, separators=(",", ":"), sort_keys=True).encode("utf-8")
    torn.buf.extend(HEADER.pack(len(payload)))  # length only; no body
    torn.hwm = len(torn.buf)  # WRONG: mark advanced before flush/full write
    torn_visible = False
    try:
        torn.replay()
    except TornRecord:
        torn_visible = True
    check("advancing hwm before flush exposes a partial record", torn_visible)

    return 1 if failures else 0


def main(argv: list[str]) -> int:
    if argv[1:] == ["--self-test"]:
        return self_test()
    print("usage: event_log_idempotence.py --self-test", file=sys.stderr)
    return 2


if __name__ == "__main__":
    sys.exit(main(sys.argv))
```

## What the tests pin

| Test | If it failed, the bug would be |
|---|---|
| Double replay, two ids | derived state is a bag; restart duplicates work |
| Crash before flush, only `evt-1` visible | readers use file length, not HWM; torn tails leak |
| Crash before HWM, only `evt-1` visible | visibility tracks flush, not the mark |
| Wrong-order HWM raises `TornRecord` | the ordering rule is load-bearing, not style |

The named rule in the script is **write → flush → high-water mark**. Readers expose `[0, hwm)` only.
