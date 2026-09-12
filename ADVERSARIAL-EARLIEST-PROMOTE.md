# adversarial review: priority queue using earliest-promote-wins

Mechanism: a lane is `effective_priority` **frozen at `promoted`**; inside the lane the first flushed `promoted` in `[0,hwm)` is the only id that may `released` (`ORDERED-RELEASE-QUEUE.md`). Success is typically **`may_release`**, **`released`**, **`list_next` line 1**, or **“order held.”** Confident wrong = **success, completeness, or safety** that does not hold.

## 1. Any Sig1∧Sig2 in the lane `released` (earliest-promote dead)

**Failure:** a later id **closed**. Order **preserved** in the UI (“whoever finished”).
**Trigger:** `if verify: released`; presence consumer (`PROVENANCE-ARCHIVE.md`); `C` local gate (`DEPENDENCY-INVERSION.md`).
**Symptom:** head still inflight; tail COMPLETED; `C` takes the later file.
**Cheapest guard:** `may_release` iff id is head ∧ A∧B. Later finished bytes sit. `C` waits on ledger head, not `exists`.

## 2. Failed head skipped silently; rest **released**

**Failure:** poison head **handled**. Successors **unblocked**. Head never `stale_archived`.
**Trigger:** Sig1 fail; 402; `exists` leftover; “don’t stall the lane.”
**Symptom:** head still `lane[0]`; or gone without archive (`ORPHAN-RECLAMATION.md`); bag looks drained.
**Cheapest guard:** skip only via `released.kind=stale_archived` after verified archive (`STALE-CLAIM-RECOVERY.md`). Silent skip ⇒ ERROR, not complete.

## 3. Empty / failed ledger look ⇒ all `may_release`

**Failure:** no prefix ⇒ **no heads** ⇒ everyone free, or `[]` ⇒ idle (`POSITIVE-CONTROL.md`).
**Trigger:** timeout; HWM=0; wrap (`TASK-22-ROLLING-WINDOW-AUDIT.md`); canary skip.
**Symptom:** leapfrog storm; or `list_next=[]` while `promoted` sit (`SILENT-FAILURE-DETECTION.md`).
**Cheapest guard:** look ERROR ⇒ no release. Canary-hit missing ⇒ not empty. Empty is not “all may.”

## 4. Clock / `scheduled_utc` / enqueue time as “earliest”

**Failure:** sort by `at_utc` or schedule. Skew **wins**. Order **held**.
**Trigger:** NTP step; local civil `at_utc` (`TASK-19-TIMESTAMP-HAZARD.md`); never-popped older `scheduled_utc` (`LEGACY-30.md`).
**Symptom:** later promote with earlier clock `released` first; or never-popped blocks the lane forever (called priority).
**Cheapest guard:** file order in `[0,hwm)` only. Clock is display. Never-popped is not a promote.

## 5. `ls` / mtime / filename `p{n}` is the lane and the head

**Failure:** display order is **the** queue (`LEGACY-35.md`, `LEGACY-11.md`).
**Trigger:** `p0-…` lexical; `p10` vs `p2`; touch (`LEGACY-27.md`); operator “cancel line 1.”
**Symptom:** human and worker disagree; wrong `unit` killed; aged `p3` starved or leapfrogs.
**Cheapest guard:** `list_next()[0].unit == peek()`. Filename does not compute rank or head (`LEGACY-36.md`).

## 6. Lane recomputed at release (aging leapfrog)

**Failure:** waiting `p3` ages to `p0` and **wins** over a live frozen `p1` (`LEGACY-34.md`).
**Trigger:** `effective_priority(now)` on `may_release`; rewrite body; `urgency` dual key.
**Symptom:** in-flight reorder; “fairness” COMPLETED; original head stuck.
**Cheapest guard:** freeze `L` at `promoted`. Aging is dequeue view only, never release order.

## 7. `released` / COMPLETED without A∧B on the head

**Failure:** head **popped**. Completeness of the lane (`DUAL-SIGNATURE-COMPLETION.md`).
**Trigger:** `kind=sent`; HTTP 200; `W` `DONE` (`LEGACY-05.md`); `released` before `promoted` (`ADVERSARIAL-LEDGER-QUEUE.md` §2).
**Symptom:** `offer` no-ops; bytes ≠ `expected`; successors run.
**Cheapest guard:** pop only after this process’s Sig1∧Sig2 (or verified `stale_archived`). Name/`DONE` is not head-done.

## 8. Consumer takes later bytes (`exists` / glob / mailbox)

**Failure:** safety of **order**. Ledger still has an earlier head.
**Trigger:** `C` globs `out/`; mailbox ACK (`ADVERSARIAL-MAILBOX-QUEUE.md`); burst `review_closed` (`BURST-REVIEW.md`).
**Symptom:** later file consumed; head UNVERIFIED; two stories of “next.”
**Cheapest guard:** `C` admits only ledger head. Presence of a later path is not `released`.

## 9. Two `promoted` in one flush / memory pop ≠ ledger

**Failure:** tie undefined; both **first**. Or RAM order **safe** after reboot disagrees.
**Trigger:** batch append; no HWM between; in-memory queue as truth (`IDEMPOTENT-EVENT-LEDGER.md`).
**Symptom:** split heads; double-pop; reboot “heals” by omitting a promote.
**Cheapest guard:** one event per append; HWM between. Replay `[0,hwm)` is the only order. Memory is a checksummed cache.

## 10. Override: chat / whole lane / no `would_deny` / no MAC

**Failure:** leapfrog **granted**. Safety of exemption (`LEGACY-22.md`).
**Trigger:** `P` ACK; bool on the WO; one row releases all; `G` timeout ⇒ local (`EVIDENCE-TIERING.md`).
**Symptom:** `predecessor_id` missing; `R` unlabelled; A∧B skipped.
**Cheapest guard:** one flushed row, `would_deny` recomputed (`may_release` was false), `predecessor_id`, still Sig1∧Sig2. Chat is not an override.

## 11. Cover / retry **prepended**; routing rewrites promote sequence

**Failure:** new `enqueue_id` is **head**. Old promote **handled** (`SUBSTITUTE-REVIEW-QUEUE.md`).
**Trigger:** substitute cover; UUID per retry (`TASK-27-RETRY-SEMANTICS.md`); cheapest-first hop (`BUDGET-AWARE-ROUTING.md`).
**Symptom:** later cover `released` first; original never launched or leapfrogged.
**Cheapest guard:** append on promote only. Same `enqueue_id` on retry. Routing must not reorder `[0,hwm)`.

## 12. HWM / torn `promoted` / two writers ⇒ wrong head

**Failure:** prefix **complete** through a hole or a split ledger (`ADVERSARIAL-LEDGER-QUEUE.md` §4, §8).
**Trigger:** hwm++ before fsync; skip-bad-JSON; clone + primary.
**Symptom:** each side’s head “valid”; one `released` the other’s inflight.
**Cheapest guard:** append→fsync→HWM. Torn line ⇒ ERROR. One lock on the ledger.

## 13. `stale_archived` without a verified copy, or without `released`

**Failure:** head **gone** from the lane, or still head while disk says archived.
**Trigger:** delete-hot; symlink archive (`ADVERSARIAL-ARCHIVE-MANIFEST.md`); archive without ledger pop.
**Symptom:** successors blocked forever **or** unblocked with bytes lost.
**Cheapest guard:** archive then `released.kind=stale_archived` only after this process hashed the archive payload.

## 14. `list_next` ≠ `may_release` heads (`order_key` vs “whichever finished”)

**Failure:** line 1 is **next**. Action pops another `unit` (`LEGACY-35.md`).
**Trigger:** UI finished-first; live queue before burst remainder (`BURST-REVIEW.md`); `ls`.
**Symptom:** operator cancels/archives the wrong id; two leads (`LEGACY-01.md`).
**Cheapest guard:** `list_next` = heads of lanes in `order_key`, then live queue. Same predicate as `may_release`/`peek`.

## 15. Promote without a live claim (or pid-only “holder”)

**Failure:** ledger **in-flight/safe**. OS has no holder or a recycled pid (`IDEMPOTENT-EVENT-LEDGER.md`).
**Trigger:** promote-on-enqueue; `start` missing; pid-only (`ADVERSARIAL-PID-ONLY-LIVENESS.md`).
**Symptom:** head never finishes; lane frozen “working”; or steal under a stranger.
**Cheapest guard:** `promoted` only after lock publish `{pid,start}`. Recycle ⇒ not inflight.

## 16. `verify` classified as MUTATE; head cannot Sig1 ⇒ skip as done

**Failure:** deadlock looks like a **failed head handled** (`GATE-DEADLOCK.md`).
**Trigger:** hook blocks hashing; ERROR→skip (§2); MEASURE refuse ⇒ NONE.
**Symptom:** lane “cleared”; no A; rest released.
**Cheapest guard:** MEASURE free. Hook ERROR ⇒ not `stale_archived`. Do not skip a head you could not hash.

## 17. Civil / TZ / day-folder reorder of `promoted`

**Failure:** partition sort is **file order**. Today’s promote **first**.
**Trigger:** `ledger/YYYYMMDD/`; local `at_utc`; DST (`TASK-07-DATE-ROLLOVER.md`).
**Symptom:** yesterday’s head invisible; today’s id `released`; unmatched prefix.
**Cheapest guard:** one UTC JSONL. `at_utc` is `…Z` only and not the sort key.

## 18. `urgency` / missing `priority` / silent reset = lane **known**

**Failure:** a second key or default `3` **is** `L` (`LEGACY-36.md`, `LEGACY-27.md`).
**Trigger:** add-only reader ignores `priority`; packaged default; SDK “high.”
**Symptom:** all work one lane (FIFO lie) or p0 flood; `order_key` disagrees.
**Cheapest guard:** missing/unknown `priority` ⇒ raise. One `parse_priority`. No `urgency` without `priority`.

## 19. `I` / monitor decremented on `promoted` or on leapfrog `released`

**Failure:** quota **spent**; `errors=0` because a later id closed (`BUDGET-AWARE-ROUTING.md`).
**Trigger:** billing on promote; `M` ignores leapfrog; `open_units` wrap (`TASK-34-METRIC-THAT-LIES.md`).
**Symptom:** included gone; head UNVERIFIED; greens.
**Cheapest guard:** decrement `I` only after this process’s A∧B on a `may_release` head.

## 20. Reclaim / new id inserts at head (or leapfrogs a real earlier promote)

**Failure:** recovered `qid` **first**. Safety of never-launched (`ORPHAN-RECLAMATION.md`).
**Trigger:** reclaim always prepend; mint new `enqueue_id`; timeout ⇒ orphan.
**Symptom:** live earlier promote skipped; two shells; DESIGN brake bypassed.
**Cheapest guard:** reclaim re-inserts at its frozen place only if it was head when falsely released; else append. UNKNOWN `same` ⇒ no reclaim.

## 21. Dual predicate: dequeue uses aging `order_key`, release uses another sort

**Failure:** both **correct** locally. Next-to-run ≠ next-to-release (`LEGACY-10.md`).
**Trigger:** five copies of priority (`LEGACY-36.md`); router cheapest-first; archive FIFO.
**Symptom:** `list_next` vs `may_release` diverge; starvation called policy.
**Cheapest guard:** one `order_key` module. Queue, list, release, router import it. Tombstone `sort_by_name`.

## 22. `queue_full` / lock skip as **done** or **not in lane**

**Failure:** never-promoted id **absent**. Completeness of the bag (`TASK-25-QUEUE-BACKPRESSURE.md`).
**Trigger:** N=32; merely-old lock skipped as drop (`LEGACY-14.md`); admit deny.
**Symptom:** `list_next` omits; `offer` duplicate; work OPEN.
**Cheapest guard:** skip ≠ `released`. `queue_full` is UNKNOWN, same `enqueue_id`. Old+live ⇒ wait, not pop.

## 23. Canary-miss FOUND / hit missing and `may_release` continues

**Failure:** controls failed; remaining heads **trusted** (`LEGACY-39.md`).
**Trigger:** skip controls “to drain”; exists-only canary.
**Symptom:** empty look as all-may (§3); or prefix matcher names the head (`RECEIPT-MATCHING.md`).
**Cheapest guard:** miss in hits or hit missing ⇒ no `may_release` this look.

## 24. Owner renames `p0-…` / edits `priority` to move the head

**Failure:** display mutate is **the** freeze (`LEGACY-17.md`).
**Trigger:** `mv`; body rewrite; gold-set `urgency`.
**Symptom:** leapfrog without exemption row; digest/order both moved (`LEGACY-21.md`).
**Cheapest guard:** cannot rename/edit into a new lane. Override is the exemption row only.

## 25. `W` / model writes `promoted` or `released` for a later id

**Failure:** worker bit is **head popped** (`TASK-10-SELF-REPORT.md`).
**Trigger:** SDK “done”; `status=completed`; burst model `review_closed`.
**Symptom:** COMPLETED out of order; checker Sig2 never ran.
**Cheapest guard:** only checker/wrapper append `released`. `W` cannot `promoted`/`released`. Guess never greens `list_next`.

## 26. Index / cancel-by-line from a stale `list_next`

**Failure:** operand is **line 1** of an old snapshot. Action **succeeds** on the wrong `unit`.
**Trigger:** reuse index after a pop; parse `ls`; no `pid+start` on the listing (`LEGACY-35.md`).
**Symptom:** cancel/archive/kill the new head; old head still running.
**Cheapest guard:** actions take `unit`. Index only from **this** listing incarnation, discarded after one use.
