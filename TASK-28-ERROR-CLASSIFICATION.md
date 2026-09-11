# TASK-28-ERROR-CLASSIFICATION

Map transport failures to a disposition: **retry now**, **retry with backoff**, **do not retry**, **escalate**. Timeout is UNKNOWN, not “no effect” (`TASK-27-RETRY-SEMANTICS.md`). This checkout has no HTTP client; the script is the classifier.

Bodies are scanned for a **quota exhaustion** token. That is a synthetic policy label, not an account or invoice.

## Usage

```text
python3 error_classification.py --self-test
```

Extract the script from the fence that starts `#!/usr/bin/env python3`.

## Why 429 and 402 differ

| Code | Meaning | Retry? |
|---|---|---|
| **429** Too Many Requests | This seat or provider is **rate-limited**. Capacity exists; the window is full. | **Retry with backoff** (and queue backpressure, `TASK-25-QUEUE-BACKPRESSURE.md`). Same send token; do not mint a new id. |
| **402** Payment Required | The provider **will not accept more work** until a human changes plan, billing, or entitlement. | **Do not retry.** **Escalate.** Backoff burns the queue and looks like a slow consumer outage. Failover must still `min(seat, provider)` (`TASK-23-POLICY-INHERITANCE.md`); a wider vendor is not a payment fix. |

A 429 that later becomes a **quota exhaustion** body is treated as 402-class (do not retry + escalate). Rate-limit and quota are not the same: one is time, the other is entitlement.

## Classifier

```python
#!/usr/bin/env python3
"""Classify transport errors into retry / backoff / no-retry / escalate.

Stdlib only. No network. No real account or payment fields.
"""

from __future__ import annotations

import sys
from typing import Any

QUOTA_MARKERS = ("quota exhaustion", "quota_exhausted", "quota exceeded")


def classify(status: int | None, body: str = "", error: str = "") -> dict[str, Any]:
    text = (body or "").lower()
    err = (error or "").lower()
    quota = any(m in text for m in QUOTA_MARKERS)

    if quota or status == 402:
        return {
            "disposition": "do_not_retry",
            "escalate": True,
            "reason": "quota_or_payment",
        }
    if status == 429:
        return {
            "disposition": "retry_with_backoff",
            "escalate": False,
            "reason": "rate_limited",
        }
    if status == 403:
        return {
            "disposition": "do_not_retry",
            "escalate": True,
            "reason": "forbidden",
        }
    if status == 500:
        return {
            "disposition": "retry_with_backoff",
            "escalate": False,
            "reason": "server_error",
        }
    if "connection reset" in err or err in {"econnreset", "connectionreseterror"}:
        return {
            "disposition": "retry_now",
            "escalate": False,
            "reason": "connection_reset",
        }
    raise ValueError("unclassified transport error")  # fail loud, not []


def self_test() -> int:
    failures = 0

    def check(name: str, ok: bool) -> None:
        nonlocal failures
        print(f"{'PASS' if ok else 'FAIL'}: {name}")
        if not ok:
            failures += 1

    r429 = classify(429)
    check("429 is backoff, not escalate", r429["disposition"] == "retry_with_backoff" and r429["escalate"] is False)

    r402 = classify(402)
    check("402 is do_not_retry", r402["disposition"] == "do_not_retry")
    check("402 escalates", r402["escalate"] is True)
    check("429 and 402 differ", r429["disposition"] != r402["disposition"])

    r500 = classify(500, body="internal error")
    check("500 is backoff", r500["disposition"] == "retry_with_backoff" and r500["escalate"] is False)

    r403 = classify(403)
    check("403 is do_not_retry+escalate", r403["disposition"] == "do_not_retry" and r403["escalate"] is True)

    rreset = classify(None, error="Connection reset")
    check("connection reset is retry now", rreset["disposition"] == "retry_now" and rreset["escalate"] is False)

    rquota = classify(500, body="QUOTA EXHAUSTION: unit cap reached")
    check("quota body is not treated as 500 backoff", rquota["disposition"] == "do_not_retry")
    check("quota body escalates", rquota["escalate"] is True)
    check("quota reason distinct", rquota["reason"] == "quota_or_payment")

    r429q = classify(429, body="quota exceeded")
    check("429 + quota body follows quota, not 429", r429q["disposition"] == "do_not_retry")

    return 1 if failures else 0


def main(argv: list[str]) -> int:
    if argv[1:] == ["--self-test"]:
        return self_test()
    print("usage: error_classification.py --self-test", file=sys.stderr)
    return 2


if __name__ == "__main__":
    sys.exit(main(sys.argv))
```

## Disposition table

| Input | Disposition | Escalate | Retry send? |
|---|---|---|---|
| 429, no quota body | retry with backoff | no | Same idempotency token only |
| 402 | do not retry | **yes** | No |
| 500, generic body | retry with backoff | no | Same token; not a new send id |
| 403 | do not retry | **yes** | No — seat/provider ceiling or admit failure |
| connection reset | retry **now** | no | Same token; likely dropped before accept |
| Body contains quota exhaustion (any status) | do not retry | **yes** | No — entitlement, not a blip |

Unclassified errors **raise**. They do not map to retry now (`TASK-20-FAIL-LOUD.md`).
