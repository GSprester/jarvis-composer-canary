# LEGACY-29

**Harness retirement** removes the wrapper/router binary from every run path (`LEGACY-15.md`, `LEGACY-28.md`). The **seat** is not retired. A successor harness cold-starts the same seat into the same store (`TASK-05-DOCTRINE-DRAFT.md`). Facts that lived only in the old process, the old install prefix, or the old SDK are gone. In-seat memory must not come back as trust.

Locks (`pid+start`), LKG the job wrote, and `status=completed` **do not** survive as claims (`LEGACY-08.md`, `LEGACY-09.md`, `TASK-17-SEPARATION-OF-DUTIES.md`). New incarnation, new admit.

## Per-seat facts that must survive

| Fact | Why the successor needs it |
|---|---|
| `seat` token | Identity. Not a pid. Not a hostname. |
| `seat.ceiling` and the provider allow-list | Authorization is on the seat, not on whichever vendor last answered (`TASK-13-FAILOVER-CEILING.md`). Factory defaults after a package pull are a silent reset (`LEGACY-27.md`). |
| Open / UNVERIFIED units: `unit` / `re`, declared `(path, expected_sha256)` | Work is not done. Resume must not invent a new id or a new digest (`TASK-24-ARTIFACT-DECLARATION.md`, `TASK-27-RETRY-SEMANTICS.md`). |
| Unexpired exemption grants | An override is not `seat.ceiling = provider.class`. The new gate must recompute `would_deny` (`LEGACY-22.md`). |
| In-flight dedupe keys (content hash / send token) | Timeout + resend must not double-deliver (`TASK-26-DEDUPE-BY-CONTENT.md`). |
| `key_id` → declared region bind | So `unknown_key` vs `region_mismatch` remains decidable (`LEGACY-23.md`, `LEGACY-24.md`). **Not** the secret bytes. |
| Last intentional config digest + `config.gen` | Distinguishes operator edit from harness reinstall defaults (`LEGACY-27.md`). |
| Wrapper dispositions (`reboot_interrupted`, refuse, timeout) | Operational history for the unit. Not COMPLETED (`TASK-05-DOCTRINE-DRAFT.md`). |
| Receipts / artifacts that still `verify` | The only completions that survive (`TASK-06-ARTIFACT-CHECK.md`). Archive before the old tree is evicted (`TASK-22-ROLLING-WINDOW-AUDIT.md`). |

## Where they should live

All of it **outside** the harness install and **outside** model memory. The retired binary may be deleted. The packaged template may return to defaults.

- **Seat record** (token, ceiling, allow-list, `key_id` names): durable store keyed by `seat`, path not under the wrapper prefix. Canonical JSON, add-only (`TASK-33-SCHEMA-EVOLUTION.md`). Successor reads this on admit; it does not read the old env.
- **Secrets:** secret manager only, referenced by `key_id`. Never in the seat record, never in receipts, never in the tombstone (`LEGACY-24.md`).
- **Units and declarations:** the **unit root** (handoff body + declared artifact paths). Matcher identity is `re`, not the old harness job name (`TASK-07-DATE-ROLLOVER.md`).
- **Grants, dispositions, dedupe:** append-only JSONL (flush then HWM, `TASK-11-IDEMPOTENCE.md`), one stream each, `seat` on every row. Not appended onto the unit artifact (`LEGACY-21.md`).
- **Config intent:** sidecar `intentional.sha256` + `config.gen` next to the **live** policy file, plus `packaged.sha256` declared at image build — not inside the template the installer overwrites.
- **Completions:** hash-addressed bytes at the declared path; digest on the handoff, not in the worker. Monthly archive if the hot tree will vanish.
- **Tombstone:** successor module name only (`LEGACY-15.md`). It points at the store. It does not carry ceiling, grants, or secrets.

**Rule:** If the successor cannot re-derive the fact without the old harness, it was not a surviving seat fact. It was a session. Admit as cold start; verify what is still on disk; UNVERIFIED for the rest.
