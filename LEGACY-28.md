# LEGACY-28

**A router is preferable** to N bespoke client wrappers for N models because the estate has **one** admit, **one** lock, **one** error class, and **one** completion view. N wrappers are N copies of those predicates. Copies drift. Drift is split-brain (`LEGACY-01.md`, `LEGACY-12.md`) and a reliable lie (`LEGACY-10.md`).

Each vendor SDK wants to own retry, 401, region, and “done.” One will map timeout to `[]` (`LEGACY-03.md`). One will treat `invalid_api_key` as a dead secret (`LEGACY-23.md`). One will hop to a wider `provider.class` and rewrite `seat.ceiling` (`TASK-13-FAILOVER-CEILING.md`). One will set `status=completed` because the HTTP body said so (`TASK-17-SEPARATION-OF-DUTIES.md`). A third party cannot re-run N private gates. Policy parity becomes a grep (`TASK-12-POLICY-PARITY.md`). Adding model N+1 adds a new way to fail open (`LEGACY-07.md`) and a new silent default file (`LEGACY-27.md`).

A router is the **smallest enforcement point** already named for failover (`TASK-13-FAILOVER-CEILING.md`): `admit(seat, provider)` on every hop, `effective = min(seat, provider)` (`TASK-23-POLICY-INHERITANCE.md`), one exemption log (`LEGACY-22.md`), one three-endpoint probe (`LEGACY-24.md`), one classifier (`TASK-28-ERROR-CLASSIFICATION.md`). The N clients shrink to **transports**: send declared bytes to an allow-listed URL. They do not parse completion, do not hold the lock, do not mint receipts. The worker still cannot write COMPLETED (`LEGACY-16.md`). The router may **read and refuse**. It must not author the digest it then accepts (`LEGACY-17.md`) and must not pre-process judgment for the downstream model (`LEGACY-20.md`).

N wrappers optimize the vendor’s object. The router optimizes the **unit**: one `re`, one `pid+start`, one `policy_sha256`. Failover changes who executes, not what the seat may handle. Observability is one wire type per disposition, not N vendor novels (`LEGACY-25.md`). Retirement is one tombstone, not N SDK majors (`LEGACY-15.md`).

Bespoke wrappers remain as **adapters under the router**, not as seats. If a wrapper can `admit`, `verify`, or skip the lock, it is a second lead. Prefer the router so those verbs exist once.

**Rule:** Fan-out of models is a table of providers. Fan-out of gates is an estate you cannot check.
