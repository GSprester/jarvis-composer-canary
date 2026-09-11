# LEGACY-24

Probe **three declared** regional endpoints for one credential so a third party can tell `unknown_key` from `region_mismatch` (`LEGACY-23.md`) without writing the secret into the estate. The wrapper holds the bytes (`TASK-05-DOCTRINE-DRAFT.md`). The seat, the model, and the receipt body never see them. Candidates come from the policy document (`policy_sha256`), not from failover invention (`TASK-13-FAILOVER-CEILING.md`). `admit` still `min(seat, provider)` (`TASK-23-POLICY-INHERITANCE.md`). Do not probe a wider class to “find a working region.”

## Do not leak the credential

- Send it only as an **Authorization** (or vendor-equivalent) header on TLS to an **allow-listed** host. Never query string, path, filename, or `re`.
- Logs, exceptions, watchdog lines, and exemption rows record `key_id`, `provider`, endpoint **name**, status class — not the secret, not the header (`LEGACY-22.md`). Redact `Authorization` before any serialize. Do not append the key onto a hash-addressed artifact (`LEGACY-21.md`).
- One wrapper incarnation (`pid+start`, `LEGACY-08.md`). No second clone “just to try eu-west” (`LEGACY-01.md`).
- Healthy probes print nothing (`TASK-08-WATCHDOG-PATTERN.md`).

## Do not call one 401 a dead key

Each candidate is its own completed check. Identity is `(key_id, endpoint)`, not `status != 200`.

| Per-endpoint outcome | Class |
|---|---|
| Authn accepted | `bound` |
| Completed 401 / `invalid_api_key` / 403-as-unknown | `denied_here` |
| 429 | `rate_limited` (not invalid, `TASK-28-ERROR-CLASSIFICATION.md`) |
| 402 / quota | `quota_or_payment` (not invalid) |
| Timeout, reset, unreadable, unclassified | `UNKNOWN` (raise; not `denied_here`, `LEGACY-03.md`, `LEGACY-07.md`) |

Aggregate **only after** three completed rows (or a `bound`). Missing a look is not a deny.

```
unknown_key       iff  all three completed AND none bound AND none rate_limited/quota
region_mismatch   iff  at least one bound AND at least one denied_here
ambiguous         iff  any UNKNOWN remains
split_bind        iff  two or more bound
```

Rotate or escalate “dead key” **only** for `unknown_key`. For `region_mismatch`, bind the declared endpoint that accepted, still under the seat ceiling. For `ambiguous`, stay UNVERIFIED; do not rotate (`TASK-08-WATCHDOG-PATTERN.md`: do not overwrite LKG on a failed probe). For `split_bind`, fail closed — two regions accepted; do not pick `max`.

A first-hop `denied_here` must not short-circuit the other two. That is how a valid foreign-region key becomes a dashboard “no key” (`TASK-34-METRIC-THAT-LIES.md`). The probe log is the three structured rows plus `policy_sha256`. A third party re-derives the aggregate from those rows, not from a wrapper sentence (`LEGACY-06.md`).
