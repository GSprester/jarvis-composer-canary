# Declared vs derived audit

**Status:** design only.  
**Depends on:** `ARTIFACT-DERIVED-LEDGER.md`, `SPARSE-RE-JOIN.md`, `TASK-18-COUNT-RECONCILIATION.md`, `OUTWARD-JOB-AUDIT.md`.  
**Does not exist in this tree:** auditor, declared bag, evidence enumerator.

An audit that walks **only** what the declaring party listed (`D`) assumes `D` is complete. The declarer is a writer (`TASK-10-SELF-REPORT.md`): seats, issuers, READMEs, handoff dumps. They omit outward jobs, bare receipts, worker leftovers, and hidden hops (`OUTWARD-JOB-AUDIT.md`, `SPARSE-RE-JOIN.md` 494). Completeness of `D` is not a MEASURE.

The audit enumerates **derived evidence** `E` independently, then reconciles. Unmatched rows are first-class. They are not noise and not auto-declared.

## Two bags (independent looks)

```
D  =  declared work          # handoffs / WO: unit, path, expected
E  =  derive(artifacts)      # this process: files+digests, receipts, locks,
                             # ledger [0,hwm), outward registers, archives
```

`E` does **not** start from `D` and “fill in.” A glob of `D`’s paths will never see a file the declarer omitted. Enumerators are the same class as outward-job listing: completed look per store; canary-hit must appear; missing canary → ERROR, stop (`LEGACY-39.md`).

Failed look of `D` or `E` → whole audit **UNKNOWN**. Do not publish `unmatched=0` (`TASK-20-FAIL-LOUD.md`, `TASK-34`).

Identity of a row is `unit` / content digest / non-empty `enqueue_id` (`HANDOFF-RENAME-ID.md`, `SPARSE-RE-JOIN.md` R1–R3). Not `name[:8]`, not `exists`.

`derive` ignores worker `status` and `DONE` (`ARTIFACT-DERIVED-LEDGER.md`).

## Reconciliation

```
R = reconcile(keys(D), keys(E))     # TASK-18 bags, not len(D)==len(E)
```

Join uses the sparse-`re` ladder. `n≠1` → no pair (template / cartesian). Wrong `re` DISCARD, no digest fall-through.

| `R` | Name | Not |
|---|---|---|
| unchanged and field-equal (`expected` vs `sha256(locator)`) | **matched** | `exists` / worker complete |
| in `D` not in `E` | **declared_only** | “nothing to verify” / COMPLETED |
| in `E` not in `D` | **evidence_only** | “extra file, ignore” / auto-handoff |
| same key, different digest / `kind` | **conflict** | last-write / prettier path |
| look failed | **unknown** | empty bag |

`D` **wins** only as the **obligation** on a matched key (`expected` stays). `E` **wins** as the **view** of what happened (`verify`). The declarer cannot shrink `E` by omitting a line. The auditor cannot grow `D` by minting a handoff from `E` (`LEGACY-17.md`).

Canary-hit: a declared unit whose artifact derives.  
Canary-miss: **evidence with no row in `D`** (planted receipt / outward job / leftover digest). If the audit is clean, it trusted the declarer’s completeness.

Zero-acceptance: one dropped `evidence_only` or one `declared_only` marked COMPLETED fails the stratum.

## Treatment of unmatched rows

Every id from `D` and `E` lands in exactly one bag. `reconcile(|D|+|E|, bag keys)` is all unchanged.

### `declared_only`

- Status **UNVERIFIED**. Not COMPLETED, not “closed unused.”
- Write `unmatched/declared/{unit}` `{reason:no_derived_evidence, last_successful_scan}`.
- Keep the handoff. Do not delete so the next scan looks never-declared (`STALE-CLAIM-RECOVERY.md`).
- Re-run same `unit`. Do not mint a new id to “help.”

### `evidence_only`

This is the incompleteness of `D`. Shadow work: outward cron, bare receipt, payload without a handoff, ledger close `derive` can see that `D` never named.

- Status **undeclared**. Not COMPLETED (no `expected` on a handoff). Not ignored.
- Write `unmatched/evidence/{id}` `{digest, locator, enumerator, reason:not_in_declared}`.
- **Do not** invent a handoff with `expected := sha256(file)` so it matches (`LEGACY-17.md`, `MECHANICAL-ACCEPT-SPLIT.md`).
- **Do not** unlink the artifact to make `E` look like `D`.
- Escalate: issuer must declare (`unit`, `expected`, path) **or** quarantine/archive with provenance (`CAPABILITY-QUARANTINE.md`, `PROVENANCE-ARCHIVE.md`). Until then the row stays unmatched.
- `list_next` / dashboards show `n_evidence_only` as a gauge. Zero is publishable only with both canaries and `unknown==0`.

### `conflict`

Same `unit`, different bytes than `expected`. Artifact view is UNVERIFIED. Ledger does not rewrite `expected`. Two locators: conflict as in two-lane merge — do not pick mtime.

### `unknown`

Do not empty the unmatched dirs. Do not overwrite last good bag (`TASK-08`).

## What this must not do

- Walk only `D` and call leftover files out of scope.
- Treat `len(D)==len(E)` as reconciled.
- Auto-declare from `E` or auto-delete `E` to match `D`.
- Mark unmatched COMPLETED or drop it from the next publish.

**Rule:** Enumerate `E` without asking `D` what exists. Reconcile bags of `unit`. `declared_only` stays UNVERIFIED. `evidence_only` stays undeclared until the issuer writes a handoff or a provenance archive — never by trusting the declarer to have listed everything.
