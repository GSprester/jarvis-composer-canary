# adversarial review: auto-start job that launches one task per tick

Mechanism: a timer/cron/watchdog **auto-starts** and `offer`s / `Popen`s **exactly one** unit, then exits. Success is typically **tick 0**, **“launched 1,”** **`last_ok`**, or **already-running skip**. Confident wrong = **success, completeness, or safety** that does not hold.

## 1. Tick 0 after launching one = interval / bag **complete**

**Failure:** completeness of the **tick**, not of `due` (`BURST-REVIEW.md` trickle).
**Trigger:** `max_launch=1`; `break` on first 0; “keep the seat warm” (`ADVERSARIAL-CRON-10M.md` §5).
**Symptom:** N−1 OPEN; greens forever; change-detector “in progress” (`LEGACY-09.md`).
**Cheapest guard:** exit 0 only if MEASURE shows `due==∅` **and** canary-hit. Else typed `REMAINING` ≠ 0. One launch is not a drain.

## 2. Wrapper 0 without waiting on that one

**Failure:** auto-start **succeeded**. Child running, dead, or never exec’d (`ADVERSARIAL-CRON-10M.md` §1).
**Trigger:** `&`; `Popen` no wait; systemd `simple` vs `oneshot`.
**Symptom:** every tick green; no Sig1∧Sig2; next tick overlaps or skips.
**Cheapest guard:** tick status = wait on **that** child. Cron 0 is not COMPLETED (`DUAL-SIGNATURE-COMPLETION.md`).

## 3. `last_ok` at fire — freshness is **launch**

**Failure:** schedule **healthy**. The one never finished or never started.
**Trigger:** `touch last_ok` then exec; scrape at crontab (`TASK-08-WATCHDOG-PATTERN.md`).
**Symptom:** `last_successful_scan` within the interval; bag untouched (`SILENT-FAILURE-DETECTION.md`).
**Cheapest guard:** `last_ok` only after wait **and** a completed MEASURE. Start-touch is not a scan.

## 4. Skip “already running” as the successful one

**Failure:** tick **did its job** (launched 0, held 1). Holder dead, recycled, or a stranger (`LIVENESS-PREDICATE.md`).
**Trigger:** pid-only lock (`ADVERSARIAL-PID-ONLY-LIVENESS.md`); overlap > interval; `flock -n || exit 0`.
**Symptom:** greens; no `promoted`; backlog never starts — or two leads if steal anyway (`LEGACY-01.md`).
**Cheapest guard:** skip only if `same_process`. Else stale-archive and run, or ≠ 0. Overlap is `OVERLAP`, not 0.

## 5. Failed look `[]` = no task this tick = **idle**

**Failure:** one-per-tick of **nothing** is complete (`POSITIVE-CONTROL.md`).
**Trigger:** store timeout; ledger ERROR⇒empty (`ADVERSARIAL-LEDGER-QUEUE.md` §3); hook-blocked MEASURE (`ADVERSARIAL-FAIL-CLOSED.md` §1).
**Symptom:** ticks green; due work on disk; `unmatched=0`.
**Cheapest guard:** look ERROR ⇒ ≠ 0. Canary-hit missing ⇒ ERROR. Empty is NONE only after a completed look.

## 6. The one is the wrong `unit` (`ls`, newest, cheap, filename)

**Failure:** **a** task launched. Safety of `order_key` (`LEGACY-35.md`).
**Trigger:** `ls`; mtime; `p{n}` (`LEGACY-11.md`); cheapest-first (`BUDGET-AWARE-ROUTING.md`); live `p3` before review (`BURST-REVIEW.md`).
**Symptom:** aged `p0` waits; human line 1 ≠ peek; later `released` leapfrogs (`ADVERSARIAL-EARLIEST-PROMOTE.md`).
**Cheapest guard:** the one = `peek()` of `order_key` after the same MEASURE. `list_next()[0]==peek()`.

## 7. Two timers / two hosts each launch **their** one

**Failure:** each tick **safe** locally. Two process trees (`ADVERSARIAL-CRON-10M.md` §11).
**Trigger:** crontab + systemd; laptop+server; user and root `*/10`.
**Symptom:** two `promoted` one `qid`; torn JSONL (`LEGACY-01.md`).
**Cheapest guard:** one lock on the **shared** root `{pid,start,boot_id}`. Second host ≠ 0.

## 8. SIGTERM at next tick: the one `status=completed`

**Failure:** killer fires because “one per interval.” Supervisor **OK**.
**Trigger:** `timeout` = tick; overlapping `-9`; `RuntimeMaxSec`.
**Symptom:** stub; presence brake (`CRASH-LOOP-BRAKE.md`); next tick launches another or skips exists.
**Cheapest guard:** never write COMPLETED. Timeout disposition, ≠ 0 (`TASK-31-GRACEFUL-SHUTDOWN.md`).

## 9. `queue_full` / 402 / admit deny = this tick **handled**

**Failure:** nothing launched; auto-start **OK** (`TASK-25-QUEUE-BACKPRESSURE.md`).
**Trigger:** N=32; 402 (`ADVERSARIAL-QUOTA-RETRY.md`); `min(seat,provider)` deny.
**Symptom:** backlog grows; “launched 0” as success; monitor quiet.
**Cheapest guard:** `queue_full`/402/deny ⇒ ≠ 0 or typed ERROR. 429 ⇒ ≠ 0, same `qid`.

## 10. Wrong binary / PATH / `TREE` is the one that exits 0

**Failure:** auto-start launched **something**. Not the worker (`ADVERSARIAL-CRON-10M.md` §8–9).
**Trigger:** cron `PATH`; root vs user tree; unmounted home.
**Symptom:** no `enqueued`; ticks green; artifacts under `/` or `/root`.
**Cheapest guard:** absolute argv + digest. `realpath(TREE)` == declared + canary-hit. Else ≠ 0.

## 11. `exists` / leftover receipt: the one is **skipped as done**

**Failure:** tick chose skip. Completeness of that `unit` (`LEGACY-05.md`, `LEGACY-09.md`).
**Trigger:** `if output.exists()`; `re` names it (`ADVERSARIAL-RECEIPT-NAMES.md`); prior stub.
**Symptom:** that one never replaced; rest trickle; digest ≠ `expected`.
**Cheapest guard:** the one runs unless A∧B. Exists/name is not a skip.

## 12. Anacron / missed ticks: one launch covers N intervals

**Failure:** catch-up **complete**. N−1 intervals had no MEASURE (`ADVERSARIAL-CRON-10M.md` §13).
**Trigger:** machine asleep; `RANDOM_DELAY`; one-shot `@reboot` plus `*/10`.
**Symptom:** hours of due work; one green line; or burst of overlapping ones (§7).
**Cheapest guard:** each interval needs a completed MEASURE. Gap > period+ε ⇒ ERROR, not one combined 0.

## 13. Change-detector: one inflight ⇒ skip all future ticks **safe**

**Failure:** “already running” forever. Completeness of the auto-start policy (`LEGACY-09.md`).
**Trigger:** job longer than the period; pid recycle; `review_opened` count (`ADVERSARIAL-SELF-REVIEW-QUEUE.md` §16).
**Symptom:** one stuck unit; bag never advances; greens.
**Cheapest guard:** skip only `same` live on **that** `unit`. Other due still require `REMAINING` ≠ 0. Recycle ⇒ not inflight.

## 14. The one is a canary / control consumed as work

**Failure:** controls gone; next empty tick is **real idle** (`POSITIVE-CONTROL.md`).
**Trigger:** `list_next` includes `canary-hit`; “launch first json.”
**Symptom:** hit missing ⇒ should ERROR, treated as NONE.
**Cheapest guard:** canaries are not offerable. Missing hit after a completed look ⇒ ERROR. Do not launch controls.

## 15. Decrement `I` / `last_ok` on **launch**, not A∧B

**Failure:** quota **spent**; schedule **healthy** (`BUDGET-AWARE-ROUTING.md`).
**Trigger:** billing on `Popen`; `promoted`; HTTP 200 of the one.
**Symptom:** included gone; that unit UNVERIFIED; rest unstarted.
**Cheapest guard:** decrement `I` only after this process’s Sig1∧Sig2. `last_ok` after MEASURE.

## 16. `W` / model `completed` before the one exits

**Failure:** self-report is the tick (`TASK-10-SELF-REPORT.md`).
**Trigger:** fire-and-forget + `status=completed`; `C` local (`DEPENDENCY-INVERSION.md`).
**Symptom:** COMPLETED; file empty; next ticks skip (`LEGACY-31.md`).
**Cheapest guard:** tick 0 never from `model_guess`. Only checker Sig2 after wait.

## 17. Pipe / `echo OK` / grep last line after the one fails

**Failure:** auto-start **OK** (`ADVERSARIAL-SELF-SUMMARY.md` §6).
**Trigger:** `worker | tee`; `|| echo OK`; CI `grep OK`.
**Symptom:** logs FAIL; crontab 0; one lost, bag implied fine.
**Cheapest guard:** `pipefail`; exit = worker only. Typed object, not a substring.

## 18. TZ / DST / civil tick: “no one due this interval”

**Failure:** the one lives in another civil hour (`TASK-19-TIMESTAMP-HAZARD.md`).
**Trigger:** crontab local TZ; DST skip; `inbox/YYYYMMDD/`.
**Symptom:** hole or double tick; prefix unmatched (`RECEIPT-MATCHING.md`).
**Cheapest guard:** schedule UTC. Gap > period+ε ⇒ ERROR.

## 19. Auto-start disabled; last one still **OK**

**Failure:** safety of the last success. No launches for days (`ADVERSARIAL-CRON-10M.md` §18).
**Trigger:** `crontab -r`; host down; `# */n`.
**Symptom:** dashboard last-run green/old; “no errors in the last hour.”
**Cheapest guard:** `last_successful_scan` age > period+ε ⇒ broken, not healthy.

## 20. `DRY_RUN` / `SKIP` / flock not installed: 0 launched, tick **OK**

**Failure:** by-design no one. Read as caught-up (`ADVERSARIAL-MAILBOX-QUEUE.md` §25).
**Trigger:** env leftover; `flock` missing no-op (`ADVERSARIAL-CRON-10M.md` §23).
**Symptom:** greens; no `promoted`; `last_ok` advancing if §3.
**Cheapest guard:** skip/dry-run ⇒ ≠ 0 or typed `SKIP` that pages. `flock` fail ⇒ ≠ 0.

## 21. Next tick launches a **new** `qid` for the same work (or the same after timeout)

**Failure:** first one **handled** by the tick. Second effect (`TASK-27-RETRY-SEMANTICS.md`).
**Trigger:** UUID per tick; timeout UNKNOWN resend; exists skip then new id.
**Symptom:** two shells; or original orphaned (`ORPHAN-RECLAMATION.md`).
**Cheapest guard:** same `enqueue_id`. UNKNOWN ⇒ read, do not mint. New id ⇒ refuse.

## 22. Earliest-promote: the one is whoever **finished**, not the head

**Failure:** auto-start picks a ready later unit (`ORDERED-RELEASE-QUEUE.md`).
**Trigger:** `max_launch=1` among Sig1; `C` `exists`; release-on-verify.
**Symptom:** head stuck; tail COMPLETED; order “held.”
**Cheapest guard:** the one = lane head ∧ `may_release` after MEASURE. Later bytes sit.

## 23. Review trickle: one review item per tick, `B` **draining**

**Failure:** burst policy **met**. Hundreds of `review_id`s remain (`ADVERSARIAL-SELF-REVIEW-QUEUE.md` §25).
**Trigger:** 1 review / live job / hour; live-first UI; `S` keeps opening.
**Symptom:** `C` leftovers as done; live `p0` starved or review never ends.
**Cheapest guard:** burst `B` in hours, not one-per-tick. `list_next` = `B` then live. Trickle ≠ 0.

## 24. OOM / 137 / exec fail remapped to 0 — the one **ran**

**Failure:** kernel/exec killed the only launch; tick **OK** (`ADVERSARIAL-CRON-10M.md` §17).
**Trigger:** `|| true`; cgroup; `flock` exec fail.
**Symptom:** stub; next tick skip or crash-loop.
**Cheapest guard:** wait status must propagate. Exec/137 ⇒ ≠ 0.

## 25. MAILTO empty / stdout discarded; only tick 0 observed

**Failure:** the one printed ERROR; auto-start **success** (`ADVERSARIAL-CRON-10M.md` §19).
**Trigger:** `>/dev/null`; `MAILTO=""`.
**Symptom:** operators see green; bag unmoved.
**Cheapest guard:** exit from wait+MEASURE, not mail. Do not 2>&1 away before status.

## 26. `I` / monitor / UI: “1 launched” as progress **and** health

**Failure:** attempt count is **delivery**; one-per-tick is **capacity enough**.
**Trigger:** gauge `launched_per_tick=1`; `M` ignores `REMAINING`; badge from last_ok.
**Symptom:** included burned on attempts; `open_units` flat; greens (`TASK-34-METRIC-THAT-LIES.md`).
**Cheapest guard:** progress is A∧B count, not launches. Alert on stale scan and `due` age, not on `launched==1`.
