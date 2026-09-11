# TASK-03-REBOOT-SWEEP

Proposal only. This checkout has no job-store implementation (`cloud_dispatch.py`, `external_provider_policy.py`, and `scripts/handoff_lifecycle.py` are absent). The following is a design for boot-time reconciliation, not code to land here.

## Goal

On host boot, before any new claim is issued, rewrite every job-store row that is still live on a process that cannot exist after this boot:

- **Match:** `status ∈ {claimed, running}` AND recorded process start time **predates** the current host boot time.
- **Write:** set `status = reboot_interrupted` (terminal). Do not resume, retry, or re-claim in this sweep.

Rows that are already terminal, or whose process start is at or after the current boot, are left unchanged.

## Predicate

Let `boot_at` be the current host boot timestamp (monotonic wall clock from the host, captured once at sweep start).

Let `process_started_at` be the row’s recorded process start.

Rewrite iff:

```
status in {claimed, running}
AND process_started_at is a well-formed timestamp
AND process_started_at < boot_at
```

The sweep is idempotent: a second pass finds no matching live rows.

## Missing or malformed start time

If `status ∈ {claimed, running}` and `process_started_at` is missing, unparseable, or not comparable to `boot_at`, treat the row as **interrupted**, not live:

- Rewrite to `reboot_interrupted`.
- Do not leave the row claimed/running (that would be fail-open: a stuck lock that blocks the job forever).
- Record a distinct reason on the row if the store has a reason field (for example `missing_process_start` vs `pre_boot_process_start`). Status is still `reboot_interrupted`.

## Proposed test cases

| Case | Fixture | Expected |
|---|---|---|
| Interrupted | `status=running` (or `claimed`), well-formed `process_started_at < boot_at` | Row rewritten to `reboot_interrupted`. No other rows touched. |
| Legitimately finished | `status` already terminal (`succeeded` / `failed` / `reboot_interrupted` / equivalent), even when `process_started_at < boot_at` | Row unchanged. Sweep does not rewrite finished work. |
| Still running after boot | `status=claimed` or `running`, well-formed `process_started_at >= boot_at` | Row unchanged. Post-boot workers are not interrupted. |
| Malformed / missing start time | `status=claimed` or `running`, `process_started_at` null, empty, or unparseable | Row rewritten to `reboot_interrupted`. Must not remain claimable as live work. |

Additional checks the tests should pin:

- Mixed store: one row of each case above in a single sweep; only interrupted and malformed live rows change.
- Idempotence: running the sweep twice does not alter already-rewritten rows.
- `claimed` and `running` are both in the match set; other non-terminal statuses, if any, are out of scope unless added later.

## Retention implication of a bounded row cap

If the job store keeps at most N rows, this sweep **does not free a slot**. It converts a live row into a terminal `reboot_interrupted` row. That row still counts toward the cap.

Implications:

- Repeated boots can accumulate `reboot_interrupted` history until the cap evicts older rows.
- Eviction of those rows drops the only record that a claim was interrupted rather than completed. Callers that treat “row gone” as “never existed” will misread interrupted work.
- A cap that evicts oldest-first may delete pre-boot interruption evidence while retaining newer post-boot claims, which is acceptable for liveness but not for audit.
- If the cap is enforced by refusing inserts, a store already at cap remains at cap after the sweep; new claims still fail until a separate retention policy deletes or archives terminal rows.

Any implementation should pair this sweep with an explicit retention rule for `reboot_interrupted` (for example: keep until cap pressure, then evict oldest terminal rows first, never evict live `claimed`/`running` rows to make room).
