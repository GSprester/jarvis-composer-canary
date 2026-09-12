# approval token TTL

A human-in-the-loop **approval token** is the live face of an exemption grant (`LEGACY-22.md`): `grant_id`, `unit`, `grantor`, `would_deny`, `expires_at_utc`. The wrapper checks it on MUTATE / failover / crash-loop resume. This checkout has no token issuer. The design is how to choose **TTL**, argued from the **work cycle**, not from a security default.

```
TTL  =  expires_at_utc − granted_at_utc     # UTC instants only (TASK-19)
```

`null` expiry is not a TTL. It is standing policy and needs a different review path. Do not use it to paper over a short TTL.

## Work cycle, not threat theatre

The token exists so a **human decision** covers a **bounded stretch of work** on one `unit`: grant → (optional backoff) → ACQUIRE → MUTATE → durable `verify` or wrapper disposition → lock release. That stretch is the cycle. The TTL must **outlive the cycle it authorizes** and **die before the next unrelated cycle**.

Stated estate durations (not measurements here):

| Segment | Bound | Source |
|---|---|---|
| Drain to durable verify | **T_drain = 30 s** | `TASK-31-GRACEFUL-SHUTDOWN.md` |
| Crash-loop resume backoff floor | **T_back = 300 s** | `CRASH-LOOP-BRAKE.md` |
| One shell / model hop | **T_hop** (p95 of claim→verify for that unit class) | seat runtime; if unknown, treat as UNKNOWN, do not invent 15 m |
| Human re-attend (same grantor, same shift) | **T_human ≥ 8 h** if the grantor may leave the desk mid-cycle | substitute cover; `P` silent days (`SUBSTITUTE-REVIEW-QUEUE.md`) |
| Weekend look | **L = 2 d** | `TASK-09-RETENTION-RISK.md` — audit lag, **not** a live TTL |
| Monthly review | **30 d** | audit. A 30-day live token is standing policy |

```
T_cycle  =  T_back + T_hop + T_drain
TTL      =  min(TTL_cap, max(T_cycle + T_slack, T_min))
```

- **T_slack** is one chance to finish the same incarnation after a page (same `pid+start`, same `unit`), not a second unit.
- **T_min** is at least `T_back + T_drain` (330 s) so a legal crash-loop resume cannot expire during the mandated wait.
- **TTL_cap** for a live MUTATE grant is **one UTC working day (24 h)**, not 30 days. Weekend/month numbers are for **archive review**, not for a token that still opens a hop.
- If `T_hop` is UNKNOWN, do not pick 15 minutes “because security.” Measure p95(grant→verify) on completed cycles, or bind the token to **one claim incarnation** (below) instead of guessing a wall clock.

Security still binds **scope** (`unit`, predicate, `seat`, `would_deny` recomputed, wrapper-written row). Shortening TTL is not a substitute for that bind. A 15-minute token on a four-hour unit does not reduce blast radius; it increases grant rate.

## Bind to the cycle

The token authorizes **one** of:

1. **One hop** — one `admit` / one `lock_publish` / one crash-loop attempt. TTL ≥ `T_back + T_hop + T_drain`.
2. **One claim incarnation** — `{unit, pid, start}` from ACQUIRE until `verify` or disposition. Wall-clock TTL is a cap, not the identity. A new `pid+start` needs a new look at `would_deny`, same `grant_id` if the predicate is unchanged (`TASK-27-RETRY-SEMANTICS.md`: do not mint a new id to “help”).

Expired ⇒ REFUSE new MUTATE. In-flight MEASURE (`verify`, `find_*`) still runs (`GATE-DEADLOCK.md`). Disposition `grant_expired`. Do not treat expiry as “never granted” (`TASK-20-FAIL-LOUD.md`). Do not steal the lock because the clock moved (`LIVENESS-PREDICATE.md`).

## Failure mode: TTL shorter than the task → keepalive habit

**Name:** mid-cycle expire trains a **keepalive habit**. The human leaves the loop and the token stays.

If `TTL < T_cycle` the grant dies while the same `unit` is still claimed. The next MUTATE refuses. The grantor is paged again for the **same** `would_deny`, same `policy_sha256`, same bytes. After a few cycles they stop MEASURE-ing the deny:

| Habit | What it does | Why it is dangerous |
|---|---|---|
| Rubber-stamp re-grant | Approve without rereading `would_deny` | Grantor is a heartbeat, not a decision (`TASK-10-SELF-REPORT.md`) |
| New `grant_id` per page | UUID per retry | Looks like many human looks; one tired click (`ADVERSARIAL-QUOTA-RETRY.md` §25) |
| `expires_at_utc = null` | Standing override | Ceiling rewrite by another name (`TASK-13-FAILOVER-CEILING.md`) |
| Widen scope to survive the clock | One token for all units / all verbs | Compensates TTL with blast radius |
| Shared break-glass / env token | Model or next seat refreshes | Gate no longer has a human (`LEGACY-17.md`) |
| Disable `H` / “just run the shell” | Unstick the third expire | One-writer and admit are gone (`GATE-DEADLOCK.md`) |

The tell: grant rate ≈ 1/TTL on the same `unit` while `verify` is still UNVERIFIED. That is not vigilance. It is a watchdog fingerprint you trained the operator to emit (`TASK-08-WATCHDOG-PATTERN.md`). Downstream, `open_units` can look handled because grants exist (`TASK-34-METRIC-THAT-LIES.md`).

**Break:** set TTL from `T_cycle`, not from a policy-doc “15m.” Detect `n≥3` same `{grantor, unit, predicate}` inside `T_cycle` with no change in recomputed `would_deny` → do **not** auto-extend; escalate (same shape as crash-loop DESIGN). Re-grant keeps `grant_id`. Wrapper writes the row. Chat ACK is not a token.

## What this does not do

- It does not make expiry a completion claim.
- It does not let the model write `expires_at_utc`.
- It does not use local civil midnight as a TTL boundary.
- It does not treat a 30-day audit window as a live grant.

**Rule:** TTL ≥ the work cycle it covers, and ≤ one UTC day for live MUTATE. If the clock is shorter than the task, the operator will keep the token alive and stop being the loop.
