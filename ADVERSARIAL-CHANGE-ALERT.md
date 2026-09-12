# adversarial review: monitor that alerts only on change

Mechanism: `M` pages on a **delta** (new fingerprint, `errors` increase, hash ≠ last). Same state ⇒ **silence**. Silence is treated as healthy / caught up / safe (`TASK-08-WATCHDOG-PATTERN.md`). Confident wrong = **success, completeness, or safety** that does not hold.

## 1. Same fingerprint, still broken — silence = **healthy**

**Failure:** one speak per episode. Repeats are quiet. Hour contains the silence (`SILENT-FAILURE-DETECTION.md` §3).
**Trigger:** `last_alert_fp` match; “no new errors”; `watchdog_alerts=0` (`TASK-34-METRIC-THAT-LIES.md`).
**Symptom:** estate on fire; dashboard no-change; operators: monitor working.
**Cheapest guard:** healthy iff completed MEASURE + both controls + fresh `last_successful_scan`, not `fp==last`. Alert if `silent` (`errors==0 ∧ ¬healthy`).

## 2. Probe UNKNOWN takes the silent path

**Failure:** no reading is **no change** / health (`TASK-21-WATCHDOG-SILENCE.md`).
**Trigger:** timeout; raise; `except: pass`; empty stdout⇒NONE (`POSITIVE-CONTROL.md`).
**Symptom:** LKG untouched or still displayed as now; cron 0 (`LEGACY-04.md`).
**Cheapest guard:** UNKNOWN must speak if new, **never** overwrite LKG, never count as healthy. `M` exit ≠ 0 on its own timeout.

## 3. Wrap / rotate drops ERROR — delta back to zero = **recovered**

**Failure:** hot empty is a **change** to healthy (`TASK-22-ROLLING-WINDOW-AUDIT.md`, `TASK-30-LOG-ROTATION.md`).
**Trigger:** cap evicts the episode; `M` reads current `*.log` only; archive has the ERROR.
**Symptom:** “cleared”; `open_units` 0; incident unreplayable (`BLAST-RADIUS-ESTIMATE.md`).
**Cheapest guard:** count hot+archive, same UTC window. Eviction is not a close. Missing canary-hit ⇒ ERROR, not recovery.

## 4. No look / `[]` — fingerprint of empty is stable **OK**

**Failure:** instrument never ran. Empty hash **unchanged** (`LEGACY-09.md` §2, `LEGACY-03.md`).
**Trigger:** job not scheduled; fail-closed peek (`ADVERSARIAL-FAIL-CLOSED.md` §1); `find_due` timeout.
**Symptom:** ticks silent; due work on disk; never-tried (`LEGACY-30.md`).
**Cheapest guard:** empty is NONE only after completed look + canary-hit/miss. No look ⇒ UNKNOWN, must not be “no change.”

## 5. ERROR mapped to success — error counter **does not change**

**Failure:** defect emits COMPLETED/`exists`/prefix/`released` without delivery (`SILENT-FAILURE-DETECTION.md` §2).
**Trigger:** `status=completed`; `test -f`; `re` names it (`ADVERSARIAL-RECEIPT-NAMES.md`).
**Symptom:** `M` correct: zero ERROR rows; units CLOSED wrong.
**Cheapest guard:** `M` cannot see those. Health is `G.measure` canary-hit/miss, not `errors`. Dual-run until success predicates are tombstoned.

## 6. LKG / last_hash written by the job — detector **same**, estate drifted

**Failure:** hash includes `W` status or LKG the probe wrote (`LEGACY-09.md` §3).
**Trigger:** `content_hash(state)==last_hash`; failed probe mapped healthy; start-touch `last_ok` (`ADVERSARIAL-CRON-10M.md` §2).
**Symptom:** skip forever; LKG never moves; “no change” green.
**Cheapest guard:** hash only inputs the job does **not** write (`re`, declared digest, `pid+start`). LKG only after a successful **healthy** probe.

## 7. Fingerprint too coarse — new units, same `kind`, **already alerted**

**Failure:** episode id is `kind` or “errors>0”. Completeness of the page.
**Trigger:** `fingerprint("anomaly", "")`; omit `unit`/`enqueue_id`; 402 and timeout share a token.
**Symptom:** first WO paged; 621 silent; `R` asserted as 1 (`BLAST-RADIUS-ESTIMATE.md`).
**Cheapest guard:** fp includes `kind` + identity (`unit`, `qid`, probe class). New id ⇒ new episode. Decisive check on a non-canary green row.

## 8. Filter / civil partition — counted bag **unchanged**

**Failure:** UNKNOWN, 402, `denied_here` excluded; or `YYYYMMDD` local vs UTC hour (`SILENT-FAILURE-DETECTION.md` §5–6).
**Trigger:** `level=error`; HTTP 5xx only; “last hour” local (`TASK-19-TIMESTAMP-HAZARD.md`).
**Symptom:** change-only monitor sees 0 delta; work stalling.
**Cheapest guard:** window `[now_utc-interval, now_utc]`. UNKNOWN/402/`unhealthy` are first-class. Wrong partition + missing canary ⇒ ERROR.

## 9. Dead writer, quiet lock — no new events = **no change**

**Failure:** nothing runs ⇒ nothing diffs (`LIVENESS-PREDICATE.md`, `LEGACY-37.md`).
**Trigger:** stale `opened` (`ADVERSARIAL-OPENED-NEVER-CLOSED.md`); pid recycle “live”; holder gone.
**Symptom:** “no new errors”; units blocked; skip inflight (`ADVERSARIAL-ONE-PER-TICK.md` §13).
**Cheapest guard:** require `same_process` or proven absent. Dead/UNKNOWN ⇒ not healthy silence. Age of oldest `opened` without `same` pages.

## 10. `M`’s look fails, exit 0 — monitor **OK**

**Failure:** the instrument’s timeout is **no delta** (`LEGACY-03.md`).
**Trigger:** `except: return 0`; `errors==0` on swallow; self-as-sample only (`LEGACY-13.md`).
**Symptom:** “no errors in the last hour” true of a failed COUNT.
**Cheapest guard:** `M` ERROR ⇒ ≠ 0 and `probe_kind=ERROR`. Do not insert a heartbeat so COUNT looks live (`LEGACY-17.md`).

## 11. Change-detector skips the job that would update the hash

**Failure:** no change ⇒ skip ⇒ hash never moves ⇒ **safe** (`LEGACY-09.md`).
**Trigger:** `exists`; `[]`; `last_hash`; auto-start one-per-tick skip.
**Symptom:** leftover stub forever; cron 0; digest ≠ `expected`.
**Cheapest guard:** suppress only on A∧B. UNKNOWN/`[]`/exists must **run** or escalate.

## 12. `M` watches `C`; `G` would ERROR — local **unchanged success**

**Failure:** consumer drift is not a delta on `C` (`DEPENDENCY-INVERSION.md`).
**Trigger:** prefer local; `exists` gate; `G` timeout unused.
**Symptom:** `C` quiet green; `G` UNKNOWN; estate wrong.
**Cheapest guard:** `M` is `G.measure`. `G` ERROR ⇒ `C` ERROR. No local fallback.

## 13. Wrap “recovery” + one OK line **clears** the episode

**Failure:** return to empty/`OK` is **healthy change** (`ADVERSARIAL-SELF-SUMMARY.md`).
**Trigger:** `W` prints OK; `exists` leftover; rotate then `last_ok`.
**Symptom:** fp updates to healthy; still UNVERIFIED (`LEGACY-31.md`).
**Cheapest guard:** clear episode only after a completed healthy probe + both controls. OK/`exists` is `model_guess`.

## 14. First speak never durable; later ticks **already alerted**

**Failure:** fp stored, line not flushed. Completeness of the page (`TASK-31-GRACEFUL-SHUTDOWN.md`).
**Trigger:** print then crash; HWM ahead; MAILTO/`devnull` (`ADVERSARIAL-CRON-10M.md` §19).
**Symptom:** no page ever; silence thereafter; state says spoken.
**Cheapest guard:** fsync state **after** the speak is durable, or speak every UNKNOWN/unhealthy until ack. Missing first line ⇒ not “no change.”

## 15. Hash-then-normalise / pretty JSON — fp **same**, bytes drifted

**Failure:** detector **unchanged**. Citations orphan (`TASK-15-IDEMPOTENT-RECEIPT.md`, `LEGACY-21.md`).
**Trigger:** hash before canonicalise; append a note; key order.
**Symptom:** skip; payload moved; `verify` UNVERIFIED.
**Cheapest guard:** hash canonical declared fields only. Digest mismatch ⇒ unhealthy, not “same.”

## 16. Episode id includes `at_utc` / tick — or omits it wrongly

**Failure:** every tick is new (noise) **or** (if they then debounce by `kind`) back to §7. Operators mute `M`; silence = **safe**.
**Trigger:** `event_id` with clock (`IDEMPOTENT-EVENT-LEDGER.md`); then “alert only if kind changes.”
**Symptom:** pages ignored; real new `unit` dropped.
**Cheapest guard:** `event_id` minus `at_utc`, plus identity. Mute is not a MEASURE. Do not debounce away `unit`.

## 17. Monitor disabled / last fp leftover — **no change** for days

**Failure:** safety of the last episode state (`ADVERSARIAL-CRON-10M.md` §18).
**Trigger:** crontab commented; `M=0`; host down; state file restored from backup.
**Symptom:** dashboard last-alert old/green; “no new alerts.”
**Cheapest guard:** `last_successful_scan` age > interval ⇒ broken. Restored fp is not this run.

## 18. Canary missing; fp still “healthy empty”

**Failure:** controls failed; empty bag **unchanged** (`POSITIVE-CONTROL.md`).
**Trigger:** skip canaries “no news”; exists-only canary; `M` inserts canary so look succeeds.
**Symptom:** wrap/wrong day looks like last hour; `errors==0`.
**Cheapest guard:** hit missing or miss FOUND ⇒ `probe_kind=ERROR`, must speak. Do not mint the control.

## 19. `opened` never closed — ledger **unchanged** = in-progress **safe**

**Failure:** no close event is **no delta** (`ADVERSARIAL-OPENED-NEVER-CLOSED.md`).
**Trigger:** claims stay `opened`; `M` diffs `review_closed` count; silence as live (`LEGACY-37.md`).
**Symptom:** holders dead; `open_units` flat; no page.
**Cheapest guard:** page if oldest `opened` lacks `same` and lacks A∧B. Status spelling is not a delta key.

## 20. Dual predicate: change-monitor greens; hasher / `re` matcher red

**Failure:** the quiet `M` is on the run path (`LEGACY-15.md`).
**Trigger:** README says A∧B; cron still “alert on change”; `pip` old watchdog.
**Symptom:** split-brain; job skipped (`LEGACY-09.md`).
**Cheapest guard:** tombstone change-equals-health. One published health predicate: completed probe. Dual-run until the silent-zero branch is dead.

## 21. `I` / `launched==1` / `errors` gauges flat — **capacity fine**

**Failure:** attempt/quota metrics **unchanged** (`BUDGET-AWARE-ROUTING.md`, `ADVERSARIAL-ONE-PER-TICK.md` §26).
**Trigger:** decrement on launch; one-per-tick always 1; `M` pages only if gauge moves.
**Symptom:** included gone; bag unmoved; no page.
**Cheapest guard:** alert on stale scan and `due` age, not on gauge delta. Progress is A∧B count.

## 22. Self-as-sample / observer row is the only change tracked

**Failure:** `M` measures `M` (`LEGACY-13.md`).
**Trigger:** heartbeat `opened`; COUNT of “our” type; canary `M` wrote.
**Symptom:** delta always healthy; estate rows excluded.
**Cheapest guard:** subtract `{self.pid,start}` and `M`’s types before decide. Remainder empty is a completed look of **others**, not “I am healthy.”

## 23. Parallel / last-writer fingerprint covers later FAIL

**Failure:** first shard’s fp is **the** episode (`ADVERSARIAL-SELF-SUMMARY.md` §17).
**Trigger:** interleaved probes; two `M`s; last write wins.
**Symptom:** FAIL then OK fp; bag “no change” after a fail.
**Cheapest guard:** one typed object after **all** shards. Any shard ERROR ⇒ not silent-healthy.

## 24. Peer / `P` ACK / chat: “we already alerted” — **no change**

**Failure:** a sentence is the episode (`EVIDENCE-TIERING.md`, `LEGACY-19.md`).
**Trigger:** inherit last_fp on resume (`LEGACY-29.md`); mailbox ACK; burst chat.
**Symptom:** successor silent; no MEASURE.
**Cheapest guard:** DISCARD peer fps. Cold admit. ACK is `model_guess`.

## 25. `list_next` / UI “no new alerts” as **caught up**

**Failure:** the badge is **status** (`LEGACY-35.md`).
**Trigger:** hide `silent`; sort by last_alert mtime; omit `tier` (`EVIDENCE-TIERING.md`).
**Symptom:** operator acts on idle; UNKNOWN bag invisible.
**Cheapest guard:** show `probe_kind`, `last_successful_scan`, `silent`. Guess never greens. Same predicate as the probe.

## 26. Mute after noise / “only alert on change” after they disabled speak-on-UNKNOWN

**Failure:** policy **held**. UNKNOWN and repeats both quiet.
**Trigger:** paging fatigue; `ALERT_ON_CHANGE=1` plus `except: pass` (§2).
**Symptom:** first outage paged; continuation and probe-death dark.
**Cheapest guard:** UNKNOWN/unhealthy speak until a completed healthy probe. Change-only applies to **healthy** repeats, not to failed looks.
