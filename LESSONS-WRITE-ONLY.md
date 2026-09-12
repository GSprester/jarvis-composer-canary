# Lessons store: write without read

**Status:** design only.  
**Depends on:** `LEGACY-15.md`, `STALE-DOC-CLAIMS.md`, `EVIDENCE-TIERING.md`, `SCHEDULER-POLICY-ENFORCE.md`.  
**Does not exist in this tree:** lessons file, injection hook, lesson reader.

A **lessons store** `S` accepts appends (“do not prefix-join,” “do not fail the remaining bag,” “empty stdout is not NONE”). If nothing on the **run path** reads `S` before the next admit, match, verify, or dispatch, `S` is **write-only memory**. The next seat repeats the defect. The row **exists**. Behaviour does not change.

This is document-without-retire (`LEGACY-15.md`) and a current-tense claim whose source is never measured (`STALE-DOC-CLAIMS.md`).

## Write-only memory

```
write(S, lesson)     # always succeeds: JSONL append, README bullet, chat
read_run(P, S)       # P ∈ {admit, match, verify, dispatch, form(C), E}
```

```
write_only(S)  iff  no P has a completed read of S
                    before P returns a verdict
```

| What operators see | What the next unit does |
|---|---|
| `lessons.jsonl` grew | Same prefix matcher (`TASK-04`) |
| “We learned 402 is not retry” | Same 402→200 loop (`TASK-28`) |
| Lesson in the worker prompt | Seat may ignore it; `T` is not a bind (`MODEL-ROUTE-IDENTITY.md`) |
| Dashboard “lessons: 12” | Count of writes, not of injections (`TASK-34`) |

A store with a **human** READ (someone might open the file) and no **machine** READ on `P` is still write-only for the estate. Grep in a PR review is not `read_run`. A model “recalling” the lesson is `model_guess` and must not be the launch predicate (`EVIDENCE-TIERING.md`).

`exists(S)` and `len(S)` are not a read. They prove the write side works.

## Why writes do not compose

The forbidden behaviour is a **callable** (matcher, `on_tick`, `derive`, hook). `S` is a **comment** unless that callable’s inputs include a digest of the lesson. Two predicates stay live: the old code path and the new sentence (`LEGACY-15.md`). Agreement between “we documented it” and “the job succeeded” is the change-detector skip (`LEGACY-09.md`).

Resume does not inherit in-seat memory of `S` (`TASK-05-DOCTRINE-DRAFT.md`). A later process with no chat history never sees the write.

## Injection point (makes a lesson change behaviour)

A lesson is **in force** only when a named `P` **reads** a canonical form of it and that read can **refuse** the old behaviour.

```
lesson = {
  id,
  forbids,              # machine predicate (e.g. match == prefix, 402→retry, empty⇒NONE)
  injection,            # P and the symbol it binds
  policy_sha256,        # digest of the clause P actually loads
  successor_id          # what P does instead (re join, latch, typed NONE)
}
```

**Legal injections** (one is enough; the lesson names which):

| `P` | How the lesson is read | Old behaviour after inject |
|---|---|---|
| `match` | Tests / `match = identity_match` (`TASK-04`) | Prefix join fails CI / raise |
| `admit` / `E` | Clause in effective snapshot (`HIDDEN-OVERRIDE-CONFIG.md`) | Hidden override or ceiling hop refused |
| `dispatch` | `pool_latch` on exhaustion (`DISPATCH-POOL-LATCH.md`) | Remainders not marked failed |
| `form(C)` | Mechanical accept (`MECHANICAL-ACCEPT-SPLIT.md`) | Unverifiable unit not enqueued |
| `derive` | Artifact-only append (`ARTIFACT-DERIVED-LEDGER.md`) | Worker `released` refused |
| hook | Tombstone raises (`LEGACY-16.md`, `CAPABILITY-QUARANTINE.md`) | Old symbol unsuccessful |

`in_force(lesson)` iff `read_run(P, lesson.policy_sha256)` completed **and** a planted **canary-miss** of `forbids` is **rejected** on that same `P` (`ACCEPTANCE-RATE.md`). Write-to-`S` without this look is not in force.

The injection is **before** the verdict, not after. A post-hoc “we should have learned” append is another write. Refuse-after-dispatch is the unverifiable-unit bug (`MECHANICAL-ACCEPT-SPLIT.md`).

`P` must not mint the lesson it then accepts (`LEGACY-17.md`). Issuer / wrapper binds `policy_sha256`. The seat that failed may **propose** a row in `S`; it may not edit `P`.

## Proof it is not write-only

```
write_only_probe:
  plant canary-miss = the forbidden behaviour
  if P accepts the miss:  S is write-only (or injection missing)
  if P rejects the miss:  lesson changed behaviour
  if look of P UNKNOWN:   UNKNOWN, not “lesson working”
```

Canary-hit: allowed behaviour still accepted (the successor). Zero-acceptance: one accepted miss fails the lesson stratum.

A count of rows in `S` is not this probe. Deleting `S` to “clean up” does not un-inject; the bound `P` remains until tombstoned with `successor_id`.

## What this must not do

- Treat append-to-`S` as the fix.
- Inject only into a prompt or a README current-claim.
- Let the worker be the only reader of its own lesson.
- Skip the canary-miss and publish `lessons_in_force = len(S)`.

**Rule:** A lessons store with no `read_run` on a gate is write-only memory. A lesson changes behaviour only when a named `P` loads its `policy_sha256` and rejects a planted instance of `forbids` before the verdict.
