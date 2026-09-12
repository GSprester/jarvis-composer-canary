# adversarial review: scanner reporting completion by file existence

Mechanism: a scanner (due-bag, receipt look, presence gate, change-detector) treats `exists(path)` / `is_file` / `lexists` / “any name in `out/`” as **FOUND**, **COMPLETED**, **skip**, or **safe**. Confident wrong = **success, completeness, or safety** that does not hold.

## 1. Leftover empty or stale file ⇒ FOUND / COMPLETED / skip

**Failure:** a name is on disk. Obligation **closed**. Bytes are `b""`, last week’s payload, or a preview (`LEGACY-05.md`).
**Trigger:** create-then-crash; retry of the same path; `if output.exists(): skip` (`LEGACY-09.md`).
**Symptom:** job never writes `expected`; drain greens; third-party hash UNVERIFIED (`TASK-06-ARTIFACT-CHECK.md`).
**Cheapest guard:** COMPLETED iff `sha256(canonical bytes)==expected` ∧ `expected≠empty` (`LEGACY-32.md`). Exists is a candidate only. Empty/stale must **run**.

## 2. Scanner mints the file, then exists

**Failure:** the look **satisfies itself**. Gate authored the evidence (`LEGACY-17.md`).
**Trigger:** `if not p.exists(): p.write_bytes(b"")`; `touch` canary so the scan “works” (`POSITIVE-CONTROL.md`); `find_receipts` inserts a dummy row.
**Symptom:** every subsequent scan FOUND/COMPLETED; digest empty; canary-hit is a stub.
**Cheapest guard:** scanner is MEASURE only. It must not create, `touch`, or rewrite the paths it scores. Missing canary ⇒ ERROR, not mint.

## 3. Exists without `re`: filename / prefix / glob is the hit

**Failure:** **some** file exists. Scanner reports **this** unit complete.
**Trigger:** `name[:8]`; `out/*.json`; `DONE` / `unit.ok` (`DUAL-SIGNATURE-COMPLETION.md`); 74 quoted names (`RECEIPT-MATCHING.md`).
**Symptom:** 494 unquoted CLOSED; canary-miss FOUND; wrong day’s receipt (`TASK-07-DATE-ROLLOVER.md`).
**Cheapest guard:** hit cites `re==unit` **and** digest. Filename is display. Canary-miss in FOUND ⇒ matcher is the defect.

## 4. Sidecar `.done` / `.ok` / `status=completed` file exists

**Failure:** author of the work is author of **closed** (`TASK-10-SELF-REPORT.md`). Payload missing or wrong.
**Trigger:** worker `touch out/id.ok`; wrapper copies `W`’s bit to a path; Sig2 implemented as a name.
**Symptom:** scanner FOUND; `verify(payload)` fails; next ticks skip.
**Cheapest guard:** sidecar names are not hits. Only checker Sig2 after this process hashed the declared payload path.

## 5. `expected` copied from the file that exists

**Failure:** gate mints the digest it accepts. Exists **and** hash now “match.”
**Trigger:** “helpful” verify after leftover; `W` supplies `expected` post-write (`TASK-24-ARTIFACT-DECLARATION.md`); empty-hash declared so `b""` closes.
**Symptom:** COMPLETED; bytes are the stub; `R` = every unit that shared the template (`BLAST-RADIUS-ESTIMATE.md`).
**Cheapest guard:** `expected` on the handoff **before** write, ≠ empty, not read from `W` or from the file. Scanner hashes and compares; it does not set `expected`.

## 6. Directory / “any file in `out/`” / non-empty listing = bag complete

**Failure:** completeness of a **tree**, not of each `enqueue_id`.
**Trigger:** `os.listdir` truthy; `out/` mkdir leftover; glob `len>0`; presence-gated consumer (`PROVENANCE-ARCHIVE.md`).
**Symptom:** one stub closes the lane; later units never scanned; `open_units` 0 (`TASK-34-METRIC-THAT-LIES.md`).
**Cheapest guard:** one hit per declared `(path, expected, re)`. Listing non-empty is not NONE/FOUND for the bag. Canary-hit required.

## 7. Exists-only canary blesses the rest of the scan

**Failure:** control **passed**. Remaining misses/hits are trusted.
**Trigger:** canary is `touch canary-hit`; scanner checks `exists(canary)` not `verify` (`POSITIVE-CONTROL.md`).
**Symptom:** wrapped table / wrong UTC day → NONE with `control.hit=true`; work invisible (`TASK-22-ROLLING-WINDOW-AUDIT.md`).
**Cheapest guard:** canary-hit is A∧B on declared bytes. Exists-only canary ⇒ treat as no control ⇒ ERROR.

## 8. Presence leftover = done **and** blocks relaunch

**Failure:** safety of “already handled.” Accident looks like a brake (`CRASH-LOOP-BRAKE.md`).
**Trigger:** hot receipt exists, no live claim, no Sig2 (`STALE-CLAIM-RECOVERY.md`); `C` gates on `exists`.
**Symptom:** dispatcher refuses; consumer treats path as complete; unit stuck UNVERIFIED.
**Cheapest guard:** exists without live `{pid,start}` and without A∧B ⇒ archive, not COMPLETED, not skip-forever. `C` waits on ledger head + digest, not presence.

## 9. Size > 0 or mtime “this run” without hash

**Failure:** a non-empty or freshly touched name is **this obligation**.
**Trigger:** `DONE:\n`; preview write; `touch` after fail; mtime as liveness (`LEGACY-14.md`).
**Symptom:** scanner FOUND; digest ≠ `expected`; `touch` looks live while holder is dead.
**Cheapest guard:** size and mtime are not A. Hash after canonicalise (`TASK-15-IDEMPOTENT-RECEIPT.md`). mtime is not a process.

## 10. One path exists, many units share it

**Failure:** first exists closes **N** WOs that declared the same filename or template digest.
**Trigger:** `out/latest`; shared `expected` across 494; `qid` from date only.
**Symptom:** blast radius asserted as 1; one stub, N COMPLETED.
**Cheapest guard:** path is lexical per `unit`. Shared `expected` or `latest` ⇒ refuse declare. Decisive check on one green row that is **not** the canary (`BLAST-RADIUS-ESTIMATE.md`).

## 11. Append / edit in place: exists stays true, digest moved

**Failure:** scanner still FOUND. Citations orphan (`LEGACY-21.md`).
**Trigger:** “add context” to `out/<unit>`; receipt appended onto payload; log rotate of the artifact.
**Symptom:** old `expected` UNVERIFIED; new bytes not declared; `C` still presence-gates.
**Cheapest guard:** payload is immutable after declare. Scanner re-hashes every look. Digest mismatch ⇒ UNVERIFIED, not “still exists so done.”

## 12. Symlink / hardlink / dangling `lexists`

**Failure:** a name exists in the namespace. Target is other bytes, other unit, or nothing.
**Trigger:** `ln -s` leftover; restore pointer to `archive/` (`PROVENANCE-ARCHIVE.md`); Windows junction; `exists` vs `is_file`.
**Symptom:** FOUND on the link; payload elsewhere or empty; or dangling ⇒ NONE while the real file sits under another name.
**Cheapest guard:** refuse unless regular file, `st_nlink==1`, not a symlink, `realpath` still the declared lexical path, then hash. `lexists` ∧ not `is_file` ⇒ ERROR.

## 13. Traversal / absolute path: exists **outside** counted as a hit

**Failure:** scanner **found** an artifact. Estate tree untouched or escaped.
**Trigger:** `unit=../out`; `/tmp/done`; resolve-before-validate (`TASK-32-PATH-SAFETY.md`).
**Symptom:** FOUND; live `open_units` unchanged; bytes not in the declared root.
**Cheapest guard:** lexical validate **before** join. Escape ⇒ not a hit (refuse). Do not hash outside.

## 14. Wrong tree / case-fold / day-dir: exists of a different bag

**Failure:** **that** listing is complete. The obligation’s path was not read.
**Trigger:** root crontab vs user `TREE`; `Unit.json` vs `unit.json`; `inbox/YYYYMMDD/` (`TASK-19-TIMESTAMP-HAZARD.md`).
**Symptom:** scanner FOUND/NONE on the alias; real file unread or a homoglyph closes it.
**Cheapest guard:** `realpath(root)` == declared. Names hex-of-hash or exact `re`. Civil day folders are not the scan root.

## 15. Failed look + cached last-exists = still complete

**Failure:** timeout/UNKNOWN, then LKG “files were there.” Scan **healthy**.
**Trigger:** NFS hang; `stat` ERROR mapped to last bool; watchdog LKG (`TASK-08-WATCHDOG-PATTERN.md`).
**Symptom:** greens; files gone or replaced; `errors=0` (`SILENT-FAILURE-DETECTION.md`).
**Cheapest guard:** failed `stat`/`open` ⇒ ERROR, not FOUND/NONE. Do not reuse last exists. Canary-hit missing ⇒ ERROR.

## 16. Scanner marker (`scan.ok` / `last_ok`) exists = look completed

**Failure:** freshness or presence of the **instrument** is the result (`LEGACY-13.md`).
**Trigger:** wrapper `touch last_ok` then scan; cron start-touch (`ADVERSARIAL-CRON-10M.md` §2); self-as-sample heartbeat file.
**Symptom:** `last_successful_scan` within interval; look never finished or returned `[]` on error.
**Cheapest guard:** marker only after a typed FOUND/NONE with both controls. Start-touch is not a scan (`LEGACY-04.md`).

## 17. TOCTOU: exists, then replace (or hash skipped because exists)

**Failure:** decision used a name that is no longer those bytes — or never hashed.
**Trigger:** `if exists: return COMPLETED` without open+hash; writer replace between `stat` and skip; two seats (`LEGACY-01.md`).
**Symptom:** COMPLETED on the pre-replace stub; new bytes unverified or a second lead.
**Cheapest guard:** open, `fstat`, hash the fd, compare `expected`. No `exists` short-circuit. One lock on the path.

## 18. Directory, FIFO, or device named as the artifact

**Failure:** `exists` true, `is_file` false — mapped to FOUND or to “present so skip.”
**Trigger:** `mkdir out/unit`; leftover socket; Windows reserved name that “exists.”
**Symptom:** scanner complete; no regular-file bytes; hash not run (or hashes a directory error mapped to 0).
**Cheapest guard:** not a regular file ⇒ ERROR, not FOUND. Do not skip the job.

## 19. Archive exists, hot exists, restore pointer: three “founds”

**Failure:** any one presence is **the** completion. Consumer and scanner disagree which.
**Trigger:** copy without unlink; symlink hot→archive; restore after archive (`PROVENANCE-ARCHIVE.md`).
**Symptom:** FOUND on stub and on archive; or FOUND on pointer with digest of neither.
**Cheapest guard:** hash the **declared hot** path only. Archive is evidence, not a hit. Pointer ⇒ ERROR.

## 20. NFS / client cache: `stat` says exists (or not) vs server

**Failure:** scanner’s bool is **this** cache. Peer sees the opposite.
**Trigger:** close-to-open not held; attribute cache; stale handle after unlink.
**Symptom:** A FOUND, B NONE, both sure; ACK/skip on a ghost; or miss a real file.
**Cheapest guard:** open+read+hash after fsync-visible publish. Cache-only `stat` is not FOUND. Disagreement with canary ⇒ ERROR.

## 21. Partial listing: first exists closes the scan

**Failure:** one hit (or one `out/` name) ⇒ bag **done**.
**Trigger:** `break` on first exists; `max_hits=1`; newest-first (`LEGACY-35.md`).
**Symptom:** tail unexamined; canary never reached; `unmatched=0`.
**Cheapest guard:** completed look over the declared bag + both controls. First exists is not NONE/FOUND for the rest.

## 22. `I` / monitor decremented on exists

**Failure:** quota **spent**; dashboard **no errors** because a name was present.
**Trigger:** billing on `path.is_file()`; `M` counts missing-file ERROR only (`BUDGET-AWARE-ROUTING.md`).
**Symptom:** included gone; bytes ≠ `expected`; greens.
**Cheapest guard:** decrement `I` only after this process’s Sig1∧Sig2. Alert on stale scan, not on `exists` count.

## 23. Dual predicate: exists-scanner wins over verify

**Failure:** two tools; the exists one is on the run path and reports **safe**. Verify stays UNVERIFIED and is ignored (`LEGACY-15.md`).
**Trigger:** README says hash; cron still `test -f`; `C` local gate (`DEPENDENCY-INVERSION.md`).
**Symptom:** split-brain status; change-detector skips the hasher (`LEGACY-09.md`).
**Cheapest guard:** tombstone `exists` as completion. One published predicate: A∧B. Dual-run until the exists branch is dead.

## 24. Snapshot / backup: restored names exist again (or still)

**Failure:** rolled-back stubs are **already closed**.
**Trigger:** AMI restore; `git checkout` of `out/`; Time Machine.
**Symptom:** scanner FOUND; live work not re-run; `open_units` 0.
**Cheapest guard:** restore is not COMPLETED. Re-MEASURE A∧B. Exists after rollback without Sig2 ⇒ UNVERIFIED.

## 25. Negative control missing: prefix exists matches everything

**Failure:** every queried name **FOUND** because a file exists somewhere the prefix hits.
**Trigger:** no `canary-miss` row; miss is a random string not in store (`POSITIVE-CONTROL.md`).
**Symptom:** scanner complete; matcher too wide; real `re` never tested.
**Cheapest guard:** canary-miss must be a real stored row that prefix would steal. Miss in hits ⇒ ERROR, not FOUND.

## 26. `exists` after unlink race: name gone mid-scan ⇒ NONE for the unit

**Failure:** scanner **finished** with NONE/skip. File was a candidate that vanished (quarantine, other ACK) or never durable.
**Trigger:** AV delete; mailbox unlink (`ADVERSARIAL-MAILBOX-QUEUE.md` §1); scan without canary.
**Symptom:** unmatched=0; work lost; “scan succeeded.”
**Cheapest guard:** NONE only after completed look + both controls. Mid-scan vanish of a declared path ⇒ ERROR/`ORPHAN`, not NONE. Absence is not completeness without MEASURE.
