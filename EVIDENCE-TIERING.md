# evidence tiering

Every verdict row carries `tier ∈ {deterministic, inferred, model_guess}` plus `kind` (`FOUND`/`NONE`/`ERROR` or `COMPLETED`/`UNVERIFIED`). `tier` is set by **how the bit was produced**, not by confidence prose. The model must not write `tier` (`TASK-17-SEPARATION-OF-DUTIES.md` — **fail:** it will stamp `deterministic`). **Fail if `tier` is omitted ⇒ deterministic:** empty looks get the strongest label (`POSITIVE-CONTROL.md`).

## Tiers

| Tier | Producer | Re-run without the author? |
|---|---|---|
| `deterministic` | Checker / OS / declared-field equality | Yes |
| `inferred` | Completed look, exact but **indirect** (one safe ladder rung, last MEASURE) | Yes, same inputs; **not** A∧B |
| `model_guess` | Seat/model/worker sentence, fuzzy, pre-process | No |

**Fail if you add `likely` / `degraded`:** callers treat them as FOUND (`LEGACY-38.md`).

### `deterministic`

Produced only by MEASURE that a third party can repeat on declared bytes or the kernel (`LEGACY-06.md`):

- A∧B: `sha256(path)==expected` **and** checker receipt `re==unit` (`LEGACY-32.md`)
- R1: `receipt.re==wo.unit` with `n_wo=n_receipt=1` (`RECEIPT-MATCHING.md`)
- `same_process(pid,start)` True/False after a completed probe (`LIVENESS-PREDICATE.md`)
- `admit` = `min(seat,provider)` from declared classes (`TASK-23-POLICY-INHERITANCE.md`)
- Canary-hit visible **and** canary-miss absent (`LEGACY-39.md`)
- `order_key` from body `priority`+`scheduled_utc` (`LEGACY-36.md`)

**Fail if `exists(path)` or HTTP 200 is stored as deterministic.** **Fail if `same` UNKNOWN is stored as deterministic False** (steal). **Fail if hash-before-canonicalise** (`TASK-15-IDEMPOTENT-RECEIPT.md`). ERROR (timeout, torn JSON) is **not** a deterministic NONE.

### `inferred`

Produced by a **completed** exact look that is not A∧B: R2 (digest+declared path, `re` empty), R3 (`enqueue_id`), `usable` from last quota MEASURE (`BUDGET-AWARE-ROUTING.md`), `stale` via `start<boot_at`, typed `NONE` after both controls, `review_opened` without close (`SUBSTITUTE-REVIEW-QUEUE.md`).

**Fail if inferred is used to `released` or set COMPLETED** (`IDEMPOTENT-EVENT-LEDGER.md`). **Fail if inferred from a failed look (`[]` / empty stdout).** **Fail if R2 with `n≠1`:** ambiguous is ERROR, not inferred FOUND. May **display** and may **queue**; may not close the unit.

### `model_guess`

Produced by the seat: `status=completed`, “covered,” prefix/fuzzy join, LKG the job wrote, weak pre-process (`LEGACY-20.md`), operator ACK. Wrapper may **record** it as a disposition (`TASK-05-DOCTRINE-DRAFT.md`). **Fail if it is the launch predicate.** DISCARD as peer analysis (`LEGACY-19.md`).

## Model must not override deterministic

Compare on the **same `ask`** (`unit`, path, expected, `enqueue_id`):

```
if det is ERROR: verdict = ERROR          # do not guess
if det.kind disagrees with guess.kind: verdict = det
if det is UNVERIFIED/NONE and guess is FOUND/COMPLETED: keep det
guess may annotate; it must not write kind, tier, or receipt
```

**Why:** the guess is the command trying to satisfy the gate (`LEGACY-17.md`). A third party cannot re-hash a sentence. Override is how unverified is read as settled (`LEGACY-31.md`) and how `C` drifts from `G` (`DEPENDENCY-INVERSION.md`). **Fail if “the strong model reviewed the digest and says OK” after `verify` failed:** that is override. **Fail if inferred FOUND overrides deterministic NONE** (canary-miss / wrong `re`). **Fail if a later seat inherits the guess on resume** (`TASK-05-DOCTRINE-DRAFT.md`).

Cut-over (`G.measure`): only `deterministic`/`inferred` from `G` enter `C`. Local guess is dual-run log only, never “prefer local.”

Blast radius of override: all rows where `guess.kind` won (`BLAST-RADIUS-ESTIMATE.md`). **Fail if you delete the guess row:** you lose R. Relabel those `unit`s UNVERIFIED; archive stubs (`PROVENANCE-ARCHIVE.md`).

`list_next` sorts `order_key`, not “model is sure.” **Fail if UI badges guess as green.**

## Still refuse

Worker `set_tier(deterministic)`, empty-hash A, fuzzy as inferred, steal on UNKNOWN, overage without exemption, `released` on guess. Tombstone any `if model.done: return COMPLETED` (`LEGACY-16.md`).

**Rule:** Tier is the producer. Deterministic wins every disagreement. Guess is a note. Inferred cannot close.
