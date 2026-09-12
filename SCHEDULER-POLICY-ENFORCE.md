# scheduler-enforced continuous policy

A policy that must **hold in time** — no-contact window, freeze, throttle — whose mechanism is a **scheduler entry** (crontab, systemd timer, cloud schedule), not an `admit` / `class(argv)` predicate. This checkout has no live timer. The design is that mechanism and the **verification that it is in force**, not merely written down (`OUTWARD-JOB-AUDIT.md`, `LEGACY-39.md`, `LEGACY-04.md`).

```
P  ∈  { no_contact(W), freeze(W), throttle(k, W) }
W  =  UTC interval or calendar   # not local civil (TASK-19)
```

Enforcement-as-schedule means: the **only** intended launcher is a register whose calendar **is** P (skip W, or fire at most k times in W). Documentation of P without that live entry is a comment (`LEGACY-22.md`: an override is a row, not a README).

## Why “it’s in the crontab comments” is not in force

| What operators have | What can still happen in W |
|---|---|
| README “freeze until Friday” | A second register (`user crontab` + systemd) still fires (`ADVERSARIAL-ONE-PER-TICK.md` § dual) |
| In-repo `deploy/freeze.timer` | Host still points at last week’s unit (`OUTWARD-JOB-AUDIT.md` STALE) |
| Empty crontab in git | Prod cron was never the only launcher |
| Commented `# */10` | Anacron / cloud scheduler / laptop copy still runs |
| “We don’t schedule weekends” | Manual `systemctl start`, `sudo` hop, or `at` |

Absence of a documented job is **not** P. A missing freeze-timer is **not** “nothing runs.” Both empty listings without a canary are ERROR (`TASK-20-FAIL-LOUD.md`).

Scheduler-only enforcement is also **bypassable by any path that is not that timer** (interactive shell, second host, Model-B `just run`). This design still requires verification of the **register**. It does not replace `admit` for MUTATE (`TASK-13-FAILOVER-CEILING.md`). If MUTATE must be impossible in W, the gate must refuse too — the timer is not a hook (`GATE-DEADLOCK.md`).

## Mechanism (the entry)

One declared outward job `E` per policy, identity `register_id` + argv digest:

| P | What `E` actually is |
|---|---|
| no-contact W | Calendar of **page/mail/launch** timers has **no** next_elapse in W. `E` may be a timer that **only** exists to be listed (witness) plus **absence** of others in W |
| freeze W | No BOUND/OUTSIDE/STALE launcher has OnCalendar ∩ W except an explicit `thaw` job whose calendar is **outside** W |
| throttle k / W | The one launcher’s calendar implies ≤ k fires in W; overlap lock `{pid,start}` so a long tick does not add a second lead |

`E` is BOUND (`OUTWARD-JOB-AUDIT.md`): live digest == declared. STALE or OUTSIDE launchers **violate P** even if `E` is perfect.

Write `policy_sha256` of canonical `{P, W, k, E}` next to the unit, not inside the payload (`LEGACY-21.md`). Intentional calendar edit is a grant-shaped receipt (`LEGACY-27.md`). Packaged default “every 10 minutes” returning after a reimage is a silent reset of P.

## Verification that P is in force

A completed look, **same path as production registers**, every probe interval (not a weekly doc review). Canaries first (`POSITIVE-CONTROL.md`).

```
in_force(P)  iff
    look(enumerators) completed
    AND canary-hit register row present     # enumerator works
    AND canary-miss job is absent/DISABLED  # a job that MUST fire in W if policy is ignored
    AND E is BOUND and enabled
    AND ∀ job in bag \ {E, canaries}:
          calendar(job) ∩ W = ∅
          OR verdict(job) ∈ {DISABLED}
    AND throttle: fires_in(W, E) ≤ k
        measured from ledger execs / timer LastTrigger, not from the comment
    AND n_unknown == 0
```

If any conjunct fails → **P not in force** (or UNKNOWN if the look failed). Do not publish “freeze holds.”

### Positive control (`canary-hit`)

A declared timer that **must** appear in the listing (witness that enumeration still sees the register). Missing → ERROR, not “no jobs, so freeze is fine.”

### Negative / seeded fault (`canary-miss`)

A declared **forbidden** identity: a job whose calendar **intersects W** (or a throttle of k+1). It must **not** be enabled in the live bag.

- If `canary-miss` is **present and enabled** → instrument or estate **rejects P**. Halt on that.
- If `canary-miss` was never installed in the register’s universe (only a string in git) → you did not test the enumerator (`LEGACY-39.md`: miss must be a real row a prefix would steal). Plant a **disabled** miss unit the listing still returns, or a live miss in a **lab register**. Listing that cannot see disabled units cannot verify freeze.

### Edge probe

At `W.start` and `W.end` (UTC): read `next_elapse` / `crontab` next fire. A prod launcher with next_elapse ∈ W → breach. Do not trust OnCalendar text alone (timezone, DST, `OnBootStrap`).

### Throttle count

`k` is checked against **completed launches** in W (wrapper wait, not `touch last_ok` at start — `LEGACY-04.md` §4). Overlap tick that exits 0 “already running” still counts as a fire if a second `{pid,start}` exists.

### Continuous

Same function every admit/tick that cares about P (`LEGACY-39.md`). Caching “freeze ok” in LKG the job wrote is the change-detector deadlock (`LEGACY-09.md`). Hash declared `policy_sha256` + live unit digest, not the last sentence.

Zero-acceptance (`CLASSIFIER-ZERO-ACCEPT.md`): one enabled launcher in W fails the **in_force** stratum for that host. One accepted miss (forbidden job running) fails the verifier.

Append `policy_probed` { `policy_sha256`, `in_force` | `UNKNOWN` | `breach`, `E_digest`, `n_jobs_in_W` } to the same ledger (`LEDGER-CLASSIFICATION-RECONCILE.md`). README is not that row.

## What this does not do

- It does not treat a documented window as a gate.
- It does not infer freeze from `open_units=0` or empty crontab in git.
- It does not let the model write the calendar.
- It does not skip `admit` for MUTATE during W if the estate also needs a code gate.

**Rule:** P is in force only while the live registers say so: enforcer BOUND, no other enabled calendar intersecting W, canary-hit seen, seeded forbidden job not enabled. A policy that exists only in the tree is not holding.
