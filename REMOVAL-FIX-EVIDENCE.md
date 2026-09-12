# Removal-fix evidence

**Status:** design only.  
**Depends on:** `LEGACY-15.md`, `LEGACY-16.md`, `POSITIVE-CONTROL.md`, `TASK-20-FAIL-LOUD.md`, `ACCEPTANCE-RATE.md`.  
**Does not exist in this tree:** retired symbol, tombstone module, live successor.

A fix that **removes** X (predicate, hook, cron, symbol, file) cannot be proved by `not exists(X)`. That observation is also **never present** and also **look failed** (`[]` / timeout). It is also **removed the gate** so nothing can fail. The evidence is a triple that a third party can re-run: **was present**, **not on the run path**, **successor still a gate**.

## The three look-alikes

| Estate | Naive `grep` / `exists` / empty listing |
|---|---|
| Removed and working | X gone |
| Removed and broken | X gone |
| Never present | X gone |
| Look did not complete | X “gone” (`TASK-20-FAIL-LOUD.md`) |

`NONE` without controls is not a removal proof (`POSITIVE-CONTROL.md`).

## What must be on the handoff before the delete

The issuer declares X as a **retirement**, not as a missing file:

```
retired = {
  symbol,                 # import path, argv[0], register_id, predicate name
  before_digest,          # sha256 of the bytes that used to run
  archive_path,           # those bytes after copy+verify (TASK-22)
  successor_id,           # S: the callable that remains (LEGACY-15)
  successor_digest,
  tombstone_class         # BypassRetired / SearchFailed — must raise, not return []
}
```

`before_digest ≠ sha256(b"")`. SUPERSEDED without `successor_id` is not this fix. The worker does not supply `before_digest` after unlink (`TASK-24-ARTIFACT-DECLARATION.md`).

## Checkable triple

All three looks **completed**. Any UNKNOWN → do not publish “fix worked.”

### 1. Was present (`history`)

```
history(X)  iff  archive verify(archive_path, before_digest) == COMPLETED
                 AND ledger has type=retired citing that digest and successor_id
                 AND grantor / wrapper pid+start wrote the row (not the seat)
```

Hold → X **was** in the estate. Miss → you cannot claim a removal-fix. That is **never present** (or a story).

### 2. Not on the run path (`absent_live`)

A completed look that **would have invoked X** must hit the tombstone and **raise** (`LEGACY-16.md` §1).

```
absent_live(X)  iff  import/call/exec of symbol raises tombstone_class
                     AND enumerators (cron, pip, failover hop) show no live digest == before_digest
                     AND canary-miss: a planted call to the old name is ERROR/raise, not 0, not []
```

| Wire | Reading |
|---|---|
| Tombstone raises | X retired from the callable path |
| Old digest still BOUND / import succeeds | **not removed** |
| `[]` / exit 0 / NONE without raise | **removed and broken** (or never-tombstoned): empty-as-success |
| Timeout / unreadable listing | UNKNOWN, not absent |

Documentation that X is gone while the module still imports is LEGACY-15 document-without-retire.

### 3. Successor is still a gate (`successor_ok`)

```
successor_ok(S)  iff  canary-hit on S ACCEPTED
                      AND canary-miss on S REJECTED
                      AND n_unknown == 0
```

Same path as production (`ACCEPTANCE-RATE.md`). If X **was** the only rejector, deleting it without a working S makes A→1. That is **removed and broken**, even when `absent_live` holds.

## Verdict

```
removed_working   iff  history ∧ absent_live ∧ successor_ok
removed_broken    iff  history ∧ (not absent_live in the empty-success sense
                                  OR not successor_ok)
                      # X was here; either it still runs, or it is gone and the estate cannot accept+reject
never_present     iff  not history
                      AND absent_live is “no symbol / no archive”
                      # do not call this a successful fix
unknown           iff  any look failed to complete
not_removed       iff  history ∧ (live digest == before_digest ∨ old call exits 0)
```

`never_present ∧ successor_ok` is “S works and X was not in this tree.” Useful inventory. **Not** evidence the fix landed.

`never_present ∧ not successor_ok` is a broken estate with no retirement story. Do not treat as “we removed the problem.”

## Distinguishing table (worked)

X = prefix matcher; S = `receipt.re == handoff.name`.

| history | call prefix matcher | S hit / S miss | Verdict |
|---|---|---|---|
| archive + `retired` | `BypassRetired` | accept / reject | **removed and working** |
| archive + `retired` | `BypassRetired` | miss accepted | **removed and broken** (gate gone) |
| archive + `retired` | `[]` / exit 0 | anything | **removed and broken** (silent stub) |
| archive + `retired` | still pairs on `YYYYMMDD` | anything | **not removed** |
| no `retired` row | no symbol | accept / reject | **never present** |
| look timeout | — | — | **UNKNOWN** |

## What this must not do

- Prove the fix by `git rm` / `not exists` / empty grep.
- Leave a stub that returns `[]` or `True`.
- Skip `history` and call absence a win.
- Delete X and declare success before S’s hit **and** miss complete.
- Let the gate mint `before_digest` from an empty file after unlink (`LEGACY-17.md`).

**Rule:** A removal-fix is proved only by archive+`retired` (it was there), tombstone raise on the old call (it is not on the path), and successor hit-accept + miss-reject (the estate still works). Absence alone is never present or broken or UNKNOWN.
