# adversarial review: review queue a worker writes its own rows into

Mechanism: `W` / substitute `S` / primary model appends `review_opened` / `review_closed` / `status=covered` about **its own** cover (`SUBSTITUTE-REVIEW-QUEUE.md`). Success is typically **row exists**, **`P` ACK**, **empty `review/`**, or **burst drained**. Confident wrong = **success, completeness, or safety** that does not hold.

## 1. Worker / `S` writes `review_closed`

**Failure:** author of the cover is author of **closed** (`LEGACY-17.md`). Gate self-satisfied.
**Trigger:** `S` model `status=covered`; wrapper copies `W`’s bit; `review_closed` as Sig2 (`DUAL-SIGNATURE-COMPLETION.md`).
**Symptom:** COMPLETED; digest ≠ `expected` or empty; next ticks skip (`LEGACY-09.md`).
**Cheapest guard:** only the checker process writes `review_closed`, after this process’s A∧B. `S`/`W` API has no close path (`TASK-17-SEPARATION-OF-DUTIES.md`).

## 2. `review_opened` alone ⇒ closed

**Failure:** a claim is a **check** (`EVIDENCE-TIERING.md`). Opened is asserted.
**Trigger:** `if exists(review/id): COMPLETED`; `P` treats opened as covered; inferred close.
**Symptom:** stub/`DONE` CLOSED; `R` = every opened from that writer (`BLAST-RADIUS-ESTIMATE.md`).
**Cheapest guard:** opened is inferred at best, never `released`/COMPLETED. Close iff A∧B ∧ opened row cites the **same** `expected`.

## 3. `P` ACK / mailbox / chat grades the row

**Failure:** a sentence is **review_closed** (`BURST-REVIEW.md`).
**Trigger:** owner types ACK; mailbox unlink (`ADVERSARIAL-MAILBOX-QUEUE.md`); burst is “read chat.”
**Symptom:** bag drained; no MEASURE; leftovers presence-gate (`LEGACY-05.md`).
**Cheapest guard:** ACK is `model_guess`. `P` may MEASURE only. Close is checker Sig2, not chat.

## 4. Worker mints `expected` / `unit` / `path` on the row it opens

**Failure:** the join is authored by the cover (`LEGACY-17.md`). Row **matches**.
**Trigger:** `expected` from `W` hash; `unit=name[:8]`; `path` from notes (`ADVERSARIAL-REGEX-EXTRACT.md`).
**Symptom:** template/empty digest closes; 494-style bind (`RECEIPT-MATCHING.md`).
**Cheapest guard:** fields copied from the **pre-declared** WO, not from `W`. Backfill ⇒ DISCARD. Empty-hash refuse.

## 5. `exists(out/<unit>)` while `P` was away ⇒ closed

**Failure:** leftover / `S` stub is **the** cover (`LEGACY-05.md`).
**Trigger:** presence gate; `C` globs hot; burst “file is there.”
**Symptom:** UNVERIFIED bytes CLOSED; accident brake (`CRASH-LOOP-BRAKE.md`).
**Cheapest guard:** presence is a candidate. Hash == `expected` (≠ empty) **and** checker receipt. Archive leftovers.

## 6. Empty `review/` = nothing to review / covers never happened

**Failure:** failed look or invisible queue is **complete** (`POSITIVE-CONTROL.md`).
**Trigger:** timeout; `P` is the only writer (offline ⇒ no rows); `/tmp`/laptop/mailbox (`SUBSTITUTE-REVIEW-QUEUE.md`).
**Symptom:** `unmatched=0`; days of cover gone; cron 0 (`SILENT-FAILURE-DETECTION.md`).
**Cheapest guard:** look ERROR ⇒ not idle. Canary-hit missing ⇒ ERROR. Queue lives in the estate tree. `P`-only writer is a design defect.

## 7. Fuzzy / notes / regex of `S` prose names the WO closed

**Failure:** similarity is the review (`ADVERSARIAL-RECEIPT-NAMES.md`).
**Trigger:** stem; “we handled it”; checkbox; `DONE` in `S` chat (`ADVERSARIAL-SELF-SUMMARY.md`).
**Symptom:** 74-style prefix close; canary-miss FOUND.
**Cheapest guard:** no fuzzy. `re==unit` + this process’s hash. Prose is `model_guess`.

## 8. `P` steals a live `S` claim / “takes back”

**Failure:** safety of **stale**. Two leads or cover deleted (`LIVENESS-PREDICATE.md`).
**Trigger:** mtime (`LEGACY-14.md`); pid-only; UNKNOWN ⇒ dead; `P` ACK as reclaim.
**Symptom:** split-brain writes (`LEGACY-01.md`); `S` still writing; `P` CLOSED the row.
**Cheapest guard:** steal only `same(S)==false` after a completed probe. UNKNOWN ⇒ skip item, not close. `P` does not reclaim live covers.

## 9. Collapse by `unit` / delete `review_opened` to drain

**Failure:** two `enqueue_id`s, one **reviewed**. Or `R` uncomputable (`BURST-REVIEW.md`).
**Trigger:** dedupe on `unit`; “go faster”; wrap (`TASK-26-DEDUPE-BY-CONTENT.md`).
**Symptom:** second cover vanished; bag empty; incident unreplayable.
**Cheapest guard:** dedup by `review_id` only. Do not delete opened rows. Unverifiable stays UNVERIFIED.

## 10. Cutoff / 72h / `T_burst` ⇒ auto-COMPLETED or delete

**Failure:** time **closed** the item. Completeness of the burst.
**Trigger:** civil “three days”; mtime of the JSON (`LEGACY-14.md`); cutoff=1 item.
**Symptom:** remainder green; first crash never retries; live `p0` starved or never reached.
**Cheapest guard:** cutoff leaves `review_opened` `reason=cutoff`, UNVERIFIED, same `unit`. Age from `attempt_start` UTC, not mtime.

## 11. New `unit` / chat id / `P`+date as `review_id`

**Failure:** cover **handled** under a new identity. Original WO unnamed (`TASK-27-RETRY-SEMANTICS.md`).
**Trigger:** UUID per note; thread id; `YYYYMMDD` (`TASK-07-DATE-ROLLOVER.md`).
**Symptom:** two shells or invisible cover; `offer` duplicate on the real `qid`.
**Cheapest guard:** `review_id = sha256({unit, enqueue_id, substitute, attempt_start})`. Same `unit`. Chat is not an id.

## 12. Row appended onto `out/<unit>` (or the mailbox)

**Failure:** digest **moved**; `C` presence-gates; queue **durable** (`LEGACY-21.md`).
**Trigger:** “keep it with the work”; model context; `P`’s inbox.
**Symptom:** A fails; opened still “there”; offline drop (§6).
**Cheapest guard:** JSONL outside payload and outside harness prefix. Restore from estate path only.

## 13. `list_next` by who covered / `ls` / newest / live-first

**Failure:** line 1 is **next review**. Action pops another `unit` (`LEGACY-35.md`).
**Trigger:** sort by `S`; `opened_at`; UI live queue first (`BURST-REVIEW.md`).
**Symptom:** `P` reviews the wrong row; trickle; `p0` buried.
**Cheapest guard:** `order_key` on the **WO body**. Burst list = remaining `B` then live. `list_next()[0]==peek()`.

## 14. Inferred R2 / presence / `review_opened` used to `released`

**Failure:** lane head **popped** from a self-row (`ORDERED-RELEASE-QUEUE.md`, `EVIDENCE-TIERING.md`).
**Trigger:** burst `review_closed` ⇒ `released`; `kind=sent`; leapfrog later cover.
**Symptom:** earliest-promote dead; `offer` no-ops; bytes unverified.
**Cheapest guard:** `released` only after checker A∧B on a `may_release` head. Opened/R2 do not pop.

## 15. Cover inherits `P`’s ceiling or `I`

**Failure:** hop **admitted** / quota **spent** as `P` (`TASK-13-FAILOVER-CEILING.md`).
**Trigger:** `S` uses `P.ceiling`; decrement `I` on opened; overage to catch up (`BUDGET-AWARE-ROUTING.md`).
**Symptom:** restricted WO on public; included gone; burst greens.
**Cheapest guard:** `S` is `min(S.ceiling, provider)` and **S’s** `I`/`reserve`. Decrement only after checker A∧B.

## 16. `S` keeps opening while `P` is `same` live; detector skips

**Failure:** `B` grows; “review in progress” **safe** (`LEGACY-09.md`).
**Trigger:** substitutes do not stop; trickle 1/hour; change-detector on opened count.
**Symptom:** burst never drains; live work skipped; `C` treats leftovers as done.
**Cheapest guard:** once `P` is `same` live, `S` must not `review_opened` for `P`’s units. Opened is not a skip key.

## 17. `same(S)` pid-only / mtime — skip as still live **or** steal

**Failure:** recycle **live** (skip forever) or drain **dead** (steal) (`ADVERSARIAL-PID-ONLY-LIVENESS.md`).
**Trigger:** `kill(pid,0)`; days-old file; reboot small pids.
**Symptom:** covers blocked; or two leads; burst “safe.”
**Cheapest guard:** `same` iff pid+start (+ boot). UNKNOWN ⇒ do not close, do not steal.

## 18. `review_opened` before lock publish / fsync / wait

**Failure:** row **opened**. Cover never claimed or bytes torn (`TASK-31-GRACEFUL-SHUTDOWN.md`).
**Trigger:** model writes first; `print` then `Popen` (`ADVERSARIAL-SELF-SUMMARY.md`); HWM ahead.
**Symptom:** `P` reviews a ghost; `qid` never `promoted`; or two opens one lock.
**Cheapest guard:** wrapper appends opened **after** lock publish, flush, hwm. No row without `{pid,start}`.

## 19. Two covers share `enqueue_id` (or second burst mints new ids)

**Failure:** one attempt **reviewed** twice or a new id **replaces** the bag.
**Trigger:** two `S` one pop; retry UUID; second burst new `review_id`s (`BURST-REVIEW.md`).
**Symptom:** duplicate close or leftover `B` invisible next idle.
**Cheapest guard:** one `enqueue_id` ⇒ one `review_id`. Next burst reuses the same ids. New id after stale-archive only.

## 20. Burst without MEASURE / parallel `P` clones

**Failure:** drain **complete** from memory or two leads each **safe**.
**Trigger:** “P reads 3 days”; clone (`LEGACY-12.md`); in-seat “we handled it” (`LEGACY-29.md`).
**Symptom:** no A∧B; split `review_closed`; `R` split.
**Cheapest guard:** one lock, one completed look + canaries, then maybe close. Resume is cold admit. Memory does not travel.

## 21. Window is local civil days / archive omitted

**Failure:** that bag **complete**. Covers in another civil day or archive sit (`TASK-19-TIMESTAMP-HAZARD.md`).
**Trigger:** “last three local days”; hot-only glob; wrap (`PROVENANCE-ARCHIVE.md`).
**Symptom:** `B=∅`; archive has opened; `open_units` 0 (`TASK-34-METRIC-THAT-LIES.md`).
**Cheapest guard:** `W` UTC ≥72h. Include archive. Failed archive list ⇒ ERROR.

## 22. Canary skip “to finish the 3 days”

**Failure:** controls failed; remaining self-rows **trusted** (`LEGACY-39.md`).
**Trigger:** exists-only canary; timeout `[]` as NONE; miss in hits.
**Symptom:** all opened CLOSED; or empty=done (§6).
**Cheapest guard:** canaries every pop. Fail ⇒ stop burst, not NONE. Do not close the rest.

## 23. Worker writes opened **and** closed in one flush

**Failure:** one author, two types, **done**. Conjunction faked in one file (`LEGACY-32.md`).
**Trigger:** `W` dumps both; sidecar `.ok` + json; mailbox ACK+drop.
**Symptom:** A never run; B’s writer == worker `pid+start`.
**Cheapest guard:** opened wrapper ≠ closed checker (different incarnation). Same pid+start on both ⇒ refuse close.

## 24. `I` / monitor / UI green on opened count or self-closed

**Failure:** quota **spent**; `errors=0` because `S` emitted success rows.
**Trigger:** billing on `review_opened`; `M` counts ERROR only; badge “covered.”
**Symptom:** included gone; UNVERIFIED covers; `order_key` ignored.
**Cheapest guard:** decrement `I` only after checker A∧B. Alert on stale scan, not on opened==closed.

## 25. Live `p3` interleaves every review; trickle called burst **done**

**Failure:** interval **complete**. `B` barely moved (`BURST-REVIEW.md`).
**Trigger:** live-first UI; 1 review/hour; detector “in progress.”
**Symptom:** hundreds of `review_id`s remain; `P` context-switches; leftovers look done to `C`.
**Cheapest guard:** burst spends usable included on `B` except live `p0`. `list_next` = `B` then live. Trickle ≠ drained.

## 26. Resume inherits `S`/`P` “we handled it”

**Failure:** in-seat memory is the queue (`TASK-05-DOCTRINE-DRAFT.md`).
**Trigger:** cold start loads model context; peer extract (`LEGACY-19.md`); LKG the job wrote.
**Symptom:** no rows on disk; COMPLETED; successor never measures (`LEGACY-29.md`).
**Cheapest guard:** new incarnation, new admit. DISCARD peer/self memory. Only estate JSONL + A∧B.
