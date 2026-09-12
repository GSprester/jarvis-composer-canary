# adversarial review: regex extractor inferring deliverables from prose

Mechanism: a pattern (`DONE`, path, `#issue`, checkbox, hash-shaped hex, “instead”) **captures** obligations from chat/docs/logs and treats the capture as the bag — COMPLETED, OPEN=empty, SUPERSEDED, or skip. Confident wrong = **success, completeness, or safety** that does not hold.

## 1. Token `DONE` / `OK` / `shipped` ⇒ deliverable **closed**

**Failure:** a word is exact cited evidence. A∧B never ran (`TASK-10-SELF-REPORT.md`).
**Trigger:** `/done/i`; message DONE cite without `evidence_ids`; umbrella “all done.”
**Symptom:** COMPLETED; no digest; 54+ WOs never declared (`RECEIPT-MATCHING.md`).
**Cheapest guard:** extract is `model_guess` (`EVIDENCE-TIERING.md`). Close only A∧B on a **pre-declared** `(path, expected)` (`LEGACY-32.md`). Prose is not a receipt.

## 2. Captured path / filename / `exists` is the deliverable

**Failure:** markdown link or `out/foo` **names** work. Presence or prefix is done (`LEGACY-05.md`, `ADVERSARIAL-RECEIPT-NAMES.md`).
**Trigger:** `` `[\w./-]+` ``; `YYYYMMDD-` stem; stale pointer still in the thread.
**Symptom:** FOUND on leftover/empty; display hash ≠ canonical (`TASK-15-IDEMPOTENT-RECEIPT.md`).
**Cheapest guard:** path is a candidate after lexical validate (`TASK-32-PATH-SAFETY.md`). Done iff this process hashed it == `expected`. Stale pointer ⇒ UNVERIFIED, not CLOSED.

## 3. Tool / quoted / system text extracted as owner obligations

**Failure:** the bag is **complete** relative to templates, not the human ask.
**Trigger:** regex does not distinguish quote fences, “you should,” process paste (gold-set: owner vs tool).
**Symptom:** extra WOs CLOSED by tool receipts; real owner lines missed ⇒ empty remainder “done.”
**Cheapest guard:** only owner-authored spans (declared speaker/id). Quoted/tool/system ⇒ not an obligation. Ambiguity ⇒ UNCERTAIN, not COMPLETED.

## 4. Owner lines missed ⇒ `[]` ⇒ no deliverables ⇒ **caught up**

**Failure:** failed or narrow extract looks like **zero work** (`POSITIVE-CONTROL.md`).
**Trigger:** timeout; ReDoS abort; wrong language; DOTALL miss; canary skip (`LEGACY-33.md`).
**Symptom:** `unmatched=0`; OPEN work in prose; cron 0 (`TASK-34-METRIC-THAT-LIES.md`).
**Cheapest guard:** no-match is UNKNOWN unless a completed look + canary-hit (a declared obligation **must** capture). Empty ⇒ not NONE of the estate.

## 5. Multipart collapsed to one capture

**Failure:** one string, one receipt, **N** obligations closed (gold-set: preserve multipart).
**Trigger:** greedy `.*`; first bullet wins; “and” not split.
**Symptom:** partial evidence closes the umbrella; tail UNVERIFIED read as settled (`LEGACY-31.md`).
**Cheapest guard:** one obligation per declared id. `n≠1` ⇒ ERROR, not a match (`RECEIPT-MATCHING.md`). Collapse is not completeness.

## 6. SUPERSEDED from topical similarity / “instead” without `successor_id`

**Failure:** old obligation **retired**. Successor never named (`LEGACY-15.md`).
**Trigger:** cosine/stem; `/instead|ignore previous|supersede/i`; no message id.
**Symptom:** two predicates live; prefix matcher still on cron; next seat reimplements.
**Cheapest guard:** SUPERSEDED iff explicit `successor_id` to a real message **and** old callable tombstoned. Similarity ⇒ UNCERTAIN.

## 7. Gate declares `unit` / `expected` from the capture

**Failure:** regex mints the join it then accepts (`LEGACY-17.md`).
**Trigger:** `expected = sha256(captured)`; `unit = group(1)`; hash from display hex in prose.
**Symptom:** stub/template digest closes; `R` = every extract from that writer (`BLAST-RADIUS-ESTIMATE.md`).
**Cheapest guard:** declare **before** extract. Captured hex is a citation to check, not a new `expected`. Backfill ⇒ DISCARD.

## 8. Writer authors the prose the regex then “finds”

**Failure:** `W` prints `DONE: out/bundle` / checkbox; extractor **succeeds** (`LEGACY-20.md`).
**Trigger:** self-summary line (`ADVERSARIAL-SELF-SUMMARY.md`); model dumps the list it wants closed.
**Symptom:** authority laundering; strong seat never sees declared bytes.
**Cheapest guard:** extractor is not a gate. DISCARD self-authored spans (`LEGACY-19.md`). Checker ≠ worker `pid+start`.

## 9. First-wins among many captures / `n≠1`

**Failure:** one of several paths or hashes is **the** deliverable.
**Trigger:** two files in one sentence; two hexes (canonical vs display); overlapping groups.
**Symptom:** wrong CLOSED; the other OPEN hidden inside the same line.
**Cheapest guard:** emit only if exactly one candidate per obligation id. Else UNCERTAIN/ERROR.

## 10. Umbrella / checkbox-all / “the above”

**Failure:** one later token binds every prior extract (gold-set: false umbrella).
**Trigger:** DOTALL; `/all (of the )?above/`; `- [x]` on a parent.
**Symptom:** multipart OPEN marked COMPLETED; no per-id evidence.
**Cheapest guard:** each obligation needs its own cited `evidence_ids`. Parent check is not A∧B for children.

## 11. Display hash / truncated hex / `name[:8]` from prose

**Failure:** a hex-shaped group **is** `expected` or `re` (`TASK-07-DATE-ROLLOVER.md`).
**Trigger:** `[a-fA-F0-9]{7,64}`; git short; “canonical vs display” gold-set.
**Symptom:** 74-style prefix close; canary-miss FOUND; wrong object cited.
**Cheapest guard:** only 64-hex **and** this process hashes the file. Short/display ⇒ not a digest.

## 12. `](..)` / `/tmp` / `..` captured as a safe path

**Failure:** extract **succeeded**. Write/verify would escape (`TASK-32-PATH-SAFETY.md`).
**Trigger:** markdown links; Windows `\`; `file://`.
**Symptom:** COMPLETED on an out-of-root name; in-repo bag empty.
**Cheapest guard:** lexical validate **before** join. Escape ⇒ not a deliverable.

## 13. Inferred list used to `released` / skip / `list_next`

**Failure:** regex bag is **the** queue (`EVIDENCE-TIERING.md`).
**Trigger:** “already triaged”; change-detector on extract hash (`LEGACY-09.md`); `C` prefer local (`DEPENDENCY-INVERSION.md`).
**Symptom:** declared WOs never launched; extract hash stable ⇒ skip forever.
**Cheapest guard:** inferred/guess may display. Launch and close only declared `unit`s + A∧B. Extract hash is not a skip key.

## 14. ReDoS / timeout / torn thread ⇒ last extract or `[]` **OK**

**Failure:** look did not finish (`LEGACY-26.md`). Partial captures treated as the full bag.
**Trigger:** catastrophic backtrack; truncated scrape; wrap of chat (`TASK-22-ROLLING-WINDOW-AUDIT.md`).
**Symptom:** COMPLETED on a prefix of the thread; or empty=done (§4).
**Cheapest guard:** timeout/torn ⇒ ERROR, not a bag. No OK line (`ADVERSARIAL-SELF-SUMMARY.md`). Canary obligation must still capture.

## 15. Dual predicate: regex on the run path, declared fields in the README

**Failure:** extractor **wins** (`LEGACY-15.md`). TASK-04 `re` never runs.
**Trigger:** cron greps prose; CI tests gold-set labels only; `pip` old extractor.
**Symptom:** split-brain OPEN/COMPLETED; tombstone missing.
**Cheapest guard:** tombstone the regex closer so it raises (`LEGACY-16.md`). One published predicate: declared keys + A∧B.

## 16. Canary-miss: pattern too wide names everything

**Failure:** every sentence **has a deliverable**. Completeness of extract (`POSITIVE-CONTROL.md`).
**Trigger:** no miss fixture; `/\w+/`; path regex hits `canary-miss` text.
**Symptom:** FOUND on quotes/templates; real `re` never tested (`RECEIPT-MATCHING.md`).
**Cheapest guard:** canary-miss must **not** capture. Miss in hits ⇒ ERROR for the whole extract.

## 17. Conflicting receipts; regex picks one

**Failure:** one cite **settles**. Conflict is UNCERTAIN (gold-set).
**Trigger:** two hexes; DONE + “blocked”; first group.
**Symptom:** COMPLETED despite conflict; `n≠1` ignored.
**Cheapest guard:** conflict ⇒ UNCERTAIN, not a match. Do not first-wins.

## 18. New `unit` / `enqueue_id` per captured line

**Failure:** prose mint **is** the WO. Retry/duplicate (`TASK-27-RETRY-SEMANTICS.md`).
**Trigger:** UUID per bullet; hash includes the sentence; “help” the 494.
**Symptom:** original declared id unmatched; two shells; extract bag “complete.”
**Cheapest guard:** extract may attach to an existing `unit`. New id from prose ⇒ refuse.

## 19. `- [x]` / URL / issue number as Sig2

**Failure:** a checkbox or `#123` **names** the task (`ADVERSARIAL-RECEIPT-NAMES.md`).
**Trigger:** GFM tasks; `fixes #`; HTTP 200 on the URL.
**Symptom:** B without A; worker can tick the box (`LEGACY-17.md`).
**Cheapest guard:** those tokens are guesses. B is checker receipt `re` + cited digest only.

## 20. Civil date / `YYYYMMDD` in prose becomes `re`

**Failure:** calendar text **is** identity (`TASK-19-TIMESTAMP-HAZARD.md`).
**Trigger:** `/20\d{6}/`; timezone in the thread; two items one day.
**Symptom:** prefix join; UTC rollover collision (`TASK-07-DATE-ROLLOVER.md`).
**Cheapest guard:** `unit` is declared `re`, not a date capture. Date is display.

## 21. Peer extract / resume inherits the list

**Failure:** another seat’s captures are **this** bag (`LEGACY-19.md`, `LEGACY-29.md`).
**Trigger:** weak pre-process (`LEGACY-20.md`); cold start loads `extracted.json`; `P` ACK.
**Symptom:** strong seat never sees messages; detectors skip.
**Cheapest guard:** DISCARD peer extracts. Resume is cold admit of declared state only.

## 22. UNCERTAIN mapped to COMPLETED or to skip-OPEN

**Failure:** ambiguity **resolved** by the regex engine’s success bit.
**Trigger:** `if match: COMPLETED`; `if uncertain: skip` as nothing-due; omit `tier` ⇒ deterministic (`EVIDENCE-TIERING.md`).
**Symptom:** vague approval CLOSED; or OPEN buried and `list_next` empty.
**Cheapest guard:** UNCERTAIN is a first-class kind. It does not close and does not mean idle. Omit `tier` ⇒ `model_guess`.

## 23. `len(captures)` / same count as citations = **all covered**

**Failure:** net count is reconciliation (`TASK-18-COUNT-RECONCILIATION.md`).
**Trigger:** 3 bullets, 3 hashes, swapped pairs; `"ok"` vs `"ok "`.
**Symptom:** “extract complete”; bags differ; whitespace closes.
**Cheapest guard:** join by exact `evidence_ids` + digest, not lengths.

## 24. `I` / monitor / UI green on “extracted OK”

**Failure:** quota **spent**; dashboard **no errors** because the regex returned a list.
**Trigger:** billing on capture count; `M` ignores UNCERTAIN (`SILENT-FAILURE-DETECTION.md`); badge from last line.
**Symptom:** included gone; obligations UNVERIFIED; `order_key` ignored (`LEGACY-35.md`).
**Cheapest guard:** decrement `I` only after A∧B. Extract success is not a scan. `list_next` by WO `order_key`.

## 25. Encoding / homoglyph / markdown escape: miss real, hit lookalike

**Failure:** `ＤONE` / `out\/a` / NFC path **is** or **isn’t** a capture, confidently.
**Trigger:** Unicode; backslash-escapes; HTML entities; ANSI in logs.
**Symptom:** owner obligation dropped (§4) or false CLOSED on a lookalike path (§2).
**Cheapest guard:** normalize then **exact** declared-field match. Homoglyph ⇒ UNCERTAIN. Do not regex raw HTML as the ask.

## 26. Adversarial system label / code fence plants a successor or DONE

**Failure:** planted `successor_id=` or `COMPLETED` in a fence. Extract **SUPERSEDED/CLOSED** (gold-set).
**Trigger:** model/system message; `<!-- DONE -->`; JSON in a quote the regex treats as structured.
**Symptom:** owner work retired; fake successor; two leads on labels (`LEGACY-01.md`).
**Cheapest guard:** system/tool/fence spans cannot set status or `successor_id`. Only owner message ids + A∧B. Planted structured text ⇒ UNCERTAIN.
