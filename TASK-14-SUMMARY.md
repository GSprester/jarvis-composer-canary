# TASK-14-SUMMARY

Index of `TASK-*.md` files observed in this checkout (`ls -1 /workspace/TASK-*.md`). One sentence each. A known slot with no file is listed as NOT PRESENT rather than omitted. Observed range is TASK-01–TASK-34; **no slot is missing**.

- `TASK-01-REPORT.md` — Observed inventory: README one-line description, top-level listing, and explicit non-existence of `cloud_dispatch.py`, `external_provider_policy.py`, and `scripts/handoff_lifecycle.py`.
- `TASK-02-POLICY-AUDIT.md` — Audit of extractable policy/config fences for fail-open returns; no standalone policy loader or live `POLICY.json` exists here.
- `TASK-03-REBOOT-SWEEP.md` — Proposal to rewrite pre-boot `claimed`/`running` job-store rows to terminal `reboot_interrupted`, with four test cases and bounded-cap retention notes.
- `TASK-04-MATCHER-TESTS.md` — Property-based receipt-to-handoff matcher suite (symmetric, idempotent, `re` identity) that a naive date-prefix matcher fails, including UTC rollover.
- `TASK-05-DOCTRINE-DRAFT.md` — Resume-on-restart doctrine: no completion claim without a third-party-checkable artifact (else UNVERIFIED); same writer admission as a cold start; wrapper records disposition.
- `TASK-06-ARTIFACT-CHECK.md` — Stdlib checker that exits 0 only for an in-repo file whose sha256 matches the expected digest, with self-tests for missing, empty, wrong-hash, and out-of-repo paths.
- `TASK-07-DATE-ROLLOVER.md` — Minimal repro of the UTC date-prefix miss (`20260910-*` handoff vs `20260911-*` receipt), failing vs corrected predicates, and same-day / +1 / −1 cases.
- `TASK-08-WATCHDOG-PATTERN.md` — Silent-watchdog reference: no output when healthy, one alert per state fingerprint, probe failure is UNKNOWN, last-known-good never overwritten on a failed probe.
- `TASK-09-RETENTION-RISK.md` — Aged-out-before-review failure mode of a 2,048-row rolling table, with raise-the-cap vs monthly-archive cost numbers.
- `TASK-10-SELF-REPORT.md` — Why a worker-written completion record is weaker than a third-party artifact check, the over-claim failure mode, and COMPLETED as a checker-only view.
- `TASK-11-IDEMPOTENCE.md` — Tests that an append-only log replays without duplication and hides torn writes because the high-water mark advances only after flush.
- `TASK-12-POLICY-PARITY.md` — Checker that diffs two provider permission sets, a one-class worked example, and why name-keyed policy lets failover exceed a seat ceiling.
- `TASK-13-FAILOVER-CEILING.md` — Design that binds sensitivity to the seat; smallest gate is `admit(seat, provider)` in the wrapper failover selector.
- `TASK-14-SUMMARY.md` — This index: one sentence per `TASK-*.md` file present in the repository.
- `TASK-15-IDEMPOTENT-RECEIPT.md` — Contract that the same handoff id and outcome must produce byte-identical receipt files (no clocks or session ids in the body).
- `TASK-16-ONE-LEAD-LOCK.md` — Proposal for a repo-level writer lead: at most one seat mutates a unit, identity is `pid+start`, path is not a UTC date prefix.
- `TASK-17-SEPARATION-OF-DUTIES.md` — Tests that a worker cannot mark its own work COMPLETED; the expected digest lives on the handoff, not on the worker’s record.
- `TASK-18-COUNT-RECONCILIATION.md` — Bag reconciler that reports added, removed, and unchanged by exact string identity, ignoring order and preserving duplicates.
- `TASK-19-TIMESTAMP-HAZARD.md` — Hazard of mixing local and UTC civil dates in names and keys, with a 23:30 PDT / next-UTC-day worked example.
- `TASK-20-FAIL-LOUD.md` — Rule that a failed search must raise or return a distinct error, not `[]`, so “found nothing” and “could not look” stay distinguishable.
- `TASK-21-WATCHDOG-SILENCE.md` — Three-way health probe (healthy / unhealthy / UNKNOWN) where timeout, raise, or a missing callable is UNKNOWN, never healthy.
- `TASK-22-ROLLING-WINDOW-AUDIT.md` — How a fixed cap retains a fraction of a day as write rate grows, when an incident becomes unreconstructable, and archive-before-evict costs.
- `TASK-23-POLICY-INHERITANCE.md` — Sensitivity resolver where effective class is min(seat, provider), never max, and unknown class raises.
- `TASK-24-ARTIFACT-DECLARATION.md` — Declare `(path, expected_sha256)` before write; the caller verifies, so empty or stale leftovers fail unless they match the declaration.
- `TASK-25-QUEUE-BACKPRESSURE.md` — Design for a bounded unit queue between seats and shells that refuses enqueue when full instead of dropping or expanding without bound.
- `TASK-26-DEDUPE-BY-CONTENT.md` — Dedup by canonical content hash, not by id: same body collapses, same id with different body does not.
- `TASK-27-RETRY-SEMANTICS.md` — Which operations may be retried after timeout or UNKNOWN without creating a second effect the store treats as new work.
- `TASK-28-ERROR-CLASSIFICATION.md` — Maps transport failures to retry-now, backoff, do-not-retry, or escalate; timeout is UNKNOWN, not “no effect.”
- `TASK-29-CONFIG-PRECEDENCE.md` — Config resolver: argument > environment > file > default, where an empty string is set and must not fall through.
- `TASK-30-LOG-ROTATION.md` — Rotator that keeps the last N archives, never deletes the open file, and names generations monotonically rather than by wall clock.
- `TASK-31-GRACEFUL-SHUTDOWN.md` — Shutdown design that drains in-flight work to a durable verify and must not claim COMPLETED before the artifact is flushed.
- `TASK-32-PATH-SAFETY.md` — Lexical path validator that rejects traversal, absolute escape, and Windows reserved names, checking the declared string before resolve.
- `TASK-33-SCHEMA-EVOLUTION.md` — Append-only event schema that adds fields without breaking old readers; writers never remove a shipped field.
- `TASK-34-METRIC-THAT-LIES.md` — Why a zero count (`open_units`, unmatched, alerts) can look healthy after wrap, failed search, or a quiet watchdog.

NOT PRESENT: none in TASK-01–TASK-34.
