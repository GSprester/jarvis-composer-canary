# adversarial review: 10-minute cron launching external processes

Mechanism: crontab `*/10` (or equivalent) starts a wrapper that `exec`/`Popen`s workers. Success is typically **cron exit 0**, empty mail, or “last run OK.” Confident wrong = **success, completeness, or safety** that does not hold.

## 1. Wrapper exits 0 without waiting (or after daemonize)

**Failure:** cron’s status is the wrapper. Child still running, already dead, or never exec’d. Estate marked **complete/safe**.
**Trigger:** `&`, `nohup`, `double-fork`, `start-stop-daemon`, `Popen` without `wait`, systemd `Type=oneshot` mis-set to `simple`.
**Symptom:** every tick green; no Sig1∧Sig2; orphans (`ORPHAN-RECLAMATION.md`); next tick overlaps (§3).
**Cheapest guard:** wrapper exit code = wait on the child. Non-zero if wait ERROR. Cron success is not COMPLETED (`DUAL-SIGNATURE-COMPLETION.md`).

## 2. `last_ok` / LKG / heartbeat written at tick start

**Failure:** freshness is **launch**, read as **completed/safe**. Every later failure still looks on-schedule.
**Trigger:** wrapper `touch last_ok` then `exec`; watchdog LKG before probe (`TASK-08-WATCHDOG-PATTERN.md`); metric scrape at crontab fire.
**Symptom:** `last_successful_scan` within 10m; child dead, skipped, or never exec’d; “no errors” (`SILENT-FAILURE-DETECTION.md`).
**Cheapest guard:** write `last_ok` only after wait **and** a completed MEASURE (canary-hit). Start-touch is not a scan (`LEGACY-04.md` §4).

## 3. Overlap: tick N+1 starts while N still runs

**Failure:** two external processes, two leads. Or the second exits 0 “already running” as **success**.
**Trigger:** job >10 min; drain 30s + work; hung NFS; `/10` and anacron catch-up.
**Symptom:** split-brain writes (`LEGACY-01.md`); or backlog never starts while cron stays green.
**Cheapest guard:** exclusive lock with `{pid,start}` (`LIVENESS-PREDICATE.md`). Second tick: non-zero or typed `OVERLAP`, not 0. Pid-only lock ⇒ recycle “already running” forever (`ADVERSARIAL-PID-ONLY-LIVENESS.md`).

## 4. Failed look → `[]` → exit 0

**Failure:** “no due work” is **complete**. Work is due.
**Trigger:** store timeout; ledger I/O ERROR mapped to empty (`ADVERSARIAL-LEDGER-QUEUE.md` §3); `find_due` except: return [].
**Symptom:** 10-minute greens; `last_successful_scan` stale (`LEGACY-04.md` §1).
**Cheapest guard:** look ERROR ⇒ cron ≠ 0. Canary-hit missing ⇒ ERROR (`POSITIVE-CONTROL.md`).

## 5. Partial due-set: first K waited, rest unstarted, exit 0

**Failure:** tick **complete**. Completeness of the interval, not of the bag.
**Trigger:** wall clock near +10m; `max_launch=1`; budget after first send; `break` on first 0.
**Symptom:** N−K still OPEN; cron green; next tick may skip as overlap (§3).
**Cheapest guard:** exit 0 only if `due` after MEASURE is empty **and** canary-hit present. Else ≠ 0 or typed `REMAINING`.

## 6. SIGTERM at next tick / timeout, then `status=completed`

**Failure:** killer fires at +10 min; wrapper writes success before fsync. Supervisor **last run OK**.
**Trigger:** `timeout 10m`; overlapping cron `-9`; systemd `RuntimeMaxSec=600`.
**Symptom:** `verify` UNVERIFIED; hot stub; presence brake (`CRASH-LOOP-BRAKE.md`).
**Cheapest guard:** never write COMPLETED. On SIGTERM: wait durable then disposition `timeout`, exit ≠ 0 (`TASK-31-GRACEFUL-SHUTDOWN.md`).

## 7. `queue_full` / admit deny / ceiling as success

**Failure:** nothing launched; tick **OK**. Safety of “handled.”
**Trigger:** N=32 full (`TASK-25-QUEUE-BACKPRESSURE.md`); `min(seat,provider)` deny; 402 (`TASK-28-ERROR-CLASSIFICATION.md`).
**Symptom:** backlog grows; cron green; monitor “no errors” (`SILENT-FAILURE-DETECTION.md`).
**Cheapest guard:** `queue_full` / `FailoverDenied` / 402 ⇒ exit ≠ 0 or typed ERROR. 429 ⇒ ≠ 0 this tick (retry same `qid`).

## 8. Wrong binary, PATH, cwd, or `$SHELL` — that binary exits 0

**Failure:** cron launched **something** that succeeded. Not the worker.
**Trigger:** cron `PATH=/usr/bin`; missing shebang; `python` vs venv; relative cwd `/`.
**Symptom:** no ledger `enqueued`; ticks green; “job is fine.”
**Cheapest guard:** absolute argv; wrapper checks `argv[0]` inode/digest. Exit ≠ 0 if exec target ≠ declared.

## 9. Wrong uid, `TREE`, or unmounted home — writes succeed elsewhere

**Failure:** that tree is **complete/safe**. The estate tree was not touched.
**Trigger:** root crontab vs user tree; `$HOME` unset (cwd `/`); NFS home not mounted (empty overlay); `TREE=` from crontab overrides.
**Symptom:** ticks green; live `open_units` unchanged; artifacts under `/root` or `/`.
**Cheapest guard:** refuse unless `realpath(TREE)` == declared and a canary-hit in **that** tree. Wrong uid/path ⇒ ≠ 0.

## 10. Pipe / `&&` / last-command 0

**Failure:** `worker | tee log` — tee 0 hides worker 1. Or `worker; echo done`. Cron **success**.
**Trigger:** missing `pipefail`/`set -e`; mail-friendly `echo`.
**Symptom:** logs show failure; crontab history 0.
**Cheapest guard:** `set -o pipefail`; exit = worker status only. No trailing `echo`.

## 11. Two hosts, one crontab (or one host, two schedulers)

**Failure:** each tick “safe” locally; two process trees.
**Trigger:** golden-image AMI; laptop + server; `*/10` in user and root crontab; systemd timer **and** crontab.
**Symptom:** two `promoted` for one `qid`; torn JSONL (`ADVERSARIAL-LEDGER-QUEUE.md` §8).
**Cheapest guard:** one lock on the **shared** root, `{pid,start,boot_id}`. Second host ≠ 0.

## 12. TZ / DST / UTC bag: “nothing this interval”

**Failure:** due rows in another civil hour. Tick **complete**.
**Trigger:** crontab local TZ; DST skip (interval vanishes) or double (two launches); `YYYYMMDD` partition (`TASK-19-TIMESTAMP-HAZARD.md`).
**Symptom:** 23:50–00:10 hole or double overlap; prefix unmatched (`RECEIPT-MATCHING.md`).
**Cheapest guard:** schedule and `scheduled_utc` in UTC. DST skip ⇒ ERROR if `now-last_successful_scan > 10m+ε`.

## 13. Anacron / missed ticks “caught up” as one success

**Failure:** one exit 0 covers N skipped intervals, or catch-up skipped as “too old” **OK**.
**Trigger:** machine asleep; `RANDOM_DELAY`; anacron `-n`.
**Symptom:** 3 hours of due work untouched; one green line.
**Cheapest guard:** each interval must have a completed MEASURE. Gap > 10m+ε ⇒ ERROR, not one combined 0.

## 14. Lock leftover, pid-only “still running,” exit 0 skip

**Failure:** tick **safely skipped**. Holder dead or recycled.
**Trigger:** crash; reboot; pid reuse (`ADVERSARIAL-PID-ONLY-LIVENESS.md` §1–2).
**Symptom:** 10-minute greens; lock file ancient; no `promoted`.
**Cheapest guard:** skip only if `same_process`. Else stale-archive and run, or exit ≠ 0.

## 15. `released` / empty queue from wrap or orphan

**Failure:** `list_next=[]` **complete**. Work `done` in ledger without launch, or evicted.
**Trigger:** wrap (`TASK-22-ROLLING-WINDOW-AUDIT.md`); `released` without `promoted` (`ORPHAN-RECLAMATION.md`).
**Symptom:** cron 0; `qid` dedup no-ops submit; backlog in archive.
**Cheapest guard:** idle only if canary-hit in replay and no `ORPHAN` bag. Else ≠ 0.

## 16. External process `exec`s away; wrapper waited on the old argv’s 0

**Failure:** wait returns 0 from a short parent; grandchild is the worker (or vice versa).
**Trigger:** launcher script `exec worker`; `ssh host cmd` 0 while remote still runs.
**Symptom:** cron 0; remote/child UNVERIFIED or still running.
**Cheapest guard:** wait on the **writer** incarnation (`pid+start` in the lock). SSH: wait for remote exit **and** Sig1; 0 from ssh is not COMPLETED.

## 17. OOM / cgroup kill of child; wrapper 0 or 137 remapped to 0

**Failure:** kernel killed the worker; tick **OK** or “handled.”
**Trigger:** memory.max; `oom_score`; wrapper `|| true`.
**Symptom:** stub payload; next tick presence-brake.
**Cheapest guard:** wait status must propagate SIGKILL/137 as ≠ 0. No `|| true`.

## 18. Cron itself disabled; last line still “OK”

**Failure:** safety of last success. No launches for days.
**Trigger:** `crontab -r`; `anacron` off; host down; comment `# */10`.
**Symptom:** dashboard last-run green/old; “no errors in the last hour.”
**Cheapest guard:** `last_successful_scan_utc` age > 10m+ε ⇒ broken, not healthy (`SILENT-FAILURE-DETECTION.md`).

## 19. MAILTO empty / stdout discarded; only exit 0 observed

**Failure:** worker printed ERROR; cron recorded **success**.
**Trigger:** `>/dev/null 2>&1`; `MAILTO=""`.
**Symptom:** operators see green; logs in a rotated file.
**Cheapest guard:** exit code from wait+MEASURE — not mail. Do not 2>&1 away before checking status.

## 20. Silent watchdog inside the tick: no print ⇒ cron 0

**Failure:** probe UNKNOWN mapped to silence; wrapper treats no stdout as healthy (`TASK-08-WATCHDOG-PATTERN.md`).
**Trigger:** probe timeout; fingerprint already fired.
**Symptom:** 10-minute 0; canary-hit missing.
**Cheapest guard:** watchdog UNKNOWN ⇒ wrapper ≠ 0. Empty stdout is not NONE (`POSITIVE-CONTROL.md`).

## 21. Budget / cheapest-first: send “succeeded,” work on the wrong seat or not at all

**Failure:** tick 0 after a 402 hop or skipped `reserve` (`BUDGET-AWARE-ROUTING.md`).
**Trigger:** `I` probe ERROR treated as 0; overage used; 429 as empty.
**Symptom:** included burned or work unsent; cron green.
**Cheapest guard:** quota probe ERROR or 402 ⇒ ≠ 0. Decrement `I` only after Sig1∧Sig2.

## 22. `C` local gate / model `status=completed` before the child exits

**Failure:** wrapper copies `W`’s bit and exits 0 (`EVIDENCE-TIERING.md`).
**Trigger:** fire-and-forget + self-report; dual-run prefer local (`DEPENDENCY-INVERSION.md`).
**Symptom:** COMPLETED; file empty; next ticks skip (`LEGACY-31.md`).
**Cheapest guard:** cron 0 never from `model_guess`. Only checker Sig2 after wait.

## 23. `nice`/`timeout`/`flock` is the external process; it exits 0, worker not started

**Failure:** `flock -n … -c worker` 0 because flock got the lock and exec failed, or `-n` failed and some wrappers still 0.
**Trigger:** `flock` not installed (shell no-op); `-n` on held lock with `|| exit 0`.
**Symptom:** greens; no `promoted`.
**Cheapest guard:** `flock -n` fail ⇒ ≠ 0. Exec fail ⇒ ≠ 0. Do not `|| exit 0`.

## 24. syslog / journal `CMD` line treated as success

**Failure:** cron **ran**. Completeness of the worker is inferred from the scheduler log.
**Trigger:** `CRON[…] (user) CMD (worker)`; systemd `Started`; operator grep for CMD.
**Symptom:** log proves fire; no wait, no Sig1; same lie as §1 with a different observer.
**Cheapest guard:** CMD/Started is launch, not MEASURE. Health from canary-hit + wait status only.

## 25. `DRY_RUN` / `SKIP_*` / `--once` leftover; wrapper exits 0

**Failure:** tick **OK**. Nothing launched by design, read as caught-up.
**Trigger:** env inherited from crontab; leftover `SKIP=1`; `--dry-run` in the committed line.
**Symptom:** greens; no `promoted`; `last_ok` advancing if §2.
**Cheapest guard:** dry-run / skip ⇒ ≠ 0 or typed `SKIP` that pages. Production crontab must not carry those flags.

## 26. Clock step: `last_ok` looks within 10m+ε

**Failure:** schedule **healthy**. Ticks were skipped or last_ok is in the future.
**Trigger:** NTP step back; VM restore; `hwclock`; TZ change without UTC store (`TASK-19-TIMESTAMP-HAZARD.md`).
**Symptom:** age(`last_ok`) small; gap in `scheduled_utc`; due work idle.
**Cheapest guard:** store `last_ok` as UTC. If `last_ok > now` or interval count ≠ completed MEASUREs ⇒ ERROR. Do not trust age alone after a step.
