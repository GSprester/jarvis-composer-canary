# adversarial review: done when a receipt names the task

Mechanism: a join treats **a receipt that names a WO** (`re`, filename, stem, topic, `enqueue_id`) as COMPLETED / matched / skip. Estate: 622 WOs, 568 receipts, 74 exact `re`, 494 unquoted, 54 with no file (`RECEIPT-MATCHING.md`). Confident wrong = **success, completeness, or safety** that does not hold.

## 1. Name without payload (B without A)

**Failure:** `receipt.re == unit` (or any name) is **done**. Bytes ≠ `expected` or missing (`LEGACY-32.md`).
**Trigger:** stub + quoted `re`; `DONE:` sidecar; `exists(receipt)` (`LEGACY-05.md`).
**Symptom:** 74 CLOSED; digest empty/wrong; `verify` UNVERIFIED (`TASK-06-ARTIFACT-CHECK.md`).
**Cheapest guard:** done iff A∧B: this process hashed declared path **and** checker receipt cites that digest + `re`. Name is not A.

## 2. Filename / `name[:8]` / UTC day **is** the name

**Failure:** prefix **matched**. `re` exact was never asked (`TASK-07-DATE-ROLLOVER.md`).
**Trigger:** naive `YYYYMMDD` first-wins (`TASK-04-MATCHER-TESTS.md`); `20260910-alphabet`; next-day `…-r0`.
**Symptom:** 494 collapse onto 74 stems; canary-miss FOUND; two WOs one day, wrong pair.
**Cheapest guard:** join is `receipt.re == wo.unit` only for R1. Filename is display. Prefix/fuzzy banned (`LEGACY-10.md`).

## 3. Worker writes the receipt that names it

**Failure:** author of the work is author of **named** (`TASK-17-SEPARATION-OF-DUTIES.md`).
**Trigger:** `W` `status=completed` row; model sets `re`; same `pid+start` as the payload writer (`TASK-10-SELF-REPORT.md`).
**Symptom:** B looks true; checker never ran; next ticks skip (`LEGACY-09.md`).
**Cheapest guard:** B’s writer ≠ worker incarnation. Worker API has no path to the receipt log (`LEGACY-16.md`).

## 4. Gate copies `unit` onto the receipt after write

**Failure:** the join is minted (`LEGACY-17.md`). Name **now** matches.
**Trigger:** “helpful” matcher backfills `re`; declare fields from the receipt; `expected` from file.
**Symptom:** every leftover names its WO; `R` = that writer’s matches (`BLAST-RADIUS-ESTIMATE.md`).
**Cheapest guard:** WO fields declared **before** the seat runs. Matcher only reads. Backfill ⇒ DISCARD.

## 5. `re` set and wrong, fall through to R2/R3

**Failure:** a mis-quoted receipt **names** another WO by digest or `enqueue_id`.
**Trigger:** skip R1 when `re` present; first-wins among digest hits (`RECEIPT-MATCHING.md`).
**Symptom:** quoted receipt binds the wrong unit; true WO unmatched or double-closed.
**Cheapest guard:** `re` set ∧ `re≠unit` ⇒ DISCARD that receipt. No fall-through.

## 6. First-wins / `n≠1` still emits **matched**

**Failure:** one of many names is **the** pair. Completeness of the join.
**Trigger:** two WOs one prefix; two receipts one `re`; `n_wo` omitted.
**Symptom:** wrong CLOSED; the other stays OPEN or also “named.”
**Cheapest guard:** emit only if `n_wo==n_receipt==1`. Else ERROR, not FOUND.

## 7. Fuzzy / stem / Levenshtein / topic names it

**Failure:** similarity is a **completed** look (`LEGACY-26.md`). Third party cannot re-derive.
**Trigger:** cosine; containment; chat subject; “looked close.”
**Symptom:** 494→74; two-factor cites a score; deterministic lie (`LEGACY-10.md`).
**Cheapest guard:** exact declared keys only. Similarity ⇒ `model_guess`, never match.

## 8. Empty / template `expected` + a name

**Failure:** stub digest is shared. Receipt **names** (R1) or R2 hits **N** WOs.
**Trigger:** `expected=sha256(b"")`; public template; 494 share one hash (`LEGACY-32.md`).
**Symptom:** one receipt closes many; `n≠1` ignored; blast asserted as 1.
**Cheapest guard:** empty-hash refuse at declare. R2 with shared digest ⇒ neither matches.

## 9. Failed look `[]` ⇒ unmatched=0 ⇒ **all named** / nothing to verify

**Failure:** instrument did not finish. Zero is **complete** (`LEGACY-33.md`).
**Trigger:** timeout; wrong UTC partition; wrap; canary skip (`POSITIVE-CONTROL.md`).
**Symptom:** `unmatched=0`; 54+ still OPEN; cron 0 (`TASK-34-METRIC-THAT-LIES.md`).
**Cheapest guard:** canary-hit missing ⇒ UNKNOWN, not 0. UNMATCHED omitted on look fail ⇒ ERROR. Empty ≠ named.

## 10. UNMATCHED bag treated as COMPLETED / “nothing to verify”

**Failure:** leftover names **closed** by absence of a pair (`LEGACY-31.md`).
**Trigger:** 54 no-file; R2/R3 miss; delete unmatched so next scan is clean (`STALE-CLAIM-RECOVERY.md`).
**Symptom:** dashboard caught-up; receipts gone; never-issued look.
**Cheapest guard:** unmatched stays UNVERIFIED with a row. Do not delete. Do not mint a new WO id for the 494.

## 11. R2 / R3 inferred used to `released` / COMPLETED

**Failure:** digest+path or `enqueue_id` **closes** (`EVIDENCE-TIERING.md`). Name was never R1.
**Trigger:** 494 “good enough”; missing `re`; `""==""` on `enqueue_id`.
**Symptom:** cartesian or template closes; `list_next` omits `qid`.
**Cheapest guard:** inferred may display, not close. R3 requires both `enqueue_id` non-empty. COMPLETED only A∧B.

## 12. Stale leftover receipt still **names** the retry

**Failure:** previous attempt’s row is **this** unit done (`LEGACY-05.md`).
**Trigger:** same `re`; hot receipt without live claim; archive row treated as B.
**Symptom:** skip forever; accident brake (`CRASH-LOOP-BRAKE.md`); or `C` presence-gates.
**Cheapest guard:** B after **this** checker hashed **this** payload. Archive names evidence, not done. Stale-archive then new claim.

## 13. Receipt **file** / sidecar / glob names the unit

**Failure:** path contains the id. Parser never read `re` (`ADVERSARIAL-EXISTS-SCANNER.md`).
**Trigger:** `out/<unit>.ok`; `ls` stem; mailbox subject (`ADVERSARIAL-MAILBOX-QUEUE.md`).
**Symptom:** FOUND; body `re` missing or other unit; digest uncited.
**Cheapest guard:** open, parse, `re==unit`. Filename is not a name. Exists ⇒ candidate only.

## 14. `568≈622` / same length = bag **named**

**Failure:** net count is reconciliation (`TASK-18-COUNT-RECONCILIATION.md`).
**Trigger:** `len` shortcut; 54 holes + extras cancel; `"ok"` vs `"ok "`.
**Symptom:** “almost done”; 494 unquoted unjoined; 74 over-closed.
**Cheapest guard:** bags of `re`+digest, not lengths. 568≠622 is not a match rate.

## 15. Append receipt onto the payload that it names

**Failure:** name still matches; `d` moved (`LEGACY-21.md`).
**Trigger:** “add re to the bundle”; pretty-print after hash.
**Symptom:** A fails; B cites old `expected`; both look named.
**Cheapest guard:** receipt is a different object. Checker recomputes both hashes every look.

## 16. Dual matcher: prefix on the run path, `re` in the README

**Failure:** the namer that **wins** is the old one (`LEGACY-15.md`).
**Trigger:** cron still `name[:8]`; CI tests TASK-04; `C` local join (`DEPENDENCY-INVERSION.md`).
**Symptom:** split-brain matched/unmatched; change-detector skips the exact join (`LEGACY-09.md`).
**Cheapest guard:** tombstone prefix match so it raises (`LEGACY-16.md`). One published predicate: `re` then A∧B.

## 17. Canary-miss **named** (matcher too wide) or hit **missing** and join continues

**Failure:** controls failed; remaining names **trusted** (`POSITIVE-CONTROL.md`).
**Trigger:** no miss fixture; exists-only canary; skip controls “to finish 622.”
**Symptom:** FOUND on everything; or NONE with `unmatched=0`.
**Cheapest guard:** miss in hits ⇒ ERROR. Hit missing ⇒ whole join UNKNOWN. Do not close the rest.

## 18. mtime / size / `scheduled_utc` as the name

**Failure:** same second / same size **paired**. Clock is identity (`TASK-19-TIMESTAMP-HAZARD.md`).
**Trigger:** “closest timestamp”; size equality; priority join (`LEGACY-36.md` misused).
**Symptom:** two units one second swap; DST skew; wrong CLOSED.
**Cheapest guard:** do not join on clock, size, or priority. Those are order/display only.

## 19. Torn / pretty receipt: `re` key present ⇒ **named**

**Failure:** prefix JSON parses (`LEGACY-26.md`). Name **held**.
**Trigger:** crash mid-write; no trailing newline; worker hash-then-normalise (`TASK-15-IDEMPOTENT-RECEIPT.md`).
**Symptom:** match on a truncated object; citation ≠ file bytes.
**Cheapest guard:** refuse torn/no-newline. Recompute `sha256` of canonical bytes. Partial `re` ⇒ ERROR.

## 20. Peer / model / `P` ACK: “receipt named it”

**Failure:** a sentence is the join (`EVIDENCE-TIERING.md`, `LEGACY-19.md`).
**Trigger:** unmatched list ingested; chat ACK (`BURST-REVIEW.md`); summary line (`ADVERSARIAL-SELF-SUMMARY.md`).
**Symptom:** this look never ran; resume greens (`LEGACY-29.md`).
**Cheapest guard:** DISCARD peer conclusions. Re-run MEASURE. ACK is `model_guess`.

## 21. New `unit` minted so a receipt can name it

**Failure:** old WO **handled** (unmatched); new id **matched**. Two effects or one lost (`TASK-27-RETRY-SEMANTICS.md`).
**Trigger:** “help the 494”; UUID per retry; hash includes `at_utc`.
**Symptom:** original never closed by A∧B; duplicate delivery or orphan (`ORPHAN-RECLAMATION.md`).
**Cheapest guard:** same `unit` / `enqueue_id`. New id to fix a join ⇒ refuse.

## 22. Case-fold / NFC / two spellings, one name

**Failure:** collision **absorbed** or lookup NONE after write.
**Trigger:** `Unit` vs `unit`; composed Unicode; Windows receipt store.
**Symptom:** one receipt names two WOs; or exact `re` miss.
**Cheapest guard:** `re`/`unit` canonical exact bytes. Homoglyph ⇒ not a match.

## 23. Path-unsafe receipt names a unit outside the root

**Failure:** join **succeeded**. Artifact escaped (`TASK-32-PATH-SAFETY.md`).
**Trigger:** `re` exact, `path=../out` or `/tmp/done`.
**Symptom:** COMPLETED; checker refuses hash; audit has nothing in-repo (`TASK-09-RETENTION-RISK.md`).
**Cheapest guard:** validate path before join. Escape ⇒ not a match, even if `re` equals.

## 24. `released` / `I` / monitor on “named”

**Failure:** queue **done**; quota **spent**; `errors=0` because a receipt named a row.
**Trigger:** `released` on R1 without A; billing on match; `M` counts unmatched only (`BUDGET-AWARE-ROUTING.md`).
**Symptom:** `list_next` omits; included gone; greens (`SILENT-FAILURE-DETECTION.md`).
**Cheapest guard:** `released` only after this process’s A∧B. Decrement `I` then. Alert on stale scan, not on `unmatched==0`.

## 25. Archive / hot / two stores: a name in the wrong bag

**Failure:** archive receipt **closes** live work, or hot leftover **names** after archive (`PROVENANCE-ARCHIVE.md`).
**Trigger:** `C` globs archive; month index only; wrap drops the 74.
**Symptom:** FOUND on archived stub; or unmatched=0 after eviction.
**Cheapest guard:** B is the checker log row for **this** payload digest. Archive is evidence, not a live name. Wrap without canary ⇒ UNKNOWN.

## 26. `list_next` / UI greens by “has a receipt name”

**Failure:** presence of any naming receipt is **status** and **order** (`LEGACY-35.md`).
**Trigger:** `ls` receipts; skip units that have a file; badge from `re` without A.
**Symptom:** 74 first and closed; 494+54 buried; `p0` waits.
**Cheapest guard:** `order_key` from WO body. Name without A∧B does not green or skip.
