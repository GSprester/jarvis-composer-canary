# adversarial review: claims ledger with `opened` never closed

Mechanism: a ledger (or `review/` JSONL) records `opened` / `claimed` / `inflight` and **has no close path** — or `review_closed` / `released` never runs. Success is typically **`opened` exists**, **still open = in progress**, **`open_units>0` = live**, or **empty opened = idle**. Confident wrong = **success, completeness, or safety** that does not hold.

## 1. `opened` = live / safe / one-writer

**Failure:** a row is a **holder**. OS incarnation never checked (`LEGACY-37.md`).
**Trigger:** no `pid+start`; pid-only (`ADVERSARIAL-PID-ONLY-LIVENESS.md`); mtime fresh (`LEGACY-14.md`).
**Symptom:** steal refused; unit blocked for days; or a recycled pid “holds” it (`LEGACY-01.md`).
**Cheapest guard:** live iff `same_process` after a completed probe. `opened` without `{pid,start}` is not a claim. Dead/recycle ⇒ stale-archive, not safe.

## 2. `opened` = **done** (no close state, so open is the terminal)

**Failure:** the only status is treated as COMPLETED / covered (`SUBSTITUTE-REVIEW-QUEUE.md`).
**Trigger:** `review_opened`⇒closed (`ADVERSARIAL-SELF-REVIEW-QUEUE.md` §2); schema has no `released`; UI maps open→green.
**Symptom:** stub CLOSED; A∧B never ran (`LEGACY-32.md`); `R` = every opened (`BLAST-RADIUS-ESTIMATE.md`).
**Cheapest guard:** `opened` is a claim, `tier=inferred` at best. Close only checker A∧B. Missing close type ⇒ units stay UNVERIFIED, not COMPLETED.

## 3. Silence + `opened` = still working

**Failure:** no `closed` event is **progress** (`LEGACY-37.md`). Never-tried, dead, and quiet-live are one wire.
**Trigger:** long job; drain; failed look mapped to quiet (`LEGACY-03.md`); never `promoted`.
**Symptom:** monitor “in progress”; `list_next` skips; bag never advances (`LEGACY-09.md`).
**Cheapest guard:** quiet_and_live only after `same` ∧ completed healthy probe. Missing close after a missing look is UNKNOWN, not progress.

## 4. `open_units>0` = estate **healthy** (lying non-zero)

**Failure:** cardinality of `opened` is **liveness** (`TASK-34-METRIC-THAT-LIES.md`).
**Trigger:** wrap keeps a few rows; observer counts itself (`LEGACY-13.md`); canary `opened` only.
**Symptom:** dashboard busy/safe; holders dead; or wrap already dropped the incident.
**Cheapest guard:** pair count with `last_successful_scan`, `same` of each head, and archive count. `>0` is not a completed MEASURE.

## 5. Empty / failed look at `opened` = **idle** / all closed

**Failure:** no rows = **complete** (`POSITIVE-CONTROL.md`).
**Trigger:** timeout; wrap (`TASK-22-ROLLING-WINDOW-AUDIT.md`); `/tmp` queue (`SUBSTITUTE-REVIEW-QUEUE.md`); `P`-only writer offline.
**Symptom:** `open_units=0`; covers invisible; cron 0 (`SILENT-FAILURE-DETECTION.md`).
**Cheapest guard:** look ERROR ⇒ not idle. Canary-hit missing ⇒ ERROR. Wrap without archive ⇒ UNKNOWN, not 0.

## 6. Skip every `opened` as inflight — ticks **OK**

**Failure:** auto-start / burst **did its job** (launched 0) (`ADVERSARIAL-ONE-PER-TICK.md` §13).
**Trigger:** `if status==opened: skip`; overlap; pid recycle “still running.”
**Symptom:** greens; no new `promoted`; leftover blocks relaunch (`STALE-CLAIM-RECOVERY.md`).
**Cheapest guard:** skip only `same` live on **that** `unit`. Else archive+open or ≠ 0. `opened` is not inflight.

## 7. Presence of hot / `opened` JSON blocks relaunch **and** is treated as handled

**Failure:** accident brake **safe** (`CRASH-LOOP-BRAKE.md`, `LEGACY-05.md`).
**Trigger:** `C` `exists`; `prior receipt no claim`; never close so never archive.
**Symptom:** dispatcher refuses; consumer FOUND; unit stuck UNVERIFIED.
**Cheapest guard:** `opened` without live `{pid,start}` and without A∧B ⇒ archive, not COMPLETED, not skip-forever.

## 8. Worker writes `opened` and never a close — self-cover **recorded**

**Failure:** author of the work is author of the **claim** (`TASK-10-SELF-REPORT.md`).
**Trigger:** `S`/`W` dumps `review_opened` + `status=covered`; no checker process (`TASK-17-SEPARATION-OF-DUTIES.md`).
**Symptom:** bag “has coverage”; bytes unverified; `P` ACKs the row (`BURST-REVIEW.md`).
**Cheapest guard:** wrapper opens after lock; only checker closes. `W` cannot write the ledger. Opened ≠ covered.

## 9. `P` ACK / mailbox / chat without a close row

**Failure:** asserted drain. Ledger still `opened` **or** implied closed by absence of a type (`EVIDENCE-TIERING.md`).
**Trigger:** burst is read-chat; unlink inbox (`ADVERSARIAL-MAILBOX-QUEUE.md`); no `review_closed` symbol.
**Symptom:** operators done; `B` still opened; or rows deleted so `R` dies (`BURST-REVIEW.md`).
**Cheapest guard:** ACK is `model_guess`. Do not delete `opened`. Close is A∧B only.

## 10. Decrement `I` / `promoted` accounting on `opened`

**Failure:** quota **spent**. Safety of a claim (`BUDGET-AWARE-ROUTING.md`).
**Trigger:** billing on open; `I` on `review_opened`; HTTP 200 of the cover.
**Symptom:** included gone; never A∧B; rest unstarted.
**Cheapest guard:** decrement only after this process’s Sig1∧Sig2. `opened` does not spend.

## 11. Collapse by `unit` / delete `opened` to “drain”

**Failure:** two `enqueue_id`s, one **gone**. Completeness of the ledger (`TASK-26-DEDUPE-BY-CONTENT.md`).
**Trigger:** no close so they compact; wrap; “go faster.”
**Symptom:** second cover vanished; next look never-issued (`STALE-CLAIM-RECOVERY.md`).
**Cheapest guard:** identity is `review_id` / `enqueue_id`. Do not delete `opened`. Compact only after verified archive.

## 12. Cutoff / age / wrap auto-completes never-closed rows

**Failure:** time **closed** them (`BURST-REVIEW.md`).
**Trigger:** 72h civil; mtime; hot cap evicts `opened` as if `released`.
**Symptom:** `open_units` 0; work UNVERIFIED; first crash never retries.
**Cheapest guard:** eviction is archive, not COMPLETED. Cutoff leaves `opened` `reason=cutoff`, same `unit`.

## 13. Two `opened` for one `unit` — both **safe** / first-wins **the** claim

**Failure:** two leads or one cover dropped (`LEGACY-01.md`).
**Trigger:** `S`+`P`; new `enqueue_id` while old still open; prefix `review_id` (`TASK-07-DATE-ROLLOVER.md`).
**Symptom:** split writes; or `list_next` shows one, the other invisible.
**Cheapest guard:** one published lock per `unit`. Second `opened` only after `same` false + archive. `n≠1` ⇒ ERROR.

## 14. `opened` without lock publish / before fsync

**Failure:** row **in-flight**. No holder, torn bytes (`IDEMPOTENT-EVENT-LEDGER.md`).
**Trigger:** model writes first; HWM ahead; promote-on-enqueue.
**Symptom:** skip as inflight (§6); `qid` never `promoted`; ghost covers.
**Cheapest guard:** `opened`/`promoted` only after lock `{pid,start}` flush. No row ⇒ not inflight.

## 15. Fail-closed on the close query; `opened` stays — **gate held**

**Failure:** cannot `verify`/`find` ⇒ never close ⇒ **safe deny** (`ADVERSARIAL-FAIL-CLOSED.md` §2).
**Trigger:** hook on MEASURE (`GATE-DEADLOCK.md`); timeout; UNKNOWN⇒skip.
**Symptom:** permanent `opened`; 0 closes as policy; work due.
**Cheapest guard:** MEASURE free. Close-path ERROR ⇒ `probe_unknown`, not healthy-open. Retry the read, same id.

## 16. Consumer / `C` takes `exists` because ledger will never `released`

**Failure:** side path **completes** what `opened` cannot (`PROVENANCE-ARCHIVE.md`).
**Trigger:** no `released` type; `C` globs hot; mailbox ACK.
**Symptom:** later file consumed; head still `opened`; leapfrog (`ADVERSARIAL-EARLIEST-PROMOTE.md`).
**Cheapest guard:** `C` waits on ledger head + A∧B. Presence is not close. Add a real `released` after Sig2.

## 17. `list_next` hides `opened` (inflight) or greens it (done)

**Failure:** line 1 is **next**. Action hits another `unit` (`LEGACY-35.md`).
**Trigger:** filter `status!=opened`; badge open=covered; sort by `opened_at`.
**Symptom:** operator cancels the wrong id; `p0` buried; trickle (`LEGACY-09.md`).
**Cheapest guard:** `list_next` uses `order_key` + `same`/`A∧B`, not status spelling. `opened` is not done and not skip-all.

## 18. Reclaim / new `unit` because the old one is still `opened`

**Failure:** old **handled** (stuck open); new id **progress** (`TASK-27-RETRY-SEMANTICS.md`).
**Trigger:** UUID to “unstick”; timeout ⇒ orphan (`ORPHAN-RECLAMATION.md`).
**Symptom:** two shells; or original never launched and cannot re-offer.
**Cheapest guard:** same `enqueue_id`. `opened` without `promoted` is `ORPHAN`, not done. New id ⇒ refuse.

## 19. Civil window / archive omitted: old `opened` **out of bag**

**Failure:** that window **complete** (`TASK-19-TIMESTAMP-HAZARD.md`).
**Trigger:** last-three-local-days; hot-only; wrap drops opened.
**Symptom:** `B=∅`; archive has opened; burst “nothing to review.”
**Cheapest guard:** UTC bag + archive. Failed list ⇒ ERROR. Include never-closed rows older than W as UNVERIFIED, not gone.

## 20. `opened` row appended onto the payload

**Failure:** digest **moved**; `C` gates; ledger **durable** (`LEGACY-21.md`).
**Trigger:** “keep claim with the work”; no close file so they annotate `out/<unit>`.
**Symptom:** A fails; `opened` still true; citations orphan.
**Cheapest guard:** ledger outside the payload. Do not append status onto bytes.

## 21. DESIGN / crash-loop inferred from never-closed count

**Failure:** `n` = days `opened` or “errors in the last hour” (`CRASH-LOOP-BRAKE.md`).
**Trigger:** no close ⇒ count grows; `W` three `failed`; ACCIDENT called DESIGN.
**Symptom:** bag frozen **safe**; or poison retried if called ACCIDENT.
**Cheapest guard:** DESIGN only wrapper `crash_loop` after `blocked→interrupted` ≥3. Age of `opened` is not `n`.

## 22. Canary `opened` / self-as-sample blesses the scan

**Failure:** control **passed**. Remaining never-closed rows trusted (`LEGACY-13.md`, `LEGACY-39.md`).
**Trigger:** scanner inserts `opened` so the look is non-empty; exists-only canary.
**Symptom:** wrap/timeout looks healthy; real covers missing.
**Cheapest guard:** canary is issuer A∧B, not this process’s row. Subtract `{self.pid,start}`. Missing hit ⇒ ERROR.

## 23. Monitor `errors=0` because `opened` is not ERROR

**Failure:** never-closed is **success** of the instrument (`SILENT-FAILURE-DETECTION.md`).
**Trigger:** `M` counts 5xx only; last_ok at open (`ADVERSARIAL-CRON-10M.md` §2).
**Symptom:** greens; holders dead; `last_successful_scan` stale or start-touched.
**Cheapest guard:** gauge age of oldest `opened` without `same` and without A∧B. Alert on stale MEASURE, not on `errors==0`.

## 24. Second burst / successor mints new `opened` ids; old never-closed **forgotten**

**Failure:** new bag **complete**. Old rows still open or wrapped (`BURST-REVIEW.md`).
**Trigger:** new `review_id`s; harness retirement (`LEGACY-29.md`); chat id identity.
**Symptom:** leftover `B` invisible; duplicate covers; `R` split.
**Cheapest guard:** same `review_id`/`enqueue_id` across bursts. Successor reads the estate ledger, not memory.

## 25. `released` / Sig2 implemented as “leave `opened`” (close is implicit)

**Failure:** implicit close is **delivery** (`IDEMPOTENT-EVENT-LEDGER.md`).
**Trigger:** “we don’t write closed”; consumer infers done from age or `exists`; `kind=sent`.
**Symptom:** `offer` no-ops; no `delivery.evidence_sha256`; third party cannot re-derive (`LEGACY-06.md`).
**Cheapest guard:** close is an explicit `released`/`review_closed` after this process hashed. Implicit is not A∧B.

## 26. UI / `order_key` treats never-closed as priority **working** or **finished**

**Failure:** the badge is **status** (`EVIDENCE-TIERING.md`).
**Trigger:** “model is sure”; open=green; hide open from `list_next`; inherit on resume (`LEGACY-29.md`).
**Symptom:** wrong first row; COMPLETED badge on UNVERIFIED; live `p0` buried (`LEGACY-35.md`).
**Cheapest guard:** `list_next` by WO `order_key`. Guess never greens. Resume is cold admit; `opened` dies with the seat unless A∧B.
