# adversarial review: a tool's summary line about its own success

Mechanism: the process prints a last line / banner / count — `OK`, `N passed`, `DONE`, `0 errors`, `status=completed`, `finished in Xs` — and callers treat that **string** as COMPLETED, FOUND, skip, or safe. Confident wrong = **success, completeness, or safety** that does not hold.

## 1. Author of the work is author of **OK**

**Failure:** the summary is a record, not a check (`TASK-10-SELF-REPORT.md`). Gate copies it (`LEGACY-17.md`).
**Trigger:** `print("DONE")`; `status=completed`; model “covered”; wrapper `if "OK" in last_line`.
**Symptom:** COMPLETED; no Sig1∧Sig2; third party cannot re-derive (`LEGACY-06.md`).
**Cheapest guard:** no write path to COMPLETED. Summary is `model_guess` (`EVIDENCE-TIERING.md`). Close only A∧B (`DUAL-SIGNATURE-COMPLETION.md`).

## 2. Line printed before wait / fsync / child exit

**Failure:** “success” is **launch**. Bytes missing or torn (`TASK-31-GRACEFUL-SHUTDOWN.md`).
**Trigger:** `print("OK")` then `Popen`; SIGTERM then `status=completed` (`ADVERSARIAL-CRON-10M.md` §6); `last_ok` at start (§2).
**Symptom:** green last line; path empty; next seat reads settled (`LEGACY-31.md`).
**Cheapest guard:** no summary until wait **and** MEASURE. Crash after the line ⇒ UNVERIFIED, not COMPLETED.

## 3. Partial bag: K passed, rest unrun, session **OK**

**Failure:** completeness of the **printed count**, not of the declared set.
**Trigger:** `--exitfirst`; `max_launch=1`; `break` on first 0; skipped/xfail counted as pass (`ADVERSARIAL-CRON-10M.md` §5).
**Symptom:** `N passed`; tail OPEN; `unmatched=0` (`TASK-34-METRIC-THAT-LIES.md`).
**Cheapest guard:** OK only if MEASURE of the full declared bag + both controls. Partial ⇒ typed `REMAINING` ≠ OK.

## 4. Zero tests / zero errors / empty stdout = **OK**

**Failure:** instrument ran nothing (or failed quiet). Count **0** is health (`TASK-18-COUNT-RECONCILIATION.md`).
**Trigger:** collect-only; wrong `TREE`; `[]` from timeout (`POSITIVE-CONTROL.md`); `if not stdout: OK`.
**Symptom:** `0 errors`; `1..0`; cron 0; work on disk (`SILENT-FAILURE-DETECTION.md`).
**Cheapest guard:** 0 is UNKNOWN unless a completed look + canary-hit/miss. Empty stdout is not NONE. Missing bag ⇒ ERROR.

## 5. Failed look dressed as “no issues”

**Failure:** `find_*` ERROR/`[]` → summary `nothing due` / `unmatched=0`.
**Trigger:** store timeout; hook-blocked MEASURE (`ADVERSARIAL-CMD-HOOK.md` §5); wrap (`TASK-22-ROLLING-WINDOW-AUDIT.md`).
**Symptom:** last line OK; receipts exist; `last_successful_scan` stale (`LEGACY-04.md`).
**Cheapest guard:** look ERROR ⇒ summary ERROR, never OK. Canary-hit missing ⇒ not “no issues.”

## 6. Caller greps `OK` / last line / TAP, ignores the body

**Failure:** one matching token is **the** result. ERROR lines above are commentary (`LEGACY-25.md`).
**Trigger:** CI `grep -q OK`; `tail -1`; `tee` then `echo OK` (`ADVERSARIAL-CRON-10M.md` §10); “NOT OK” still matches `OK`.
**Symptom:** pipeline 0; log has ERROR; unit CLOSED.
**Cheapest guard:** parse a typed object (`kind`, `tier`), not a substring. `OK` in text is not `kind`. Exit = worker status only.

## 7. Model / seat writes the line; checker never ran

**Failure:** guess stamped as **deterministic** (`EVIDENCE-TIERING.md`). Resume inherits it (`TASK-05-DOCTRINE-DRAFT.md`).
**Trigger:** `W` API `set_completed`; “strong model reviewed the digest”; `S` `status=covered`.
**Symptom:** UI green badge; `verify` UNVERIFIED; `list_next` skips (`LEGACY-09.md`).
**Cheapest guard:** model must not write `kind` or `tier`. Guess annotates; DISCARD as peer analysis (`LEGACY-19.md`). Override of det ⇒ UNVERIFIED + `R`.

## 8. `exists` / “N files written” in the summary

**Failure:** presence or a count is **closed** (`LEGACY-05.md`).
**Trigger:** `wrote 3 artifacts`; sidecar `.ok`; size>0; `out/` non-empty (`ADVERSARIAL-EXISTS-SCANNER.md`).
**Symptom:** FOUND; digest ≠ `expected`; shared `latest` closes N units.
**Cheapest guard:** summary must not close. Hit = hash + `re`. Count of names is not A∧B.

## 9. Wrong identity in the line (“matched 20260910-…”)

**Failure:** prefix / thread / filename **matched**. This `re` is not (`RECEIPT-MATCHING.md`).
**Trigger:** `name[:8]`; “covered unit”; chat subject (`TASK-07-DATE-ROLLOVER.md`).
**Symptom:** `matched` in the banner; 494 unquoted CLOSED; canary-miss FOUND.
**Cheapest guard:** line is not a join. `receipt.re==unit` only. Prefix in a summary ⇒ `model_guess`, not FOUND.

## 10. HTTP 200 / `kind=sent` / 402-escalated **OK**

**Failure:** transport or disposition is **delivered** (`IDEMPOTENT-EVENT-LEDGER.md`).
**Trigger:** SDK “ok”; dead-letter “handled” (`ADVERSARIAL-QUOTA-RETRY.md` §8); `released` on send.
**Symptom:** last line success; no Sig1; `I` decremented (`BUDGET-AWARE-ROUTING.md`).
**Cheapest guard:** 200/sent/escalated are not COMPLETED. Decrement `I` only after this process’s Sig1∧Sig2.

## 11. Cached / previous-run / rotated last line

**Failure:** today’s tool **succeeded** with yesterday’s sentence.
**Trigger:** LKG overwrite (`TASK-08-WATCHDOG-PATTERN.md`); log rotate scrape (`TASK-30-LOG-ROTATION.md`); `scan.ok` leftover.
**Symptom:** fresh `OK`; look did not run; episode quiet (`TASK-34-METRIC-THAT-LIES.md`).
**Cheapest guard:** summary must cite this incarnation `{pid,start}` and this `ask`. Stale/rotated line ⇒ not this run. LKG only after a healthy completed probe.

## 12. Self-minted control then “both controls passed”

**Failure:** tool created the canary it then reports (`LEGACY-17.md`).
**Trigger:** `touch canary-hit`; dummy receipt so `[]` avoided; exists-only canary (`POSITIVE-CONTROL.md`).
**Symptom:** `control.hit=true` on the line; table wrapped; NONE trusted.
**Cheapest guard:** controls are issuer data, not this process. Mint ⇒ ERROR. Missing hit ⇒ not “passed.”

## 13. Append the summary onto the payload

**Failure:** line says success; digest **moved**; citations orphan (`LEGACY-21.md`).
**Trigger:** “add a healthy note”; receipt on `out/<unit>`; JSON pretty-print after hash.
**Symptom:** `OK` in the file; `verify` UNVERIFIED; `C` still presence-gates.
**Cheapest guard:** payload immutable. Summary is a separate row, `tier=model_guess`. Re-hash every look.

## 14. Dual predicate: summary wins over `verify`

**Failure:** hasher UNVERIFIED; banner OK is on the run path (`LEGACY-15.md`).
**Trigger:** README says A∧B; cron greps `OK`; `C` prefer local (`DEPENDENCY-INVERSION.md`).
**Symptom:** split-brain; change-detector skips the checker (`LEGACY-09.md`).
**Cheapest guard:** tombstone `if summary.ok: return COMPLETED` (`LEGACY-16.md`). Deterministic wins every disagreement.

## 15. Duration / “finished” / “ran to completion” as success

**Failure:** the process **exited**. Work did not meet `expected`.
**Trigger:** `finished in 1.2s`; systemd `Deactivated successfully`; cron CMD log (`ADVERSARIAL-CRON-10M.md` §24).
**Symptom:** operators read elapsed as closed; payload stub.
**Cheapest guard:** finished is wait status, not MEASURE. Time is not A∧B.

## 16. `tier` omitted ⇒ line read as deterministic

**Failure:** empty label gets the strongest trust (`EVIDENCE-TIERING.md`).
**Trigger:** no `tier` key; UI defaults green; inferred FOUND overrides NONE.
**Symptom:** guess closes; canary-miss hidden; `R` unlabelled.
**Cheapest guard:** omit `tier` ⇒ treat as `model_guess`, not deterministic. Inferred cannot close.

## 17. Parallel / last-writer: one OK covers later FAIL

**Failure:** summary from the first or last shard is **the** session.
**Trigger:** interleaved pytest; two seats (`LEGACY-01.md`); mailbox batch ACK (`ADVERSARIAL-MAILBOX-QUEUE.md` §22).
**Symptom:** `OK` after a fail in another worker; bag closed.
**Cheapest guard:** one typed object after **all** shards’ MEASURE. Any shard ERROR ⇒ not OK. No last-line race.

## 18. Skip / xfail / `do_not_retry` printed as pass

**Failure:** not-run or escalate **counted** in `N passed`.
**Trigger:** pytest skip; 402 dead-letter (`TASK-28-ERROR-CLASSIFICATION.md`); `DRY_RUN` leftover.
**Symptom:** 100% passed; units UNVERIFIED; quota jobs “handled.”
**Cheapest guard:** skip/escalate are distinct kinds, not pass. Session OK forbidden if any skip of a declared obligation.

## 19. Help / banner / fixture text grepped as the result

**Failure:** `--help` or a docstring contains `success`/`OK`. CI **green**.
**Trigger:** usage printed on bad argv (exit 2) + grep OK; self-test name `test_ok`; ANSI color codes.
**Symptom:** pipeline 0; main never ran; or FAIL hidden by color.
**Cheapest guard:** result is a single canonical JSON object on stdout, one newline. Help is stderr. Grep banned.

## 20. Peer summary ingested as this tool’s success

**Failure:** another seat’s “healthy” becomes **our** last line (`LEGACY-19.md`).
**Trigger:** copy LKG; `P` ACK (`BURST-REVIEW.md`); weak pre-process (`LEGACY-20.md`).
**Symptom:** this ask never measured; resume green (`LEGACY-29.md`).
**Cheapest guard:** DISCARD peer conclusions. Re-run MEASURE of declared state. Chat/ACK is not a summary of this process.

## 21. `I` / monitor decremented on the line

**Failure:** quota **spent**; dashboard **no errors** because the tool said OK.
**Trigger:** billing on `print("sent")`; `M` counts ERROR tokens, not guesses (`SILENT-FAILURE-DETECTION.md`).
**Symptom:** included gone; A∧B false; greens.
**Cheapest guard:** decrement only after this process’s Sig1∧Sig2. Alert on stale scan, not on an OK string.

## 22. Same length / same “N passed” after a swap

**Failure:** `len` shortcut: 3 passed then 3 different ids, banner unchanged (`TASK-18-COUNT-RECONCILIATION.md`).
**Trigger:** reconcile by count; `ok` vs `ok `; one `"a"` swapped for `"b"`.
**Symptom:** “nothing changed” / still 3 passed; bag different; whitespace closes.
**Cheapest guard:** reconcile as bags of `re`+digest, not lengths. `"ok"` ≠ `"ok "`.

## 23. Exception / UNKNOWN mapped to a success sentence

**Failure:** timeout, torn JSON, hook refuse — last line still `OK` or `NONE` (`LEGACY-26.md`).
**Trigger:** `except: print("done")`; keep cron green (`POSITIVE-CONTROL.md`); `|| echo OK`.
**Symptom:** exit 0; look did not finish; `[]` stored as settled.
**Cheapest guard:** except/timeout ⇒ `kind=ERROR`, no OK line. Wrapper must not mint NONE/OK.

## 24. Summary names a path outside the root / `/tmp/done`

**Failure:** line **points** at an artifact. Checker refuses; record still completed (`TASK-10-SELF-REPORT.md`).
**Trigger:** `published /tmp/...`; `..` (`TASK-32-PATH-SAFETY.md`).
**Symptom:** OK + UNVERIFIED; next audit has nothing to re-hash (`TASK-09-RETENTION-RISK.md`).
**Cheapest guard:** out-of-root path ⇒ ERROR, not a successful publish. Line cannot close.

## 25. Crash-loop / brake inferred from three “failed” lines — or cleared by one “OK”

**Failure:** `W` sentences are the brake **or** the all-clear (`CRASH-LOOP-BRAKE.md`).
**Trigger:** `n` = grep failed; model “looks fixed”; `P` ACK.
**Symptom:** DESIGN skip (work frozen as safe) or ACCIDENT retried; both from prose.
**Cheapest guard:** DESIGN only wrapper `crash_loop` after `blocked→interrupted`. One OK line does not clear. Exemption ≠ summary.

## 26. `list_next` / UI sorts or greens by “model is sure”

**Failure:** the sure summary is **priority** and **status** (`EVIDENCE-TIERING.md`).
**Trigger:** badge from last line; `order_key` ignored (`LEGACY-35.md`); inherit guess on resume.
**Symptom:** wrong first row; COMPLETED badge on UNVERIFIED; live `p0` buried.
**Cheapest guard:** `list_next` by `order_key`. Guess never greens. Resume is cold admit; last summary dies with the seat (`LEGACY-29.md`).
