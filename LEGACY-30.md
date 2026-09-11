# LEGACY-30

**Down** means a **completed try** against the declared route failed in transport or at that host (connect refused, reset after connect, completed 5xx). **Never tried** means the router did not send: no `admit`, `queue_full`, ceiling deny, harness skip, or the test never called the hop (`TASK-13-FAILOVER-CEILING.md`, `TASK-25-QUEUE-BACKPRESSURE.md`). They share a dashboard zero (`TASK-34-METRIC-THAT-LIES.md`). They are empty vs failed search (`LEGACY-03.md`) for a path instead of a query.

A missing error is not “never.” A `401` is not “down” (`LEGACY-23.md`). A torn timeout after SYN is **truncated** (`LEGACY-26.md`) — UNKNOWN, not down.

## Instrument

Identity is `(probe_id, route, policy_sha256)`. `probe_id` is the test’s send token (same id on retry, `TASK-27-RETRY-SEMANTICS.md`). `route` is a **declared** provider endpoint, not a failover invention (`LEGACY-28.md`).

The router writes **two** append-only rows (`TASK-11-IDEMPOTENCE.md`). Extra keys ignored.

1. **`route_intent`** — flush **before** connect. Keys: `v`, `type=route_intent`, `probe_id`, `unit`, `seat`, `route`, `policy_sha256`, `pid`, `start`. No secret (`LEGACY-24.md`).
2. **`route_result`** — after a **completed** outcome, or not at all. Keys: same identity plus `class` ∈ `bound` | `down` | `denied_here` | `pre_send_refuse` | `UNKNOWN`.

`pre_send_refuse` (`ceiling`, `queue_full`, `admit_false`) is written **instead of** intent. It is never-tried with a reason. The gate must not invent a result so the test passes (`LEGACY-17.md`).

```
down          iff  intent flushed
                   AND result.class == down
                   AND result is a completed look
never_tried   iff  no intent for (probe_id, route)
                   AND (pre_send_refuse OR the test never invoked the router)
UNKNOWN       iff  intent without result
                   OR the attempt log look failed
                   OR result.class == UNKNOWN
```

A third party re-derives the label from the log + `policy_sha256`. A wrapper sentence “route down” is a self-report (`LEGACY-06.md`). Failed read of the log is UNKNOWN, not never-tried.

## The test (three fixtures, one `probe_id` each)

Use allow-listed stub hosts. Do not probe the live estate. Observer is not a route (`LEGACY-13.md`).

| Fixture | What the stub / gate does | Required rows | Pass label |
|---|---|---|---|
| A | Port closed or RST after accept | `route_intent` + `route_result.class=down` | `down` |
| B | `admit` false, `queue_full`, or ceiling | `pre_send_refuse` only — **no** intent | `never_tried` |
| C | Kill the router after intent flush, before result | `route_intent` only | `UNKNOWN` |

Also fail the build if:

- Fixture A yields `never_tried` or `invalid_api_key` (wrong predicate, `LEGACY-10.md`).
- Fixture B yields `down` (the hop was blamed for a gate refuse).
- Fixture C yields `down` or `never_tried` (truncated promoted, `LEGACY-25.md`).
- Any fixture hashes or logs the credential.
- The test writes `route_result` itself when the router did not.

**Rule:** Tried = flushed intent. Down = completed failure **after** that intent. Never = no intent. Anything else is UNKNOWN. The test asserts those three wires, not a single “unreachable” bit.
