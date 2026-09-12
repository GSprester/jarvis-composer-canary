# adversarial review: pid-only liveness

Mechanism: `alive iff kill(pid,0)==0` / `OpenProcess(pid)` / `/proc/<pid>` exists. No `start`, no `boot_at`. Confident wrong = **live/safe/held**, **safe-to-steal**, or **recovery complete** when that is false.

## 1. Pid recycle after the holder dies

**Failure:** a new incarnation occupies the same pid. Check reports **live**. One-writer “safe.” The lock names a dead seat.
**Trigger:** short-lived processes; Linux pid wrap; Windows rapid reuse; crash-loop workers.
**Symptom:** no steal; units blocked for days; `released` waits on a stranger; substitute cover refused; burst-review skips (`same(S)` true on the wrong process). New process’s writes accepted as the holder’s.
**Cheapest guard:** `same iff pid exists AND start_os==record.start` (`LIVENESS-PREDICATE.md`). Recycle ⇒ start mismatch ⇒ stale.

## 2. Reboot, then the same small pids exist

**Failure:** every pre-boot lock pid matches `systemd`, `sshd`, or a login. Sweep leaves `claimed`/`running`. Estate reports **locks healthy**.
**Trigger:** host reboot; TASK-03 sweep uses pid-only; `start < boot_at` never tested.
**Symptom:** `reboot_interrupted` never written; `open_units` looks in-progress/safe; work is dead; “no errors” (`SILENT-FAILURE-DETECTION.md`).
**Cheapest guard:** `record.start >= boot_at` else stale. Reboot sweep does not consult pid existence alone.

## 3. Shared lock file, two kernels (NFS, clone, laptop + CI)

**Failure:** checker’s pid table is **this** host. Recorded pid exists **here** as an unrelated process. Check reports **live/safe**.
**Trigger:** lock on NFS; git worktree on two machines; container bind-mount of `.lock/`.
**Symptom:** host A’s worker dead; host B refuses steal; or B’s pid 4412 is `nginx` and “holds” A’s unit.
**Cheapest guard:** lock identity includes host `boot_id` / machine-id; `same` only if that boot. Else treat foreign boot as stale after MEASURE of the id.

## 4. Pid namespace mismatch

**Failure:** lock pid is container-local; checker is host (or reverse). Host pid exists as a different task, or does not exist (`ESRCH` ⇒ steal).
**Trigger:** worker in pod; sweep on node; `kill` from sidecar with another namespace.
**Symptom:** false **live** (host pid reused) or false **safe-to-steal** (ESRCH) while the container writer is live ⇒ two leads (`LEGACY-01.md`).
**Cheapest guard:** record `{nspid, ns_inum}` or check from **inside** the same pidns. Cross-ns pid-only is ERROR, not live/dead.

## 5. `exec` / image replace, same pid

**Failure:** incarnation changed; pid did not. Check reports **same holder / safe**.
**Trigger:** worker `exec`s the next job; `execve` crash wrapper; Windows `CreateProcess` then inherit pid tricks are rarer, Unix exec is enough.
**Symptom:** lock still “ours”; new argv writes a different `unit` into the same tree; Sig1 for the old `expected` never appears; detector skips (`LEGACY-09.md`).
**Cheapest guard:** OS start time changes across exec on Linux (`/proc/pid/stat` starttime is boot-relative and **does not** change on exec). **Starttime does not save you here.** Guard: `record.stime` **and** `record.exe_inode`/`starttime+boot_id` is insufficient for exec — compare `proc/pid/exe` digest or a spawn token written at acquire. Pid-only cannot. If only one extra field: spawn `start` **plus** `NSpid` **plus** refuse exec-in-place (worker must new pid). Document: pid-only **and** start-only both miss exec; cheapest **pair** is `start` (recycle) + `exe`/token (exec).

## 6. ACCESS_DENIED / EPERM mapped to dead

**Failure:** process exists; probe cannot inspect. Pid-only `OpenProcess` 0 / `kill` EPERM implemented as `ESRCH`. Check reports **safe-to-steal**.
**Trigger:** Windows error 5; Unix EPERM; container seccomp; sweep as another user.
**Symptom:** second writer; torn payload; two `promoted` (`ADVERSARIAL-LEDGER-QUEUE.md` §8).
**Cheapest guard:** EPERM/5 ⇒ UNKNOWN, no steal (`LIVENESS-PREDICATE.md`).

## 7. ACCESS_DENIED / EPERM mapped to live

**Failure:** opposite of §6. Unreadable pid reported **held/safe**. Recycle hidden.
**Trigger:** “EPERM means someone is there.” After death+recycle, new owner is unreadable.
**Symptom:** permanent block; crash-loop **accident** looks DESIGN (`CRASH-LOOP-BRAKE.md`).
**Cheapest guard:** EPERM ⇒ UNKNOWN, not live. Only `start` match is live.

## 8. Timeout / missing `/proc` mapped to live or dead

**Failure:** no reading. Implementer picks a boolean. Live ⇒ silent hang. Dead ⇒ mass steal.
**Trigger:** load, hung NFS `/proc`, Windows WMI hang.
**Symptom:** “all locks healthy” or “all recovered” in one tick (`LEGACY-38.md`).
**Cheapest guard:** timeout ⇒ UNKNOWN. Never a boolean.

## 9. Pid 0, −1, or process-group `kill`

**Failure:** `kill(0,0)` / `kill(-1,…)` is not “is record.pid alive.” Returns 0. Check reports **live/safe**.
**Trigger:** missing field default 0; JSON `pid:null` coerced; wrapper uses `os.kill(0,0)`.
**Symptom:** every empty lock “held”; or signal broadcast; steal never runs.
**Cheapest guard:** `pid` must be integer `>= 1`. Else not a lock (interrupted), not live.

## 10. Zombie pid still “exists”

**Failure:** `kill(pid,0)` succeeds; worker is a zombie. Check reports **live**.
**Trigger:** parent never `wait`s; supervisor bug.
**Symptom:** no steal; no progress; quiet hour (`SILENT-FAILURE-DETECTION.md`).
**Cheapest guard:** require `/proc/pid/stat` state ≠ `Z` **and** `start` match. Zombie ⇒ stale.

## 11. Observer’s own pid

**Failure:** sweep overwrites or compares `pid==os.getpid()` and reports **held by us / safe**, or **not stale**.
**Trigger:** `ps | grep` hits the grepper; lock rewritten empty then filled with sweeper pid (`LEGACY-13.md`).
**Symptom:** recovery never; or sweeper is “lead” and SIGTERMs itself.
**Cheapest guard:** subtract `{self.pid, self.start}` before decide/kill. Pid-only cannot tell self-incarnation after exec.

## 12. Fork: lock has parent pid; child still writes

**Failure:** parent exits; pid-only ⇒ **dead, safe-to-steal**. Child holds the tree.
**Trigger:** daemonize; `fork` without rewriting lock; Windows `CreateProcess` + parent exit.
**Symptom:** steal + child write; split-brain.
**Cheapest guard:** lock must be republished after fork with the writer’s `{pid,start}`. Sweep steals only that incarnation. Pid-only of the parent is not “child dead.”

## 13. Truncated / 16-bit / string pid

**Failure:** stored `pid` 4412 vs 69928 (`4412+2^16`) or `"04412"`. Check hits the wrong slot: false **live** or false **dead**.
**Trigger:** ABI, JSON float, leading zeros, Windows handle vs pid.
**Symptom:** random live/dead; steal of an innocent process or wait on init.
**Cheapest guard:** pid integer, full width; compare OS start of **that** pid. Mismatch ⇒ stale, not live.

## 14. TOCTOU: check then act

**Failure:** pid-only snapshot; then death+recycle or spawn. Act on a stale boolean.
**Trigger:** sleep between `kill(pid,0)` and unlink/acquire; no compare-and-unlink of lock bytes.
**Symptom:** steal a **new** live process that just got the pid (false safe-to-steal); or skip steal after recycle (false live).
**Cheapest guard:** CAS unlink **the lock bytes** you judged; re-read `{pid,start}` immediately before unlink. Pid-only has no second field to CAS against recycle.

## 15. Remote job, local `kill`

**Failure:** lock pid is on the worker host; checker is the dispatcher. Local pid **live/safe** or **dead**.
**Trigger:** SSH workers; cloud VM; pid recorded, check on router.
**Symptom:** router thinks holder lives (local sshd pid) or steals while remote worker runs.
**Cheapest guard:** liveness probe on the **record’s host** (or pidfd-over-RPC). Pid-only on the wrong machine is ERROR.

## 16. `list_next` / UI: “holder alive” from pid

**Failure:** operators see safety; cancel/kill the **current** occupant of that pid (wrong process) or wait on a recycle.
**Trigger:** `LEGACY-35` listing uses pid-only `alive`.
**Symptom:** killed innocent job; or “still running” for days.
**Cheapest guard:** display `same_process` (pid+start), not `pid_exists`.

## 17. Reclaim / crash-loop / promote keyed on pid-only `same`

**Failure:** `promoted` says inflight/safe; `orphan` skipped; DESIGN brake never trips because a recycled pid is “the worker.”
**Trigger:** ledger + pid-only (`ADVERSARIAL-LEDGER-QUEUE.md` §2, `ORPHAN-RECLAMATION.md`).
**Symptom:** `qid` stuck inflight/complete; never-tried looks launched.
**Cheapest guard:** `promoted.start` must match OS start; else not inflight. Recycle ⇒ stale, not live.

## 18. Pause / freezer cgroup: pid exists, work is not safe

**Failure:** pid-only **live**. Drain/OOM-freeze; no progress. Safety of “holder working.”
**Trigger:** kube pause; SIGSTOP; Windows suspend.
**Symptom:** silence = live work (`LEGACY-37.md`); no ERROR in the hour.
**Cheapest guard:** pid-only cannot. Need `start` match **plus** a completed MEASURE (canary / heartbeat **not** written by the gate). Frozen + matching start is **old and live** — wait, do not report “healthy progress.” Separate liveness from progress.
