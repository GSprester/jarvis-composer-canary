# TASK-14-SUMMARY

Index of `TASK-*.md` files observed in this checkout (`ls -1 /workspace/TASK-*.md`). One line each. A known slot with no file would be listed as NOT PRESENT; none of TASK-01–TASK-14 are missing after this file is written.

- `TASK-01-REPORT.md` — Observed inventory: README one-line description, top-level entries, and explicit non-existence of `cloud_dispatch.py`, `external_provider_policy.py`, and `scripts/handoff_lifecycle.py`.
- `TASK-02-POLICY-AUDIT.md` — Search for provider-policy and configuration-loading FAIL-OPEN paths; none exist in this repository.
- `TASK-03-REBOOT-SWEEP.md` — Proposal to rewrite pre-boot `claimed`/`running` job-store rows to terminal `reboot_interrupted`, with test cases and bounded-cap retention notes.
- `TASK-04-MATCHER-TESTS.md` — Property-based receipt-to-handoff matcher suite (symmetric, idempotent, `re` identity) that a naive date-prefix matcher fails, including UTC rollover.
- `TASK-05-DOCTRINE-DRAFT.md` — Resume-on-restart doctrine: no completion claim without a third-party-checkable artifact (else UNVERIFIED); same writer admission as cold start; wrapper records disposition.
- `TASK-06-ARTIFACT-CHECK.md` — Stdlib checker that exits 0 only for an in-repo file whose sha256 matches the expected digest, with self-tests for missing/empty/wrong/out-of-repo paths.
- `TASK-07-DATE-ROLLOVER.md` — Minimal repro of the UTC date-prefix miss (`20260910-*` handoff vs `20260911-*` receipt), failing vs corrected predicates, and same-day / +1 / −1 cases.
- `TASK-08-WATCHDOG-PATTERN.md` — Silent-watchdog reference: no output when healthy, one alert per state fingerprint, probe failure is UNKNOWN, last-known-good never overwritten on failed probe.
- `TASK-09-RETENTION-RISK.md` — Aged-out-before-review failure mode of a 2,048-row rolling table, with raise-the-cap vs monthly-archive cost numbers.
- `TASK-10-SELF-REPORT.md` — Why a worker-written completion record is weaker than a third-party artifact check, the over-claim failure mode, and COMPLETED as a checker-only view.
- `TASK-11-IDEMPOTENCE.md` — Tests that an append-only log replays without duplication and hides torn writes because the high-water mark advances only after flush.
- `TASK-12-POLICY-PARITY.md` — Checker that diffs two provider permission sets, a one-class worked example, and why name-keyed policy lets failover exceed a seat ceiling.
- `TASK-13-FAILOVER-CEILING.md` — Design that binds sensitivity to the seat; smallest gate is `admit(seat, provider)` in the wrapper failover selector.
- `TASK-14-SUMMARY.md` — This index: one sentence per `TASK-*.md` file present in the repository.
