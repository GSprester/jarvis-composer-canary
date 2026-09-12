# Mechanical rule (not convention)

**Status:** design only.  
**Depends on:** `LEGACY-15.md`, `LEGACY-16.md`, `LESSONS-WRITE-ONLY.md`, `ARGV0-PERMISSION-HOOK.md`.  
**Does not exist in this tree:** live chokepoint, deny suite.

A **rule** `R` (worker cannot close work; prefix is not a join; unverifiable units are not dispatched; remainders are not failed on pool exhaustion) is **convention** if it lives in a README, lesson row, or prompt. Convention is write-only memory (`LESSONS-WRITE-ONLY.md`). `R` is **mechanical** when the forbidden effect is **unrepresentable** or **unsuccessful** on every live path (`LEGACY-16.md`).

“Please don’t” is discouraged. A planted violation that **cannot** produce the effect is the proof.

## Chokepoint

The **chokepoint** `C` is the unique (or fully enumerated) place the forbidden **effect** can enter the estate. Not the unique place someone might *try*.

```
effect(R)     =  the store change R forbids
                 (COMPLETED bit, prefix pair, enqueue of form(C)-fail,
                  failed disposition on a queued remainder, …)
paths(effect) =  every callable that can write that change
C             =  the gate each of those paths must pass
```

| Rule (example) | Effect | Chokepoint |
|---|---|---|
| Worker cannot COMPLETED | `released` / verify view = COMPLETED | `derive` / Sig2 append (`ARTIFACT-DERIVED-LEDGER.md`, `TASK-17`) |
| No prefix join | A `matched` pair from `name[:8]` | `match()` used by `list_next` / close (`SPARSE-RE-JOIN.md`) |
| No unverifiable dispatch | `enqueue` of a unit without mechanical `C` | `on_split` before `home` (`MECHANICAL-ACCEPT-SPLIT.md`) |
| No fail-on-exhaustion | `failed` on remaining `re` | `on_tick` behind `pool_latch` (`DISPATCH-POOL-LATCH.md`) |

`C` must sit **before** the write (`MECHANICAL-ACCEPT-SPLIT.md`: refuse before dispatch). A post-hoc check is convention plus a cleanup hope.

If `paths(effect)` is not enumerated, `C` is a comment. Outward jobs, a second matcher `pip` still provides, and “just run the shell” are paths (`OUTWARD-JOB-AUDIT.md`, `LEGACY-15.md`). Missing enumerator → UNKNOWN, not “only one chokepoint.”

`C` **measures**; it does not mint the artifact that would make `R` hold (`LEGACY-17.md`).

## Enforcement

Convention: a string near `C`. Mechanical: the bypass has **nowhere to write** a success downstream will treat as the effect.

```
enforce(C, attempt) =
    REFUSE / raise typed error     # unsuccessful
    OR the API cannot express it   # unrepresentable (no set_completed)
```

| Mechanism | Hold |
|---|---|
| Unrepresentable | No worker method sets COMPLETED; view is `verify` only |
| Unsuccessful | Tombstone raises; `match(prefix)` cannot emit `n==1` pair; latch blocks fail-writes |
| Typed, not `[]` | Refuse is ERROR / `queue_full` / `BypassRetired`, not empty-as-ok (`TASK-20`) |
| Same path as prod | Not a sidecar linter the cron does not run (`LEGACY-39.md`) |

Two predicates live (old callable + new comment) is **not** enforcement. Retire or tombstone the old symbol (`LEGACY-16.md` §1, §5).

## Test: violation impossible, not discouraged

Discouraged: a test that greps the README, or a canary that is never on the run path. **Impossible:** a planted attempt of `effect(R)` on **every** enumerated path **cannot** produce the store change.

```
# 1. Production chokepoint refuses the violation
attempt = canary-miss of effect(R)          # issuer-planted, same path as prod
assert C(attempt) is REFUSE
assert effect(R) not in replay(ledger[0:hwm])
assert verify / match / enqueue did not close or pair

# 2. Known wrappers / aliases do not widen (ARGV0 denial suite)
for W in peelable_and_side_channels:
    assert C(W ++ attempt) is REFUSE        # strip did not un-deny

# 3. The old symbol is unsuccessful or gone
assert import_or_call(retired_predicate) raises
assert grep(run_path, retired_symbol) == 0  # CI, not prose

# 4. Look failure is not a pass
assert C(timeout) is UNKNOWN                # not “no violation seen”
```

If (1) fails, `C` is convention. If (2) fails, a second path still writes the effect. If (3) fails, document-without-retire. If (4) fails, a failed look looks like “R holds.”

Zero-acceptance: **one** accepted miss fails the `R` stratum (`CLASSIFIER-ZERO-ACCEPT.md`). `A→1` without this miss is uninterpretable (`ACCEPTANCE-RATE.md`).

The test must not be the gate writing a deny row so the counter looks live (`LEGACY-17.md`). Issuer plants the attempt; `C` only measures.

Worked: `R` = worker cannot COMPLETED. Plant `status=completed` + empty file. `derive` is absent; no `released` appends; `verify` is UNVERIFIED. If a `released` line appears, violation is possible.

## What this must not do

- Enforce `R` only in a lesson store or PR template.
- Put `C` after the write.
- Leave the old callable importable.
- Call a green grep of the README a denial test.
- Treat UNKNOWN as “no one violated R.”

**Rule:** Enumerate every path that can write `effect(R)` and bind them to one `C` that refuses or cannot express it. The proof is a planted violation on those paths that cannot land in `[0,hwm)` — not a comment that asks them not to try.
