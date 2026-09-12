# Mechanical-accept split

**Status:** design only.  
**Depends on:** `TWO-LANE-SPLIT.md`, `TASK-24-ARTIFACT-DECLARATION.md`, `TASK-06-ARTIFACT-CHECK.md`, `TASK-17-SEPARATION-OF-DUTIES.md`, `LEGACY-17.md`, `ARGV0-PERMISSION-HOOK.md`.  
**Does not exist in this tree:** splitter, dispatcher, live checker hook.

A splitter partitions a job into units and may enqueue them (`TWO-LANE-SPLIT.md`). Its **acceptance criterion** is not a model sentence and not “the lane finished.” It is a **mechanical command** `C` declared on the handoff: a third party runs `C` and reads the process exit. Exit 0 is COMPLETED. Non-zero is UNVERIFIED. Timeout / unreadable status is UNKNOWN.

If `C` cannot be formed, the unit is **unverifiable**. The splitter **refuses it before dispatch**. It does not compute `home`, publish a lock, or enqueue.

## What `C` is

`C` is an argv vector on the handoff, fixed **before** any worker runs.

```
C = [checker, --path, <lexical path>, --expected, <64 hex>, …]
```

| Required | Hold |
|---|---|
| Mechanical | Allow-listed checker binary (digest of the shipped `artifact_check` shape). No seat, no prompt, no human ACK. |
| Declared acceptors | `path` and `expected_sha256` on the handoff (`TASK-24`). `expected ≠ sha256(b"")`. Worker cannot add or replace them. |
| Third-party runnable | Anyone with the repo and the handoff can exec `C`. No worker session, no `T`. |
| Gate ≠ command | `C` **measures**. It must not create the file, write the digest, or mint a receipt (`LEGACY-17.md`). |
| Path | Lexical-safe, under root, checked on the declared string (`TASK-32-PATH-SAFETY.md`). |

`exists(path)`, HTTP 200, `status=completed`, and “the model says it looks right” are not `C`. A shell compound (`cd && …`) is not `C` — the verb is not in argv (`ARGV0-PERMISSION-HOOK.md`). `timeout` wrapping the checker is peelable only to reach the checker; the criterion is still the checker’s exit, not `timeout`’s.

Forming `C` is a **syntax and allow-list** look. It does not run `C` against output (the file must not yet be the obligation). A leftover at `path` is irrelevant at admit: `C` would be UNVERIFIED until the worker writes the declared bytes.

## Admit (before dispatch)

```
admissible(u)  iff  form(C_u) succeeds
```

`form` fails when any of: missing `C`, checker not allow-listed, missing/empty/malformed `expected`, path unsafe, `C` is a prompt or a worker-supplied digest, `C` would need the worker to exist in order to run.

```
on_split(u):
    if form(C_u) fails:  refuse(u, unverifiable); return
    home = sha256(job_id, u.re) mod 2
    enqueue(home, u)          # only now
```

Refuse is a wrapper disposition (`TASK-05-DOCTRINE-DRAFT.md`). The unit never enters a lane bag. `re` is not in `declared` for merge. Do not leave a stub handoff that a later seat can treat as `open`.

Controls on `form` itself (`ACCEPTANCE-RATE.md`): a fixture with a full mechanical `C` must be admitted (`canary-hit`); a fixture with no `expected` must be refused (`canary-miss`). If miss is admitted, the splitter is always-yes.

## Why refuse before dispatch, not after

After dispatch the unit has a **lock**, a **queue slot**, and a **path**. The worker will write bytes. Then “we cannot verify this” is a different sentence than admit-refuse, and the estate will treat it as a **worker miss**.

### 1. Recovery goes the wrong way

Post-dispatch UNVERIFIED looks like `failed_both` / steal / retry (`TWO-LANE-SPLIT.md`, `STALE-CLAIM-RECOVERY.md`). Those paths assume `C` exists and the work missed it. An unverifiable unit has **no** `C`. A second identical worker cannot create one (`SAME-MODEL-PAIR.md`). Pre-dispatch refuse is an **admit** miss: do not steal, do not launch the other lane, do not increment the crash-loop counter as if the shell were poison.

### 2. Leftovers become a fake `C`

The worker (or a helpful gate) sees a file and is asked to close. Without a declared `expected`, the only digest at hand is `sha256(written)`. Setting `expected` to that is the gate minting the evidence it accepts (`LEGACY-17.md`). `exists(path)` then looks deterministic (`EVIDENCE-TIERING.md`). The change-detector hashes the leftover and **skips** the job (`LEGACY-09.md`). Refuse-after has already lost declare-before-write (`TASK-24`). Refuse-before never offers that write.

### 3. The unit cannot enter `accepted`

Merge is bag-reconcile of `re` → one digest under Sig1∧Sig2 (`TWO-LANE-SPLIT.md`). Sig1 **is** `C` exit 0. If `C` was never formable, the unit can never be accepted. Dispatching it puts `re` in the job bag and guarantees a hole. Operators then “fix” `len` by marking COMPLETED or by dropping `re` from declared — both lie (`TASK-18-COUNT-RECONCILIATION.md`). Refuse-before keeps `re` out of `declared`. The hole is not a merge problem.

### 4. Slots and locks are the bound

`N` in-flight and one writer lock per unit (`TASK-25`, `TASK-16`) exist so slow **verifiable** work stalls producers. Spending them on unverifiable units hides that stall and opens steal under a claim that can never COMPLETED. Hard-kill recovery archives a receipt for work that had no checker. That receipt is not an attempt; it is pollution.

### 5. Separation of duties is timing

`COMPLETED(u)` iff a **caller** runs `C_u` (`TASK-17`). That requires `C_u` on the handoff **before** `W` runs. Dispatch-then-invent-`C` makes `W` the author of the obligation. A third party cannot re-derive it. The splitter’s criterion would then be a worker self-report (`TASK-10-SELF-REPORT.md`).

### Worked order

```
t0  form(C) fails          # no expected
t1  WRONG: enqueue, lock, W writes out/u
t2  refuse after           # “UNVERIFIED”
t3  steal / retry / expected := sha256(out/u)
t4  C' minted; exists-gate; job looks closed
```

Correct: stop at t0. No `home`, no lock, no path obligation. Disposition `unverifiable`. Issuer must supply `C` or drop the unit.

## What this must not do

- Dispatch “and we’ll figure out the check.”
- Run the worker so it can propose `expected`.
- Treat post-dispatch UNVERIFIED as the same class as a failed `C`.
- Use a model review, ACK, or `exists` as `C`.
- Admit on canary-miss (missing `expected`) or skip the `form` controls.

**Rule:** The splitter accepts a unit only when a mechanical `C` is already on the handoff. Unverifiable units are refused before `home` and enqueue. After dispatch, refuse cannot recover declare-before-write, and the estate will retry a checker that was never there.
