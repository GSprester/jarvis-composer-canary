# burst review

`P` returns after ≥72h UTC offline. Substitutes left `review_opened` items (`SUBSTITUTE-REVIEW-QUEUE.md`). Review is a **bounded burst**: one MEASURE/A∧B pass over that bag, then live work. **Fail if `P` ACKs the mailbox:** asserted close (`EVIDENCE-TIERING.md`). **Fail if `P` steals a still-alive `S` claim** (`LIVENESS-PREDICATE.md`).

Window `W = [t_off, t_ret]` UTC (`t_ret - t_off ≥ 3×86400`). **Fail if `W` is “last three local days”** (`TASK-19-TIMESTAMP-HAZARD.md`). Include archive (`PROVENANCE-ARCHIVE.md`). Failed list of `review/` is ERROR, not “nothing to review” (`POSITIVE-CONTROL.md`, `SILENT-FAILURE-DETECTION.md`).

## Queueing policy

Admit `P` cold (`TASK-05-DOCTRINE-DRAFT.md`). `P` holds the one-writer lock for **review MUTATE only** after MEASURE (`GATE-DEADLOCK.md`).

Bag `B` = `{review_id : opened in W ∧ not closed}`. Dedup by `review_id`. **Fail if you collapse by `unit`:** two `enqueue_id`s, one dropped (`TASK-26-DEDUPE-BY-CONTENT.md`).

Pop with `order_key` on the **WO body** (`LEGACY-36.md`), not who covered, not `opened_at` alone. **Fail if `ls` or newest-first** (`LEGACY-35.md`). Aged `p0` first; `reserve` on `P`’s quota still applies (`BUDGET-AWARE-ROUTING.md`) — **fail if burst spends overage to “catch up.”**

Per item (MEASURE, then maybe close):

```
if canaries fail: ERROR; stop burst          # not NONE
if same(S): skip item (still live)
elif A∧B: checker review_closed (deterministic)
else: UNVERIFIED; archive stub; do not mint new unit
```

**Fail if `P`’s model writes `review_closed`.** **Fail if inferred R2 closes** (`EVIDENCE-TIERING.md`). **Fail if presence of `out/<unit>` closes.**

Capacity: burst takes **all** `P` usable included until `B=∅` or cutoff, except `effective_priority==0` **live** (not review) WOs — those interleave. **Fail if live `p3` interleaves every item:** that is trickle. Substitutes **stop opening** new covers for `P`’s units once `P` is `same` live — **fail if they keep covering:** `B` grows during burst.

`list_next` during burst = remaining `B` then live queue. **Fail if UI shows live first.**

## Why burst beats trickle

Trickle = 1 review / live job / hour. Three days of cover (hundreds of `review_id`s) never drains; new `S` opens keep arriving; `P` context-switches; change-detector sees “review in progress” and skips (`LEGACY-09.md`). Each pause lets `C` treat leftover presence as done (`LEGACY-05.md`). Dual-run vs `G` is not completed for the bag (`DEPENDENCY-INVERSION.md`). Blast radius stays unbounded (`BLAST-RADIUS-ESTIMATE.md`).

Burst = one completed look over `B` with controls, then `released` only after A∧B. **Fail if burst is “`P` reads chat for 3 days”:** no MEASURE. **Fail if burst is parallel `P` clones:** two leads (`LEGACY-01.md`). **Fail if burst deletes `review_opened` to go faster:** `R` uncomputable tomorrow.

## Staleness cutoff

Let `age = t_ret - attempt_start` (OS start on the item, not mtime).

```
stale_review  iff  age > 72h
                   OR opened_at < t_ret - 72h
                   OR S incarnation start < P.boot_at   # pre-reboot cover
```

Stale items: MEASURE once; if A∧B, close; else **do not** spend burst after `T_burst = 4h` wall from `t_ret`. Remainder stays `review_opened` with `reason=cutoff`, UNVERIFIED, same `unit`. **Fail if cutoff ⇒ auto-COMPLETED.** **Fail if cutoff ⇒ delete.** **Fail if cutoff uses mtime of the review JSON:** `touch` looks fresh (`LEGACY-14.md`). **Fail if 72h is local civil.** **Fail if no cutoff:** burst never returns to live `p0`. **Fail if cutoff is 1 item:** first crash never retries (`STALE-CLAIM-RECOVERY.md` loop).

After `T_burst`, `P` is live routing again. Leftover `B` is a **new** burst next idle, same ids. **Fail if a second burst mints new `review_id`s.**

Canary-hit/miss every pop (`LEGACY-39.md`). **Fail if you skip controls “to finish the 3 days.”**

**Rule:** One UTC bag, `order_key`, A∧B close only. Burst the bag in hours, not trickle it through live work. Older than 72h UTC or past `T_burst` stays UNVERIFIED, not green.
