# Artifact-derived ledger

**Status:** design only.  
**Depends on:** `IDEMPOTENT-EVENT-LEDGER.md`, `TASK-17-SEPARATION-OF-DUTIES.md`, `TASK-10-SELF-REPORT.md`, `TASK-18-COUNT-RECONCILIATION.md`, `DUAL-SIGNATURE-COMPLETION.md`.  
**Does not exist in this tree:** ledger file, derive runner, worker API.

An append-only ledger that accepts a worker’s `status=completed` / `DONE` / “I promoted” is a self-report (`TASK-10-SELF-REPORT.md`). Rows that close or move work must be **derivable** from artifacts a third party can re-hash. The worker may write **payload bytes**. It must not author the row that says those bytes settled.

`COMPLETED` remains the verify **view** (`TASK-06-ARTIFACT-CHECK.md`). The ledger is a projection of that view, not a second truth (`LEDGER-CLASSIFICATION-RECONCILE.md` inverted: artifacts win on close).

## Who may append

| Author | May append | Must not |
|---|---|---|
| Issuer / wrapper `K` | `enqueued` from a declared handoff; `promoted` after lock MEASURE; `released` after this process hashed evidence | Invent `expected`; copy `W`’s digest field |
| Checker `K` (≠ `W`) | `completion_sig2`, `released.delivery` | Write payload (`LEGACY-17.md`) |
| Worker `W` | Bytes at the declared locator | `released`, `type=completed`, `event_id` of a close |

Tombstone any `W.append(released)` (`LEGACY-16.md`). A line whose `pid+start` is `W`’s incarnation is **not** a close, even if `type` says so.

## Derivation

A row is legal only if a completed MEASURE of **named artifacts** reproduces it (minus `at_utc`). `event_id = sha256(canonical(obj minus at_utc))` (`IDEMPOTENT-EVENT-LEDGER.md`).

```
derive(unit) → row | absent | UNKNOWN
```

| `type` | Artifacts read (this process) | Derived fields |
|---|---|---|
| `enqueued` | Handoff record: `unit`, `path`, `expected_sha256` (`HANDOFF-RENAME-ID.md`) | Those fields. No `expected` ⇒ do not derive (`LEGACY-05.md`) |
| `promoted` | Published lock `writer-<unit>` `{pid,start}` **and** `same_process` completed | `pid`, `start`. Lock missing / `same` false / look UNKNOWN → no promote |
| `released` `verify_completed` | Locator bytes + handoff `expected`: `d = sha256(canonical(file))`, `d == expected` | `delivery.evidence_sha256 = d` |
| `released` `stale_archived` | Archive payload re-hashed == declared archive digest (`PROVENANCE-ARCHIVE.md`) | `evidence_sha256` of **archive**, not hot `exists` |
| `completion_sig2` | Same `d` this process just hashed + MAC under `K` | `d`, `expected`, `mac` |

`derive` does **not** read: worker `status`, stdout “done”, mtime, `exists` alone, filename `DONE`, chat ACK, `T` (`MODEL-ROUTE-IDENTITY.md`).

```
may_append(row)  iff  look(artifacts) completed
                      AND derive(unit) == row minus at_utc
                      AND author pid+start is K
                      AND n_unknown == 0
```

If `derive` is `absent`, do not append a close. If `derive` is UNKNOWN, do not append; do not mark `done` (`TASK-20-FAIL-LOUD.md`). The gate must not write the file so `derive` becomes true (`LEGACY-17.md`).

Canary-hit: planted payload matching `expected` → `derive` yields `released`.  
Canary-miss: planted `status=completed` and empty/wrong file → `derive` is `absent`. If a `released` row appears anyway, the ledger is accepting self-report.

## Reconciliation (ledger ↔ artifacts)

Two bags, same `unit` / `enqueue_id` identity — not path, not `name[:8]`.

```
A  =  { derive(u) : u in declared bag, derive ≠ absent }   # completed looks only
L  =  { replay(bytes[0:hwm]) closes and promotes }         # [0,hwm) only
R  =  reconcile(keys(A), keys(L))                          # TASK-18 bags
```

Failed look of A or L → **UNKNOWN**, not “already reconciled.”

| `R` | Meaning | Action |
|---|---|---|
| `unchanged` and field-equal (`evidence_sha256`, `pid+start`) | Projection holds | none |
| in `A` not in `L` | Artifact can close; ledger lags | `K` may `append(derive)` then flush then HWM. Do not call COMPLETED from `L` until then **or** treat verify as the view and append as catch-up — readers of `list_next` must not skip work because `L` is missing a close they have not verified |
| in `L` not in `A` | Ledger close **not** derivable | **Self-report (or torn artifact).** Do not treat as `done`. Append `type=ledger_unreflected` `{unit, pred_event_id, reason:not_derivable}`. Relabel UNVERIFIED. Do not delete the bad line (append-only). Do not rewrite `expected` to match `L` |
| same key, different `evidence_sha256` / `kind` | Conflict | Artifact wins for the **view**. Ledger keeps the old line; new `released` is refused until a human MEASURE. Do not last-write-wins on `at_utc` |
| `L.released` and `A` UNKNOWN | Not a miss | UNKNOWN; do not steal, do not fail-close |

Zero-acceptance: one underivable `released` fails the ledger stratum (`CLASSIFIER-ZERO-ACCEPT.md`).

`list_next` / `on_launch` use `replay` **intersect** `derive` for “done.” `L` alone is how a worker-written close skips the job (`LEGACY-09.md`). `A` alone without an append is a verify view, not a second inbox (`LEDGER-CLASSIFICATION-RECONCILE.md`).

Worked:

```
W writes status=completed, file empty
derive → absent
if L has released → R removed-from-A: unreflected, UNVERIFIED

W writes declared bytes, K hashes, L not yet appended
derive → released candidate
R added-to-A: K appends; then unchanged

Rename of locator (HANDOFF-RENAME-ID): derive uses current path, same unit+expected
L.unit still matches. Path-keyed reconcile would false-remove.
```

## What this must not do

- Append `released` because `W` said so.
- Reconcile by length of L vs count of files.
- Edit a prior JSONL line to “fix” underivable close.
- Let `derive` write the artifact it then accepts.
- Key `A` or `L` on filename date.

**Rule:** `may_append(row)` iff this process’s `derive(unit)` from artifacts equals the row. Reconcile bags of `unit`: artifact-only → append; ledger-only close → `ledger_unreflected` and UNVERIFIED; field mismatch → artifact wins the view, no silent rewrite.
