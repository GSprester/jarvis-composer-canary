# Process-boundary errors (failure must not become success)

**Status:** design only.  
**Depends on:** `POSITIVE-CONTROL.md`, `TASK-20-FAIL-LOUD.md`, `ABSENT-OUTPUT-STATE.md`, `LEGACY-04.md`.  
**Does not exist in this tree:** parent supervisor, child scanner, cron unit.

A **parent** process (`cron`, dispatcher, wrapper) reaps a **child**. If the child failed and the parent exits 0, the scheduler records **success** (`LEGACY-04.md`). That is converting a failure into a success. Logging the error on stderr does not fix it: the only bit most supervisors keep is the **exit code**.

Handled ≠ swallowed. Handled means the parent **preserves** the failure class across the boundary. Suppressed means the parent **maps** it to 0 / NONE / COMPLETED.

## Exit-code discipline

One typed object on the child’s stdout (`POSITIVE-CONTROL.md`). Stderr is not the type. Empty stdout + exit 0 is forbidden (parser invents NONE).

| Class | Child exit | stdout `kind` | Meaning |
|---:|---:|---|---|
| Completed hit / valid NONE | **0** | `FOUND` or `NONE` + both controls | Look finished; gate held |
| Completed reject (canary-miss, verify miss) | **0** or **3** (declared) | typed reject; **not** empty | Failure of the *unit*, look completed |
| Look / child did not complete | **2** | `ERROR` or omitted (parent invents ERROR) | Timeout, crash, bad JSON, UNKNOWN |
| Usage / cannot start | **2** | `ERROR` | Missing argv, not a grep-miss |

Do **not** use exit **1** for `NONE`. Callers treat 1 as `grep` no-match **or** as a script error (`POSITIVE-CONTROL.md`). One bit, two meanings.

Parent (`waitpid` completed — `ABSENT-OUTPUT-STATE.md`: not reaped ⇒ not finished):

```
child_status = reap(pid, start)          # same incarnation
if reap UNKNOWN: parent_exit = 2; kind = ERROR; stop

parent_exit =
    2  if child_exit == 2
       or stdout missing/unparseable
       or kind == ERROR
       or n_unknown > 0
    0  iff child_exit == 0
       AND kind in {FOUND, NONE}
       AND control.hit AND control.miss
       AND no COMPLETED written unless verify held
```

**Never:** `try: child() except: log; return 0`.  
**Never:** `if child_exit != 0: log; sys.exit(0)` to “keep cron green.”  
**Never:** map child 2 → `kind=NONE` or `unmatched=0`.

A **handled** failure is still a failure at the boundary:

```
handled  iff  child class is ERROR or unit-reject
              AND parent_exit != 0          # ERROR class
                  OR (unit-reject AND parent_exit == 0
                      AND typed reject AND verify UNVERIFIED
                      AND NOT kind==NONE without controls)
              AND no COMPLETED / released from this tick
```

Unit-reject (digest miss) may exit 0 only if the **object** says reject and the parent does not close work. Prefer exit 3 for unit-reject so cron still pages; if the estate uses 0+typed reject, the test below must still see the typed bit.

```
suppressed  iff  child_exit == 2 or kind == ERROR or verify miss
                 AND (parent_exit == 0 OR kind coerced to NONE/FOUND
                      OR COMPLETED/released appended)
```

Stderr text + exit 0 is **suppressed**. A wrapper disposition `ERROR` in the ledger + parent 0 is **suppressed** for the supervisor (`LEGACY-04` last-run OK).

Multiple children: parent exit is **2** if any child is 2 / unreaped / unreadable. Do not AND successes and hide one ERROR (`DISPATCH-POOL-LATCH.md`: one line, not 47 fake 0s).

## Test: handled vs suppressed

Issuer plants a child on the **same** path the supervisor runs (`LEGACY-39.md`). `D` is the parent.

| Plant | Required parent |
|---|---|
| `canary-child-error` — exit 2, `kind=ERROR` (or crash before flush) | `parent_exit==2`, no `released`, no `COMPLETED`, stdout `kind=ERROR` (parent may write it if the child wrote nothing) |
| `canary-child-ok` — exit 0, `NONE`+controls or `FOUND` | `parent_exit==0`, no ERROR invented |
| `canary-child-reject` — verify UNVERIFIED / canary-miss | no COMPLETED; parent_exit ≠ 0 **or** typed reject with `kind` ≠ NONE-without-controls |

```
suppressed  iff  D(canary-child-error) exits 0
                 OR publishes NONE/FOUND
                 OR appends released
handled     iff  D(canary-child-error) exits 2
                 AND effect(R) not in [0,hwm) as success
```

If `canary-child-error` is silent-healthy (`TASK-08` empty stdout + 0), the parent is the cron-green bug. Zero-acceptance: one such tick fails the boundary stratum (`CLASSIFIER-ZERO-ACCEPT.md`).

`D` must not start a fake child-error so it can pass (`LEGACY-17.md`). Timeout of `wait` is parent 2, not 0 (`ABSENT-OUTPUT-STATE.md`).

Worked: child `verify` times out (exit 2). Parent catches `TimeoutError`, prints “skipped,” exits 0. **Suppressed.** Correct: parent exit 2, `kind=ERROR`, unit UNVERIFIED.

## What this must not do

- Catch-all `except` + exit 0.
- Treat empty stdout as NONE after a non-zero or missing child.
- Use exit 1 for both grep-miss and crash.
- Record ledger ERROR and still give the supervisor 0.

**Rule:** Child 2 / ERROR / unreaped ⇒ parent 2 and no COMPLETED. Exit 0 is only a completed look with typed FOUND/NONE and both controls. The test is a planted child-error: if the parent exits 0, the failure was suppressed.
