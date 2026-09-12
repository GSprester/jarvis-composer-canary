# adversarial review: retry loop relaunching quota-failed jobs

Mechanism: a wrapper re-`on_launch`es units after **402** / quota-exhaustion / “payment required” (and often after **429** misfiled as the same). Success is typically **HTTP 200**, **retry count exhausted**, **dead-lettered**, **hopped**, or **`I` decremented**. Confident wrong = **success, completeness, or safety** that does not hold.

## 1. 402 retried until 200 / exists / max, then COMPLETED

**Failure:** entitlement refused the work. The loop’s last bit is **closed**. Bytes never met `expected`, or never sent under a live grant.
**Trigger:** `402 ⇒ retry same pair` (`BUDGET-AWARE-ROUTING.md`); `test -f` after a stub (`ADVERSARIAL-EXISTS-SCANNER.md`); `max_retries` then 0.
**Symptom:** unit COMPLETED; provider still 402; `R` = every retry from that writer (`BLAST-RADIUS-ESTIMATE.md`).
**Cheapest guard:** 402 / quota body ⇒ `do_not_retry` + escalate (`TASK-28-ERROR-CLASSIFICATION.md`). COMPLETED only A∧B (`DUAL-SIGNATURE-COMPLETION.md`). Exhaustion ≠ 0.

## 2. 402 ⇒ new `unit` / new `enqueue_id`

**Failure:** old id **handled**; new id is “the retry.” Two effects or one lost original.
**Trigger:** UUID per attempt; content hash includes `at_utc` or seat (`ORPHAN-RECLAMATION.md`); “fresh send to beat quota.”
**Symptom:** original `released`/dead-lettered; second shell or second 402; dedup cannot un-run (`TASK-27-RETRY-SEMANTICS.md`).
**Cheapest guard:** same `enqueue_id` / `re`. 402 does not mint identity. New id ⇒ refuse.

## 3. Timeout / UNKNOWN after send, then resend as “quota retry”

**Failure:** first accept may have run (and billed). Second send is **successful retry**. Two shells.
**Trigger:** t3 UNKNOWN (`TASK-27-RETRY-SEMANTICS.md`); 402 HTML truncated (`LEGACY-26.md`); connection reset after accept.
**Symptom:** duplicate delivery; `I` burned twice or once with two artifacts.
**Cheapest guard:** UNKNOWN ⇒ read for receipt/`promoted`, do not send. Timeout is not “no effect.” Same token if the peer keys it.

## 4. 429 hopped to 402 / overage / another pair as success

**Failure:** rate-limit is **time**. Hop reports **failover OK**. Remaining `I` on the limited pair sits unused; overage bills or a new pair 402s.
**Trigger:** cheapest-first (`BUDGET-AWARE-ROUTING.md`); SDK “retry other key”; 429 body contains “quota.”
**Symptom:** included inversion; overnight `p0` pays; original pair still has `I`.
**Cheapest guard:** 429 ⇒ backoff, same pair, same `enqueue_id`. Not a hop. Quota marker on 429 ⇒ classify 402 only if MEASURE of `I` completed at 0.

## 5. `I` decremented on attempt, 402, or HTTP 200

**Failure:** quota **spent**. Safety of accounting. Work UNVERIFIED or never accepted.
**Trigger:** decrement on `promoted`; on send; on 200; on each retry (`BUDGET-AWARE-ROUTING.md`).
**Symptom:** `I=0`; retries then “no entitlement” complete (§8); included gone.
**Cheapest guard:** decrement only after this process’s Sig1∧Sig2. 402/429/UNKNOWN do not decrement.

## 6. Quota probe ERROR ⇒ `I=0` ⇒ bag **handled** (nothing to send)

**Failure:** failed look looks like no entitlement (`POSITIVE-CONTROL.md`). Completeness of the retry drain.
**Trigger:** empty body ⇒ NONE; timeout ⇒ 0; canary-hit missing.
**Symptom:** 402-class jobs sit; loop exit 0; “no errors” (`SILENT-FAILURE-DETECTION.md`).
**Cheapest guard:** probe ERROR ⇒ ineligible this pop, not idle. Missing `I` is not 0. Canary-hit required.

## 7. Probe ERROR ⇒ `I=∞` ⇒ retry “entitled,” then 200 as delivered

**Failure:** you route into 402 or a billed send. Loop reports **launched/OK**.
**Trigger:** missing table ⇒ unlimited; cache stale after silent reset (`LEGACY-27.md`).
**Symptom:** 402 spin or overage; `I` never measured; COMPLETED on 200.
**Cheapest guard:** missing/`ERROR` `I` ⇒ do not send. Not ∞. 200 is not A∧B.

## 8. `do_not_retry` / escalate / dead-letter = complete

**Failure:** disposition **closed** the unit. Escalation is not delivery.
**Trigger:** classify `quota_or_payment`; ticket opened; `released` with `kind=escalated`; `P` ACK (`BURST-REVIEW.md`).
**Symptom:** `list_next` omits `qid`; `offer` duplicate; work OPEN.
**Cheapest guard:** escalate is typed ERROR, unit stays UNVERIFIED. No `released` without Sig1∧Sig2 or `stale_archived` of a **promoted** attempt.

## 9. Misclassify 429 ↔ 402 (or 401 region as quota)

**Failure:** tight loop on entitlement, or hop/rotate on a live limited key. Each path reports **correct handling**.
**Trigger:** one HTTP code (`LEGACY-23.md`); 429 body “quota exceeded”; 402 without marker treated as 500.
**Symptom:** retry storm; or key rotated while us-east still has `I`; dashboard “no key.”
**Cheapest guard:** 402/quota MEASURE of `I`. 429 is time. `unknown_key` ≠ `region_mismatch` ≠ quota. Unclassified ⇒ raise, not retry (`TASK-28-ERROR-CLASSIFICATION.md`).

## 10. Exists leftover after a refused send ⇒ skip retry as done

**Failure:** presence is **already handled** (`LEGACY-05.md`). First 402 still stands.
**Trigger:** `W` touched `out/`; previous run stub; change-detector (`LEGACY-09.md`).
**Symptom:** loop stops; digest ≠ `expected`; consumer presence-gates (`PROVENANCE-ARCHIVE.md`).
**Cheapest guard:** skip only A∧B. Exists after 402 ⇒ archive candidate, not COMPLETED.

## 11. Router self-grants overage to clear the retry bag

**Failure:** `O=1` without exemption. Tick **succeeded**. Policy would have denied.
**Trigger:** “keep the cheap seat warm”; burst catch-up (`BURST-REVIEW.md`); in-memory grant (`LEGACY-22.md`).
**Symptom:** billed; `reserve` bypassed; no `would_deny` row.
**Cheapest guard:** overage is a flushed exemption row (`I==0`, grantor, `policy_sha256`). Router cannot grant itself.

## 12. Failover hop raises ceiling

**Failure:** retry **admitted** on a wider `provider.class`. Safety of `min(seat, provider)`.
**Trigger:** other key after 402; cheapest public vendor (`TASK-13-FAILOVER-CEILING.md`).
**Symptom:** restricted WO on public; hop looks like quota fix.
**Cheapest guard:** every retry `admit` same as cold start. Deny ⇒ not a retry target. Ceiling never rewritten.

## 13. Cached `I` / rolled window: retry thinks entitled

**Failure:** yesterday’s `I>0` or a silent reset. Send **OK** then 402, or silent overage **success**.
**Trigger:** cache across UTC day (`LEGACY-27.md`); decrement local only; provider window reset not measured.
**Symptom:** loop greens; provider 402; or `I` table ≠ billed count.
**Cheapest guard:** new MEASURE of `I` before every send. Cache invalid at day boundary and after any 402.

## 14. Retry burns `reserve` (or inherit `P`’s `I`)

**Failure:** `p3` quota-retry spends tokens held for aged `p0`, or `S` spends `P`’s included. Queue **cleared**.
**Trigger:** `usable` ignored on retry path; cover inherits (`BUDGET-AWARE-ROUTING.md`).
**Symptom:** overnight `p0` overage; primary’s `I` gone while `P` offline.
**Cheapest guard:** retry uses `usable(p, prio)` from a new MEASURE. Substitutes use **their** `I`/`reserve`.

## 15. SDK inner retry × wrapper retry

**Failure:** N accepts, one wrapper **success**. Or SDK already 402-looped, wrapper records one attempt **safe**.
**Trigger:** vendor client retries 402/429; wrapper retries UNKNOWN (`LEGACY-28.md`).
**Symptom:** duplicate delivery or hidden 402 storm; attempt count = 1.
**Cheapest guard:** one retry authority. SDK retries off. Wrapper sees the raw status. Attempt count = wire attempts.

## 16. 402 then `released` / orphan-reclaim as never-launched

**Failure:** `qid` **done** or **safely re-queued**. Send may have accepted or billed.
**Trigger:** no `promoted` visible; timeout; `P` ACK (`ORPHAN-RECLAMATION.md`).
**Symptom:** two launches after reclaim; or work gone from `list_next`.
**Cheapest guard:** 402 after send is not orphan. UNKNOWN `same` ⇒ no reclaim. `released` only Sig1∧Sig2 or verified `stale_archived`.

## 17. DESIGN crash-loop called ACCIDENT (or 402 storm called DESIGN)

**Failure:** poison **retried now**, or a quota outage **braked** as complete/safe skip.
**Trigger:** three `W` failures (`CRASH-LOOP-BRAKE.md`); `n` = “errors in the last hour”; 402×3 ⇒ `crash_loop`.
**Symptom:** tight loop of the same 402; or bag frozen while `I` returns tomorrow.
**Cheapest guard:** DESIGN only wrapper `crash_loop` after `blocked→interrupted` ≥3, not HTTP class. 402 ⇒ escalate, not brake-as-done. ACCIDENT ≠ retry-now of 402.

## 18. False-positive “quota” substring on 200 / 500

**Failure:** delivered or server-error work **escalated/closed** as payment.
**Trigger:** body scan `quota exceeded` in HTML/docs (`TASK-28-ERROR-CLASSIFICATION.md`); 200 + marketing footer.
**Symptom:** unit dead-lettered after success; or 500 never retried.
**Cheapest guard:** quota class needs status 402 **or** a completed `I==0` MEASURE, not a substring alone.

## 19. Monitor filters 402: retry loop **healthy**

**Failure:** `errors=0` while the loop only produces 402/UNKNOWN (`SILENT-FAILURE-DETECTION.md`).
**Trigger:** `M` counts 5xx; 402 excluded; last_ok touched each attempt (`ADVERSARIAL-CRON-10M.md` §2).
**Symptom:** dashboard green; backlog 402; `last_successful_scan` fresh.
**Cheapest guard:** 402 is a first-class error gauge. Alert if `I` MEASURE stale or 402 rate > 0 without escalate row.

## 20. Cheapest exhausted key: loop “working,” other `I` idle

**Failure:** progress = attempts on the $0 pair. Safety of cheapest-first.
**Trigger:** `unit_price` sort; 402 as cheap (`BUDGET-AWARE-ROUTING.md`); key called dead (`LEGACY-23.md`).
**Symptom:** retry storm on pair A; pair B `I>0` unused; jobs “retried.”
**Cheapest guard:** pick `argmin` among `usable>0` only. 402 ⇒ that pair ineligible until new `I>0` MEASURE. Do not rotate the secret.

## 21. Partial bag: first retry 200, rest 402, tick complete

**Failure:** interval **done**. Completeness of the loop, not of the bag.
**Trigger:** `break` on first 0; `max_launch=1`; budget after one decrement (§5).
**Symptom:** tail still quota-failed; cron 0 (`ADVERSARIAL-CRON-10M.md` §5).
**Cheapest guard:** exit 0 only if MEASURE shows no due **and** canary-hit. Else typed `REMAINING` / 402 ERROR ≠ 0.

## 22. `queue_full` after 402 = queued / OK

**Failure:** nothing relaunched; tick **absorbed**.
**Trigger:** N=32 (`TASK-25-QUEUE-BACKPRESSURE.md`); admit deny; 402 then enqueue fail mapped to 0.
**Symptom:** backlog grows; loop green.
**Cheapest guard:** `queue_full` is UNKNOWN, same `enqueue_id`. Not success. Not a new send.

## 23. `W` `status=completed` after quota fail (or after hop)

**Failure:** worker bit copied as **closed** (`EVIDENCE-TIERING.md`).
**Trigger:** HTTP 200 body “ok”; local gate (`DEPENDENCY-INVERSION.md`); model “retried.”
**Symptom:** COMPLETED; file empty; retries skip (`LEGACY-31.md`).
**Cheapest guard:** cron/loop 0 never from `model_guess`. Only checker Sig2.

## 24. Attempt count / backoff sleep as liveness / safety

**Failure:** `n` rising or `last_ok` fresh ⇒ **making progress**, holder **safe**.
**Trigger:** retry-now (no backoff) on 402; touch marker each loop; mtime (`LEGACY-14.md`).
**Symptom:** crash-loop looks live; monitor quiet; no Sig1.
**Cheapest guard:** progress is A∧B or a completed `I` MEASURE that changes disposition. Attempt count is not delivery. 402 must backoff-and-escalate, not retry-now.

## 25. Exemption / grant_id minted per retry as “approved”

**Failure:** each attempt a new grant. Safety of **human** overage. Or one grant reused after expiry as still **valid**.
**Trigger:** `grant_id` = UUID per hop; expired `expires_at_utc` ignored; model writes the row (`LEGACY-22.md`).
**Symptom:** unbounded overage; or denied policy with a stale grant file exists (`LEGACY-05.md`).
**Cheapest guard:** one stable `grant_id` per override; `would_deny` recomputed; expired ⇒ not a retry license. Wrapper writes, not `W`.

## 26. 403/401 after 402 retry classified as delivered-or-dead-key complete

**Failure:** bind fail or forbid **closed**. Quota loop “finished the job.”
**Trigger:** hop region; 403 `do_not_retry`; 401 `invalid_api_key` (`LEGACY-23.md`, `TASK-28-ERROR-CLASSIFICATION.md`).
**Symptom:** unit gone from queue; key rotated; work never ran in the declared region.
**Cheapest guard:** 403 escalate UNVERIFIED. 401 split `unknown_key` vs `region_mismatch`. Neither is COMPLETED. Same `enqueue_id` waits on a completed bind MEASURE.
