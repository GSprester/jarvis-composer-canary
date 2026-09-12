# adversarial review: JSONL ledger as work queue

Mechanism: append-only JSONL (`enqueued` / `promoted` / `released` / later `reclaim`) replayed into `{queued, inflight, done}` (`IDEMPOTENT-EVENT-LEDGER.md`). Confident wrong = dispatcher or consumer reports **success, completeness, or safety** that does not hold.

## 1. `released` with a cited digest that was never verified

**Failure:** `done` is a function of a row, not of A∧B. `evidence_sha256` copied from the worker, from `exists`, from HTTP 200, or from `kind=sent`.
**Trigger:** release-on-promote, release-on-ack, drain `status=completed`, `C`’s local gate (`DEPENDENCY-INVERSION.md`).
**Symptom:** `qid` in `done`; `offer` no-ops; `list_next` omits the unit; work never ran or bytes ≠ `expected`.
**Cheapest guard:** refuse `released` unless this process just hashed the declared path and `d==expected` (Sig1) and wrote Sig2 (`DUAL-SIGNATURE-COMPLETION.md`). One row with `delivery.kind` not `verify_completed|stale_archived` or `evidence_sha256` not recomputed ⇒ `R` = all `released` from that writer (`BLAST-RADIUS-ESTIMATE.md`).

## 2. `released` before `promoted` (or without a lock)

**Failure:** apply treats “not inflight” as already-done no-op, or maps apply-UNKNOWN to skip. Replay marks complete. Never-tried (`ORPHAN-RECLAMATION.md`).
**Trigger:** pop/ack then crash before lock rename; release-on-dequeue; two writers, `released` flushed, `promoted` not.
**Symptom:** queue “complete”; re-enqueue `qid` is `duplicate`; no `pid+start`; no `route_intent`.
**Cheapest guard:** `released` apply if `qid` not in `inflight` → ERROR (never no-op unless prior `released` has a **recomputed** delivery). `offer` returns `ORPHAN` not `duplicate`.

## 3. Failed or torn read reported as idle / complete

**Failure:** `open` error, timeout, or skip-bad-JSON continues; empty `{queued,inflight}` ⇒ “nothing due” / “caught up.”
**Trigger:** disk full mid-append; HWM = file length; rotate/wrap of the JSONL (`TASK-22-ROLLING-WINDOW-AUDIT.md`); parser skip of a torn line that was `enqueued` or `promoted`.
**Symptom:** `errors=0`, `list_next=[]`, cron exit 0; WOs still OPEN (`SILENT-FAILURE-DETECTION.md`).
**Cheapest guard:** any I/O, JSON, or missing-canary look is ERROR, not empty (`POSITIVE-CONTROL.md`). Canary-hit must appear in replay or the queue is not idle.

## 4. HWM ahead of durable bytes (or visibility = length)

**Failure:** prefix is “complete” through a torn `released` or through a hole. Replay either applies a partial object or stops after a false `done`.
**Trigger:** hwm++ then crash before fsync; reader uses `st_size`; two processes advance HWM.
**Symptom:** `done` contains a `qid` whose payload was never fsynced; or the next boot replays a shorter prefix and “heals” by omitting work (looks like it finished last time).
**Cheapest guard:** append → fsync → then HWM; readers `[0,hwm)` only. One writer lock on the ledger file (`LEGACY-01.md`).

## 5. Prefix wrap / truncate: suffix replay looks finished

**Failure:** hot JSONL evicted or `>` truncated; remaining lines are only late `released` or nothing.
**Trigger:** log rotate without archive-before-evict; “compact the queue.”
**Symptom:** replay `done` or empty; operators see a clean queue; archive has the `enqueued` that never promoted.
**Cheapest guard:** refuse rotate until month archive verifies digest+count (`PROVENANCE-ARCHIVE.md`). Replay without canary-hit ⇒ ERROR.

## 6. `event_id` / `seen` collapses distinct facts

**Failure:** `event_id` is `qid` only, or includes `at_utc`, or UUID per write. Distinct `promoted`/`released` skipped or doubled.
**Trigger:** retry after timeout; two seats append; clock in the hash.
**Symptom:** second `enqueued` (corrected `expected_sha256`) ignored → later `released` cites old digest, `done` is confident and wrong. Or double `promoted` both “safe.”
**Cheapest guard:** `event_id = sha256(canonical minus at_utc)` including `type` and `expected_sha256`. `apply` keys state by `qid` but **does not** skip a new `type` for that `qid`.

## 7. `stale_archived` without a verified archive copy

**Failure:** lane head popped; `done` means “safely discarded.” Bytes gone or never copied.
**Trigger:** crash-loop accident treated as design (`CRASH-LOOP-BRAKE.md`); delete-hot; symlink archive.
**Symptom:** ordered-release unblocks; earlier work “complete”; consumer never sees payload.
**Cheapest guard:** `kind=stale_archived` only if `sha256(archive payload)==evidence_sha256` just hashed (`STALE-CLAIM-RECOVERY.md`).

## 8. Two ledger writers

**Failure:** two HWM, interleaved JSON, each replay “valid” and different. Each side reports its queue safe.
**Trigger:** substitute + primary; “just run the shell”; clone (`LEGACY-12.md`).
**Symptom:** split `done` sets; one side `offer` no-ops, the other launches; or both `released` the same `qid` with different `evidence_sha256`.
**Cheapest guard:** one published lock on the ledger path; `same_process` or refuse append (`LIVENESS-PREDICATE.md`).

## 9. In-memory queue is the truth; ledger is a log

**Failure:** memory says drained/safe; ledger prefix disagrees after crash.
**Trigger:** optimize replay; flush ledger async; UI/`list_next` from RAM.
**Symptom:** reboot: work vanished (memory empty ⇒ “complete”) or double-pop (memory lost inflight).
**Cheapest guard:** `list_next` and `offer` from replay only. Memory is a cache of `[0,hwm)` with checksum of prefix.

## 10. Orphan `qid` dedup as successful absorb

**Failure:** `offer` returns OK/duplicate for never-launched `qid`. Completeness of the submit path.
**Trigger:** `released` without `promoted`; caller treats duplicate as “already handled.”
**Symptom:** producer stops retrying; unit never runs (`ORPHAN-RECLAMATION.md`).
**Cheapest guard:** `ORPHAN` ≠ `duplicate`. Duplicate only if last `released.delivery` was recomputed Sig1∧Sig2.

## 11. `reclaim` without proving never-launched

**Failure:** `qid` re-queued while a worker is live or Sig1 already holds. “Safe reclaim.”
**Trigger:** timeout of lock probe ⇒ orphan; `P` ACK (`BURST-REVIEW.md`).
**Symptom:** two shells, two artifacts; or reclaim of DESIGN `crash_loop`.
**Cheapest guard:** UNKNOWN `same` ⇒ no reclaim. Sig1 true ⇒ not orphan. DESIGN row ⇒ no reclaim.

## 12. Leapfrog `released` / override without MAC

**Failure:** later `qid` in `done` while earlier never launched. Safety of “order preserved.”
**Trigger:** consumer `exists`; owner chat; `release_override` without `would_deny` (`ORDERED-RELEASE-QUEUE.md`).
**Symptom:** head still inflight; tail “complete”; `C` processes the later file.
**Cheapest guard:** `may_release` is head ∧ Sig1∧Sig2. Override is one exemption row; `C` waits on ledger head, not path presence.

## 13. `expected_sha256` omitted or worker-supplied

**Failure:** `enqueued` accepted; `released` uses `exists` or `W`’s hash. `done` is confident.
**Trigger:** schema “optional digest”; add-only reader ignores missing (`TASK-33-SCHEMA-EVOLUTION.md` misread as optional).
**Symptom:** empty stub COMPLETED; 494 WOs share a template digest (`RECEIPT-MATCHING.md`).
**Cheapest guard:** missing/empty-hash `expected` ⇒ refuse `enqueued`. Release hashes the file, does not read `W`.

## 14. `unit` / `qid` from filename or local clock

**Failure:** replay joins the wrong WO; “all of today’s work released.”
**Trigger:** `name[:8]`; `at_utc` local; `scheduled_utc` civil (`TASK-19-TIMESTAMP-HAZARD.md`).
**Symptom:** `done` count matches 74 quoted names; 494 unquoted “complete” via prefix; or tomorrow’s bag empty.
**Cheapest guard:** `unit` is `re`; `at_utc` is `…Z` only. Canary-miss in FOUND ⇒ matcher is the defect.

## 15. Unknown `type` ignored (forward-compat) as success

**Failure:** `reclaim` / `crash_loop` / `completion_sig2` not in `apply`; state stays `done`. Operators think the new event “took.”
**Trigger:** mixed-version readers; extra keys ignored ⇒ whole event ignored.
**Symptom:** reclaim flushed, `list_next` still omits `qid`; Sig2 row present, `released` still unwritten, UI shows complete from `W`.
**Cheapest guard:** unknown `type` ⇒ replay ERROR (not skip). Ship `apply` arms before writers.

## 16. Quota / monitor read `done` as safety

**Failure:** `I` decremented on `released`; monitor “no errors in the last hour” because the ledger emitted success (`BUDGET-AWARE-ROUTING.md`, `SILENT-FAILURE-DETECTION.md`).
**Trigger:** billing hook on `type=released`; `M` counts ERROR rows only.
**Symptom:** included gone, work UNVERIFIED; dashboard green.
**Cheapest guard:** decrement `I` only after this process’s Sig1∧Sig2. Alert on stale `last_successful_scan`, not on `errors==0`.

## 17. Pretty-print / key-order / no trailing newline

**Failure:** two “same” events, two `event_id`s; or last line without `\n` invisible until next write (looks not enqueued, then suddenly `done`).
**Trigger:** `json.dumps` default; editor save; Windows CRLF.
**Symptom:** flapping completeness; double `promoted`.
**Cheapest guard:** canonical dumps in the writer only; refuse parse if last record in `[0,hwm)` has no newline (truncated = ERROR).
