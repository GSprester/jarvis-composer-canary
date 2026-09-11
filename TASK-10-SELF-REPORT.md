# TASK-10-SELF-REPORT

This checkout has no worker runtime. The argument is the evidence rule behind `TASK-05-DOCTRINE-DRAFT.md` and `TASK-06-ARTIFACT-CHECK.md`.

## Why a worker-written completion record is weaker

A completion **record** is a row the worker can write: `status=completed`, “DONE:”, a receipt JSON it authored. A third-party **check** is someone else (or a later process with no seat memory) confirming that a declared path exists and its bytes hash to a declared digest (`TASK-06-ARTIFACT-CHECK.md`).

The record and the work share an author. The check and the work do not. After the worker exits, is killed, or is resumed as a new admission (`TASK-05-DOCTRINE-DRAFT.md`), only the bytes remain. A status field does not.

| Signal | Who writes it | Survives worker death | Independently re-runnable |
|---|---|---|---|
| Worker `status=completed` | the worker | as a claim only | no — it is the claim |
| File at path P with sha256 E | the worker may write bytes; **not** the verdict | yes, if the file is still there | yes — any third party runs the checker |

## Concrete examples

1. **Helix-7 changelog.** Worker writes `status=completed` and “published changelog-helix7.md.” A later seat runs the checker on `docs/synthetic/changelog-helix7.md` with the declared digest. Missing file, empty file, or wrong hash → UNVERIFIED. The row still says completed. The row is the weaker object.

2. **UTC rollover receipt (`TASK-07-DATE-ROLLOVER.md`).** Worker writes “matched handoff 20260910-alpha” because name prefixes looked close. A third party checks `receipt.re == handoff.name`. The self-report can be true in the worker’s frame and false under the identity predicate. The artifact (the receipt bytes and the `re` field) is what can be checked.

3. **Silent watchdog (`TASK-08-WATCHDOG-PATTERN.md`).** Worker writes `last_known_good` after a failed probe. That is a self-report of health. The pattern forbids it: probe failure is UNKNOWN and must not overwrite LKG. A third party can only trust LKG if it was written after a successful healthy probe, not because the worker said “done checking.”

## Over-claim failure mode

The worker emits a completion record **without** an artifact that a third party can check, or with an artifact that does not match the declared digest.

Forms:

- **Status-only:** `completed` / “DONE:” and no file.
- **Partial work:** one of three subtasks written; umbrella `completed` anyway.
- **Wrong bytes:** file exists, hash ≠ declared E (trimmed preview, stale pointer, or a different day’s receipt name).
- **Out-of-repo path:** worker points at `/tmp/...`; checker refuses; record still says completed.

Downstream treats the record as closed work. Resume, reboot sweep, and monthly audit (`TASK-09-RETENTION-RISK.md`) then have nothing to re-hash. The incident looks finished until someone looks for the file.

Discouragement (lint, “please attach a receipt,” honor-system `DONE:`) does not stop a model or a crashed wrapper from writing the bit.

## Design that makes over-claim structurally impossible

**The worker has no write path to `COMPLETED`.** That column is not a field. It is a view.

```
COMPLETED(job)  iff  checker(declared_path, expected_sha256) == 0
else UNVERIFIED
```

- Worker APIs: write bytes under the repo root; declare `(path, expected_sha256)`; write dispositions (`reboot_interrupted`, UNKNOWN). No `UPDATE jobs SET status='completed'`.
- The wrapper, a later seat, or a cron checker is the only process that evaluates the predicate. Same admission as a cold start.
- If the file is missing, empty-when-not-expected, wrong hash, or outside the repo, the view is UNVERIFIED even if the worker’s last log line said “done.”
- Over-claim is not a policy violation. There is no cell that can hold the over-claim.

That is stronger than telling the worker not to lie. The store cannot represent a completion the checker has not re-derived.
