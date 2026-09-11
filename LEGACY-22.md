# LEGACY-22

An **exemption** is an admit the published policy would have **refused**: `admit(seat, provider)` false, `effective` unknown/missing, or a hop that would raise the seat (`TASK-13-FAILOVER-CEILING.md`, `TASK-23-POLICY-INHERITANCE.md`). The gate may still let the call through. That override is not a comment and not a rewrite of `seat.ceiling` (`LEGACY-17.md`). It is a **row** in an append-only log (`TASK-11-IDEMPOTENCE.md`, `TASK-33-SCHEMA-EVOLUTION.md`). Do not append the grant onto the policy document or the unit artifact (`LEGACY-21.md`).

The wrapper writes the row (**not** the seat that benefits, `LEGACY-06.md`, `LEGACY-13.md`). Append bytes → flush → then hop. A grant that exists only in memory after the provider was called is a completion-before-durable (`TASK-31-GRACEFUL-SHUTDOWN.md`). Denies are dispositions elsewhere; this log is **grants only**, and every grant, not a sampled subset. A failed look at the log is UNKNOWN, not “no exemptions” (`LEGACY-03.md`).

## Schema (JSONL; one object per grant)

Canonical UTF-8 JSON, `sort_keys`, tight separators, one trailing newline (`TASK-15-IDEMPOTENT-RECEIPT.md`). Readers require `v >= 1` and all keys below; extra keys ignored. Writers never drop a shipped key.

| Key | Type | Meaning |
|---|---|---|
| `v` | number | Format version. |
| `type` | string | Always `exemption_grant`. |
| `unit` | string | Handoff / lock unit (`re`), not a local date name. |
| `grant_id` | string | Opaque id for **this** override. Stable across retry; not a new unit. |
| `granted_at_utc` | string | UTC timestamp `YYYY-MM-DDTHH:MM:SSZ` only (`TASK-19-TIMESTAMP-HAZARD.md`). Display and order; **not** identity. |
| `expires_at_utc` | string \| null | When the override ends. `null` = until tombstone. |
| `seat` | string | Seat token that was admitted. |
| `pid` | number | Wrapper incarnation that granted (`LEGACY-08.md`). |
| `start` | number | OS start of that pid, UTC seconds. |
| `predicate` | string | `admit` \| `effective` \| `failover`. The check that would have denied. |
| `seat_class` | string | Seat ceiling **before** the hop. Never rewritten by the grant. |
| `provider` | string | Destination name. |
| `provider_class` | string | Vendor advertisement at grant time. |
| `would_deny` | boolean | Must be `true`. If the predicate would have passed, this is not an exemption — do not write the row. |
| `deny_reason` | string | `ceiling` \| `unknown_class` \| `missing_provider` \| `missing_class`. |
| `effective_without` | string \| null | `min(seat, provider)` if both classes known; else `null` (must have raised). |
| `policy_sha256` | string | Digest of the **declared** policy bytes used for the check. |
| `grantor` | string | Human or break-glass token. Not the model. Not `seat`. |

```json
{"deny_reason":"ceiling","effective_without":"internal","expires_at_utc":null,"grant_id":"xg-1","granted_at_utc":"2026-09-11T194100Z","grantor":"oncall-a","pid":4412,"policy_sha256":"…64 hex…","predicate":"admit","provider":"beta","provider_class":"restricted","seat":"seat-a","seat_class":"internal","start":1757600001.25,"type":"exemption_grant","unit":"20260910-alpha","v":1,"would_deny":true}
```

Content-hash dedupe (`TASK-26-DEDUPE-BY-CONTENT.md`) keys on the canonical object **minus** `granted_at_utc`. Same `grant_id` + same predicate inputs = one grant.

## Question the log must answer

**Without this row, would the published policy (`policy_sha256`) have denied this `(seat, seat_class, provider, provider_class)` on `predicate` — and which wrapper incarnation (`pid+start`) and `grantor` overrode that deny, for which `unit`, until when?**

A third party must **recompute** `would_deny` from the structured fields plus the policy bytes. If the only way to know it was an exemption is to believe `type` or a prose note, the log is a self-report, not a check. Missing policy bytes or unknown class → the row is UNVERIFIED; do not treat the hop as a normal admit.
