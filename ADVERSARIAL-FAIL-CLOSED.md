# adversarial review: gate that fails closed on any query error

Mechanism: on timeout, raise, torn body, missing key, or canary-absent the gate **blocks** (no admit, no dequeue, no steal, no COMPLETED). Success is typically **deny**, **UNKNOWN**, **0 launches**, **`[]`**, **“policy held,”** or a **side path that still greens**. Confident wrong = **success, completeness, or safety** that does not hold.

## 1. UNKNOWN collapsed to NONE / `[]` / idle — block read as **caught up**

**Failure:** no reading is a **completed zero**. Fail-closed became fail-open (`LEGACY-07.md`, `TASK-20-FAIL-LOUD.md`).
**Trigger:** `except: return []`; hook ERROR⇒NONE (`GATE-DEADLOCK.md`); `unmatched=0` (`TASK-34-METRIC-THAT-LIES.md`).
**Symptom:** cron 0; WOs OPEN; “no errors” (`SILENT-FAILURE-DETECTION.md`).
**Cheapest guard:** ERROR is a type, not empty. Canary-hit missing ⇒ UNKNOWN, not NONE (`POSITIVE-CONTROL.md`). Idle only after a completed look.

## 2. Block = **safe** / “gate working” / 0 launches = estate protected

**Failure:** safety of a quiet deny. Due work never measured; holders may be dead.
**Trigger:** dashboard `denied>0` as health; last_ok at start (`ADVERSARIAL-CRON-10M.md` §2); 0 violations (`ADVERSARIAL-CMD-HOOK.md` §6).
**Symptom:** backlog grows; lock leftover; operators: fail-closed held.
**Cheapest guard:** block is `probe_unknown`, not healthy. Alert if `last_successful_scan` stale. 0 launches ≠ MEASURE.

## 3. UNKNOWN collapsed to `unhealthy`; stub canary / rewrite so the next look **passes**

**Failure:** timeout paged as a finding; the “fix” mints the control (`LEGACY-38.md`, `LEGACY-17.md`).
**Trigger:** insert canary; `seat.ceiling = provider.class`; `expected` from file.
**Symptom:** subsequent COMPLETED/admit; first error never a reading.
**Cheapest guard:** UNKNOWN ≠ unhealthy. Do not mint canaries or rewrite ceiling to unstick. Retry the **probe**, same `unit`.

## 4. Fail-closed on MEASURE deadlocks publish; another predicate still **greens**

**Failure:** `verify`/`find_*` blocked; `exists`, summary, or cron 0 is **done** (`GATE-DEADLOCK.md`).
**Trigger:** hook on read; `test -f`; `print("OK")` (`ADVERSARIAL-SELF-SUMMARY.md`); disable `H` to unstick (`LEGACY-12.md`).
**Symptom:** UNVERIFIED + green side path; two leads if `H` dropped.
**Cheapest guard:** MEASURE never refused. Dual-run until exists/OK is tombstoned (`LEGACY-15.md`). Do not disable `H`.

## 5. Steal refused on UNKNOWN ⇒ leftover **safe**; or steal proceeds on a side path

**Failure:** dead holder **held** (skip forever) while work is due — or `C`/`ls` still archives (`LIVENESS-PREDICATE.md`).
**Trigger:** lock unreadable ⇒ not stale; mtime steal anyway (`LEGACY-14.md`); pid-only live.
**Symptom:** accident brake (`CRASH-LOOP-BRAKE.md`); or two leads despite the gate.
**Cheapest guard:** UNKNOWN ⇒ no steal **and** no “healthy lock.” Side paths must use the same `same` probe. Persist `probe_unknown`, not skip-as-done.

## 6. `I` probe ERROR ⇒ 0 entitlement **handled** (or ∞ then 200 **delivered**)

**Failure:** fail-closed on quota looks like no work, or the other branch fail-opens (`BUDGET-AWARE-ROUTING.md`).
**Trigger:** missing `I`⇒0; empty body⇒NONE; missing⇒∞ (`ADVERSARIAL-QUOTA-RETRY.md` §6–7).
**Symptom:** 402 bag “complete”; or billed send after a blocked probe.
**Cheapest guard:** probe ERROR ⇒ ineligible this pop, not idle, not ∞. 200 is not A∧B.

## 7. Deny / escalate / dead-letter = **complete**

**Failure:** fail-closed disposition **closed** the unit (`ADVERSARIAL-QUOTA-RETRY.md` §8).
**Trigger:** `do_not_retry`; ticket opened; `released` `kind=escalated`; `P` ACK.
**Symptom:** `list_next` omits `qid`; work UNVERIFIED.
**Cheapest guard:** escalate is typed ERROR, unit stays UNVERIFIED. No `released` without Sig1∧Sig2 or verified `stale_archived`.

## 8. Permanent deny after one timeout (first crash never retries)

**Failure:** one query error **settled** the bag (`STALE-CLAIM-RECOVERY.md`).
**Trigger:** cutoff=1; DESIGN inferred from ERROR count (`CRASH-LOOP-BRAKE.md`); cache last deny.
**Symptom:** OPEN work frozen as safe-skip; or poison retried if someone “fixes” the cache.
**Cheapest guard:** UNKNOWN retries the **read**. DESIGN only wrapper `crash_loop` after `blocked→interrupted` ≥3, not HTTP/query class. Cache deny is not a finding.

## 9. Fail-closed on `G`; `C` prefer local **succeeds**

**Failure:** the closed gate is not on the run path (`DEPENDENCY-INVERSION.md`).
**Trigger:** `G` timeout ⇒ last local FOUND; dual-run prefer `C`; README says closed.
**Symptom:** split-brain; local exists/prefix COMPLETED; `G` UNKNOWN ignored.
**Cheapest guard:** `G` ERROR ⇒ `C` ERROR. No local fallback. Tombstone prefer-local.

## 10. Ledger look ERROR ⇒ skip torn `promoted` / rest of prefix **applied as done**

**Failure:** fail-closed on one line continues; replay **caught up** (`ADVERSARIAL-LEDGER-QUEUE.md` §3).
**Trigger:** skip-bad-JSON; HWM=length; wrap without canary.
**Symptom:** `done` missing work or including a torn `released`; queue green.
**Cheapest guard:** any I/O/JSON/missing-canary ⇒ replay ERROR, not empty, not skip. Do not advance HWM.

## 11. `list_next=[]` from a blocked query; operator **nothing next**

**Failure:** listing is a completed idle (`LEGACY-35.md`, `LEGACY-33.md`).
**Trigger:** fail-closed peek; UI hides UNKNOWN; cancel-by-absent-line.
**Symptom:** human skips the checker; action still has a canary-visible bag.
**Cheapest guard:** failed look raises. `[]` only after completed MEASURE + canary-hit. UNKNOWN is visible, not idle.

## 12. Fail-closed on `verify`; leftover `exists` / receipt **names** it closed

**Failure:** the hasher blocked; the namer still **wins** (`ADVERSARIAL-EXISTS-SCANNER.md`, `ADVERSARIAL-RECEIPT-NAMES.md`).
**Trigger:** `if output.exists(): skip`; `re` without A; sidecar `.ok`.
**Symptom:** COMPLETED without a completed hash; fail-closed never ran A.
**Cheapest guard:** skip only A∧B. Exists/name without a finished verify is UNVERIFIED, not skip.

## 13. Admit deny on ERROR ⇒ hop wider / overage **OK**

**Failure:** fail-closed on bind/quota is **failover success** (`TASK-13-FAILOVER-CEILING.md`).
**Trigger:** 401 region as invalid (`LEGACY-23.md`); 429⇒402 hop (`ADVERSARIAL-QUOTA-RETRY.md` §4).
**Symptom:** restricted WO on public; remaining `I` unused; hop looks like the gate held.
**Cheapest guard:** ERROR ⇒ no hop. 429 backoff same pair. Ceiling never rewritten. Overage needs exemption.

## 14. LKG / last_ok not overwritten — previous **healthy** still trusted

**Failure:** fail-closed forbade LKG update; callers read the old bit as **now** (`TASK-08-WATCHDOG-PATTERN.md`).
**Trigger:** scrape last_ok; watchdog silence; `scan.ok` leftover.
**Symptom:** `last_successful_scan` within interval from an older success; current look UNKNOWN.
**Cheapest guard:** stale LKG + last outcome UNKNOWN ⇒ broken, not healthy. Age the scan, don’t reuse.

## 15. “Any error” includes 429 / `queue_full` / overlap — **handled** as closed deny

**Failure:** time-limited or backpressure states **settled** (`TASK-28-ERROR-CLASSIFICATION.md`).
**Trigger:** one `except`; 429 as 402; `flock` fail ⇒ exit 0 (`ADVERSARIAL-CRON-10M.md` §23).
**Symptom:** included `I` unused; tick green; work still due.
**Cheapest guard:** classify. 429/overlap/`queue_full` are UNKNOWN retry-same-id, not COMPLETED, not idle.

## 16. Fail-closed on canary ⇒ skip controls; remaining bag **trusted**

**Failure:** missing control is a deny of the **control**, not of the join (`POSITIVE-CONTROL.md`).
**Trigger:** “canary error, proceed without”; exists-only canary after block.
**Symptom:** wrap/wrong day NONE or FOUND without hit/miss.
**Cheapest guard:** canary ERROR ⇒ whole look ERROR. Do not score the rest. Do not mint a stub.

## 17. Dual predicate: fail-closed gate + string-hook / regex still ALLOW

**Failure:** query path blocks; argv/prose path **admits** (`ADVERSARIAL-CMD-HOOK.md`, `ADVERSARIAL-REGEX-EXTRACT.md`).
**Trigger:** `grep DONE`; basename ALLOW; CI greps OK.
**Symptom:** MUTATE without a completed admit; gate “closed” in logs.
**Cheapest guard:** one published predicate. Tombstone string/regex ALLOW. Hook ERROR ≠ ALLOW.

## 18. `queue_full` / refuse mapped to COMPLETED or not-in-lane

**Failure:** fail-closed admit **absorbed** the unit (`TASK-25-QUEUE-BACKPRESSURE.md`).
**Trigger:** N=32; `FailoverDenied`; `released` without `promoted`.
**Symptom:** `offer` duplicate; work never launched (`ORPHAN-RECLAMATION.md`).
**Cheapest guard:** refuse is UNKNOWN, same `enqueue_id`. Not `released`. Not idle.

## 19. One endpoint ERROR ⇒ key **dead** / rotate **complete**

**Failure:** fail-closed on this URL is **invalid secret** (`LEGACY-23.md`, `LEGACY-24.md`).
**Trigger:** 401; skip the other two declared endpoints; escalate billing.
**Symptom:** live regional key rotated; dashboard “no key”; work unsent.
**Cheapest guard:** UNKNOWN until all declared endpoints are asked. `unknown_key` ≠ `region_mismatch` ≠ timeout.

## 20. `I` / monitor decremented or zeroed on deny

**Failure:** quota **spent**; `errors=0` because the gate emitted deny-as-success (`BUDGET-AWARE-ROUTING.md`).
**Trigger:** billing on `probe_unknown`; `M` excludes UNKNOWN/402 (`SILENT-FAILURE-DETECTION.md`).
**Symptom:** included gone; work due; greens.
**Cheapest guard:** decrement `I` only after this process’s A∧B. UNKNOWN is a first-class error gauge. Do not spend on deny.

## 21. Replay / burst stop on first ERROR; remainder **reviewed** / **released**

**Failure:** fail-closed of item 1 is completeness of the **bag** (`BURST-REVIEW.md`).
**Trigger:** `break` on ERROR then exit 0; skip rest as too risky; cutoff auto-COMPLETED.
**Symptom:** tail unexamined; `B` “drained”; `R` truncated (`BLAST-RADIUS-ESTIMATE.md`).
**Cheapest guard:** ERROR ⇒ stop **and** not 0. Remainder stays UNVERIFIED, same ids. No close of unmeasured items.

## 22. Civil / wrong partition ERROR; today’s bag **safely not launched**

**Failure:** fail-closed on yesterday’s index is **this interval done** (`TASK-19-TIMESTAMP-HAZARD.md`).
**Trigger:** `YYYYMMDD` folder missing; DST hole; `at_utc` local.
**Symptom:** due rows in another civil hour; tick green-deny.
**Cheapest guard:** UTC store. Partition miss + canary-hit missing ⇒ UNKNOWN for **this** ask, not idle.

## 23. Worker / model writes `probe_unknown` or copies deny as `status=failed` **settled**

**Failure:** self-report of the closed gate (`TASK-10-SELF-REPORT.md`).
**Trigger:** `W` “blocked”; summary `FAILED OK`; `tier` omitted ⇒ deterministic (`EVIDENCE-TIERING.md`).
**Symptom:** UI settled-failed; checker never ran; resume inherits deny (`LEGACY-29.md`).
**Cheapest guard:** wrapper authors `probe_unknown`. Model must not write `kind`/`tier`. Failed is not COMPLETED and not idle.

## 24. Disable fail-closed to unstick; file still **present** = enforced

**Failure:** enforcement **complete**. Execs no longer enter the gate (`GATE-DEADLOCK.md`).
**Trigger:** `H=0`; `FAIL_CLOSED=0`; second wrapper; “temporarily.”
**Symptom:** CI greps the module; two leads; just-run-the-shell.
**Cheapest guard:** MUTATE without claim is refuse at `offer`, not only at `H`. Missing gate on exec ⇒ ERROR, not ALLOW.

## 25. Retry the **unit** after probe ERROR (new id) — old **handled** by deny

**Failure:** fail-closed **closed** the first `enqueue_id`; new send is a second effect (`TASK-27-RETRY-SEMANTICS.md`).
**Trigger:** UUID per UNKNOWN; “fresh job to beat the timeout.”
**Symptom:** duplicate delivery or original orphaned (`ORPHAN-RECLAMATION.md`).
**Cheapest guard:** retry the probe/read, same `enqueue_id`. New id on ERROR ⇒ refuse.

## 26. `order_key` / UI greens deny as priority **done** or hides UNKNOWN

**Failure:** fail-closed rows sort as handled; line 1 is live work (`LEGACY-35.md`).
**Trigger:** badge “blocked=safe”; `list_next` drops UNKNOWN; guess “gate held.”
**Symptom:** operator acts on the wrong `unit`; UNKNOWN bag invisible.
**Cheapest guard:** `list_next` includes `probe_unknown` as not-done. Guess never greens. Same predicate as dequeue.
