# TASK-03-REBOOT-SWEEP

Proposal only. This checkout has no job-store implementation (`cloud_dispatch.py`, `external_provider_policy.py`, and `scripts/handoff_lifecycle.py` are absent). Do not land a store, a sweep runner, or tests here. The following is the contract a later implementation must satisfy.

## Goal

On host boot, **before any new claim is issued**, reconcile the job store against the current boot:

- **Match:** `status ∈ {claimed, running}` **AND** the row’s recorded process start **predates** the current host boot.
- **Write:** set `status = reboot_interrupted` (terminal). Do not resume, retry, re-claim, or verify in this sweep.

Rows that are already terminal, or whose recorded start is at or after this boot, stay unchanged.

`reboot_interrupted` is a **wrapper disposition**, not COMPLETED (`TASK-05-DOCTRINE-DRAFT.md`, `LEGACY-29.md`). It means the incarnation cannot exist on this boot. It does not mean the artifact verified.

## Recorded process start

The compared instant is the lock incarnation `start` (`LEGACY-08.md`): OS process creation time, UTC seconds, the same field used in `pid+start`. It is **not** row mtime, claim-acquired-at, or a local civil clock (`TASK-16-ONE-LEAD-LOCK.md`, `TASK-19-TIMESTAMP-HAZARD.md`).

`boot_at` is the current host boot timestamp, captured once at sweep start (Linux `/proc/stat` `btime`, or Windows `now - GetTickCount64`; see `LIVENESS-PREDICATE.md`). Compare in UTC.

## Predicate

Rewrite iff:

```
status in {claimed, running}
AND process_started_at is a well-formed UTC timestamp
AND process_started_at < boot_at
```

A second pass finds no matching live rows (idempotent).

### Missing or malformed start

If `status ∈ {claimed, running}` and `process_started_at` is missing, empty, unparseable, or not comparable to `boot_at`, treat the row as **interrupted**, not live:

- Rewrite to `reboot_interrupted`.
- Do not leave it `claimed`/`running` (fail-open: a stuck lock that blocks the unit forever; `LEGACY-07.md`).
- If the store has a reason field, record `missing_process_start` vs `pre_boot_process_start`. Status is still `reboot_interrupted`.

This is store-row fail-closed. It is not the live-OS probe: a timeout or `EACCES` against a pid is UNKNOWN and must not steal (`LIVENESS-PREDICATE.md`). A row that never recorded a start has no incarnation to defend.

### Failed `boot_at`

If `boot_at` itself cannot be read (no `btime`, tick-count error, unparseable), the sweep is **UNKNOWN**:

- Do not rewrite any row (a post-boot worker would look pre-boot if `boot_at` is guessed).
- Do not issue new claims until `boot_at` is known.
- Do not treat “could not read boot time” as “no interrupted work.”

## What the sweep must not do

- Resume or continue the same row in place (`STALE-CLAIM-RECOVERY.md`: leftover payload becomes the next attempt).
- Delete the row, the lock, or a hot receipt before archive.
- Treat `reboot_interrupted` as COMPLETED or as “never existed.”
- Use mtime, cmdline, or seat token as a substitute for `start < boot_at`.

Recovery after the sweep (archive receipt, same `unit`, new `pid+start`) is out of scope for this document.

## Proposed test cases

Fixture clock: `boot_at = 1_757_600_000` (UTC seconds). One sweep over a mixed store. Only the interrupted and malformed live rows change.

| Case | Fixture | Expected |
|---|---|---|
| **Interrupted** | `status=running` (also run with `claimed`), well-formed `process_started_at = 1_757_599_000` (`< boot_at`) | Row rewritten to `reboot_interrupted` (reason `pre_boot_process_start` if present). No other rows touched. |
| **Legitimately finished** | `status` already terminal (`succeeded` / `failed` / `reboot_interrupted` / equivalent), even when `process_started_at = 1_757_599_000` | Row unchanged. Sweep does not rewrite finished work. |
| **Still running after boot** | `status=claimed` or `running`, well-formed `process_started_at = 1_757_600_100` (`>= boot_at`) | Row unchanged. Post-boot workers are not interrupted. |
| **Malformed / missing start time** | `status=claimed` or `running`, `process_started_at` is `null`, `""`, or unparseable (`"yesterday"`, naive local string, non-numeric) | Row rewritten to `reboot_interrupted` (reason `missing_process_start` if present). Must not remain claimable as live work. |

Additional pins:

- Mixed store: one row of each case in a single sweep; only interrupted and malformed live rows change.
- Idempotence: a second sweep does not alter already-rewritten rows.
- Both `claimed` and `running` are in the match set. Other non-terminal statuses, if any, are out of scope unless added later.
- Equality: `process_started_at == boot_at` is **not** interrupted (treat as post-boot).
- `boot_at` UNKNOWN: none of the four live/malformed rows are rewritten; the sweep reports UNKNOWN.

## Retention implication of a bounded row cap

If the job store keeps at most **N** rows, this sweep **does not free a slot**. It converts a live row into a terminal `reboot_interrupted` row. That row still counts toward the cap (`TASK-09-RETENTION-RISK.md`).

Implications:

- Repeated boots accumulate `reboot_interrupted` history until the cap evicts older rows.
- Eviction drops the only record that a claim was interrupted rather than completed. Callers that treat “row gone” as “never existed” misread interrupted work (`TASK-20-FAIL-LOUD.md`).
- Oldest-first wrap can delete pre-boot interruption evidence while retaining newer post-boot claims. Acceptable for liveness, not for audit (`TASK-22-ROLLING-WINDOW-AUDIT.md`).
- After wrap, `open_units = 0` looks clean while the interrupted units were only evicted (`TASK-34-METRIC-THAT-LIES.md`).
- If the cap refuses inserts, a store already at cap stays at cap after the sweep; new claims still fail until a separate retention policy deletes or archives **terminal** rows.

Pair the sweep with an explicit retention rule: keep `reboot_interrupted` until cap pressure, then evict oldest **terminal** rows first; **never** evict live `claimed`/`running` to make room. Prefer archive-before-evict (`TASK-22-ROLLING-WINDOW-AUDIT.md`): copy, fsync, verify, then evict. Failed archive blocks eviction.
