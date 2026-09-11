# LEGACY-07

**Strict** is how tight the rule is **when the check actually runs**. **Fail closed** is what happens **when the check cannot run** (timeout, missing field, unreadable path, unknown class). They are not the same. A check can be strict and still fail **open**: the happy path is pedantic; the error path pretends the predicate held (`TASK-20-FAIL-LOUD.md`, `LEGACY-03.md`).

| | Strict | Fail closed |
|---|---|---|
| Question | How exact is a successful comparison? | What is the verdict if we have no reading? |
| Tight digest match | Yes | Irrelevant until bytes are hashed |
| Timeout / missing key / `[]` from a dead store | May still return “ok” or skip | Must be deny / UNVERIFIED / raise — **not** COMPLETED, not “equal,” not 0 unmatched |

## Example — merely strict

Artifact checker requires **exact** sha256 (64 hex, every bit). That is strict. Implementation on I/O error:

```python
def check(path, expected):
    try:
        return sha256(path.read_bytes()) == expected
    except OSError:
        return True   # or return []; caller treats empty as pass
```

The digest rule is severe. The failure path is **fail open**: “I could not read” becomes “treat as matching.” A missing file after drain (`TASK-31-GRACEFUL-SHUTDOWN.md`) looks COMPLETED. Strictness did not protect anyone.

Windows path `==` that is case-sensitive (`LEGACY-02.md`) is also merely strict: it is picky on two well-formed strings and silent if `stat` fails.

## Example — fail closed

`verify(declaration)` (`TASK-24-ARTIFACT-DECLARATION.md`): missing, empty, stale, wrong hash, escape, or unreadable → **UNVERIFIED**. Only a completed read whose digest equals the **declared** digest is COMPLETED. `effective(seat, provider)` (`TASK-23-POLICY-INHERITANCE.md`): unknown class **raises**; it does not default to the wider label. Reboot sweep (`TASK-03-REBOOT-SWEEP.md`): missing `process_started_at` on a claimed row is **interrupted**, not left live.

Those predicates may also be strict (exact digest, exact `re`). The closed part is: **no reading ⇒ do not grant the privilege the check was supposed to gate.**

## Rule

Use both: strict on a successful measurement; closed when there is no measurement. Strict-only is theater. Closed-only with a loose matcher still attaches the wrong receipt (`TASK-07-DATE-ROLLOVER.md`).
