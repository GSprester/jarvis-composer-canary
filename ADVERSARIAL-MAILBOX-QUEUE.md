# adversarial review: file mailbox as a queue between two agents

Mechanism: a shared directory (inbox / `tmp`→`new` / `processing` / `done` / outbox). Agent A drops a file; B lists, claims, works, ACKs (unlink, rename, or reply). Success is typically **write returned**, **name gone**, **`.done` exists**, or **`ls` empty**. Confident wrong = **success, completeness, or safety** that does not hold.

## 1. ACK = unlink / `done/` / chat OK without A∧B

**Failure:** absence or a reply file is **complete**. Bytes never verified or never processed.
**Trigger:** `rm` after read; `mv inbox done`; `P` types ACK (`BURST-REVIEW.md`); model writes `.ok`; `exists(done/id)` (`LEGACY-05.md`).
**Symptom:** inbox empty; both agents stop; payload UNVERIFIED or gone; `R` uncomputable if deleted (`BLAST-RADIUS-ESTIMATE.md`).
**Cheapest guard:** ACK is checker Sig2 after this process hashed the declared path (`DUAL-SIGNATURE-COMPLETION.md`). Unlink only after archive+verify (`PROVENANCE-ARCHIVE.md`). `P` ACK is `model_guess` (`EVIDENCE-TIERING.md`).

## 2. “Sent” on create/rename before durable peer-visible bytes

**Failure:** A reports **delivered**. File missing, torn, or zeros on B’s mount.
**Trigger:** write+rename, no fsync; NFS close-to-open delay; `Popen` drop without wait; ENOSPC after create.
**Symptom:** A complete; B idle or parses junk as NONE; retry mints a second drop (`TASK-27-RETRY-SEMANTICS.md`).
**Cheapest guard:** write tmp → fsync file+dir → rename into `new/`. A’s success = B can hash those bytes (or typed `UNCONFIRMED`, not delivered).

## 3. Empty inbox after a failed look = caught up

**Failure:** `ls` ERROR / timeout / unreadable → `[]` → **no work**. Completeness of the interval.
**Trigger:** hung NFS; `EACCES`; scanner stdout empty (`POSITIVE-CONTROL.md`); autofs empty overlay.
**Symptom:** greens; files sit in a tree A cannot see; “no errors” (`SILENT-FAILURE-DETECTION.md`).
**Cheapest guard:** look ERROR ⇒ not idle. Canary-hit missing in the mailbox ⇒ ERROR. Empty is NONE only after a completed MEASURE.

## 4. Presence of outbox / `.done` / reply as COMPLETED

**Failure:** a leftover or self-authored name is **closed**.
**Trigger:** prior run’s `.done`; `touch id.ok`; `W` `status=completed` as a file (`TASK-10-SELF-REPORT.md`); `C` presence-gates (`PROVENANCE-ARCHIVE.md`).
**Symptom:** B skips; A never retries; stub digest ≠ `expected` (`LEGACY-05.md`).
**Cheapest guard:** reply is a candidate. Close iff `sha256(payload)==expected` ∧ `re==unit` ∧ checker receipt. Archive leftovers; do not skip on exists.

## 5. Read while write (no `tmp`→`new`)

**Failure:** B consumes a prefix or 0-byte create. Treats it as a full message: **NONE**, **OK**, or forged COMPLETED.
**Trigger:** A writes in place; inotify on `CREATE`; editor `w`; `open`+`write` without rename.
**Symptom:** torn JSON (`LEGACY-26.md`); empty payload “handled”; A still writing after B ACKs (§1).
**Cheapest guard:** B reads only `new/` after rename. No newline / bad JSON / size 0 ⇒ ERROR, not ACK. Never watch the write path.

## 6. Same filename overwrite: first drop “sent,” bytes gone

**Failure:** second create replaces the inode. A1 **delivered**. Queue **complete** for that name.
**Trigger:** identity is `unit.json` / `YYYYMMDD-…` (`TASK-07-DATE-ROLLOVER.md`); two covers one path (`SUBSTITUTE-REVIEW-QUEUE.md`).
**Symptom:** one body survives; the other never launches; dedup-by-name no-ops the retry (`ORPHAN-RECLAMATION.md`).
**Cheapest guard:** drop name = `sha256(canonical body)` or `enqueue_id`. `O_EXCL` / no clobber. Distinct bodies, distinct names (`TASK-26-DEDUPE-BY-CONTENT.md`).

## 7. Two claimers: non-atomic take, each **safe**

**Failure:** two leads, two ACKs, two “only I hold this.”
**Trigger:** `ls`+`rename` race; copy-then-unlink (cross-device); NFS rename not exclusive; Windows non-atomic move.
**Symptom:** split-brain writes (`LEGACY-01.md`); two `done/` rows; torn payload.
**Cheapest guard:** claim = `rename` of a unique name into a lock dir **or** `link`+`O_EXCL` that fails for the loser (loser ≠ 0, not “already handled”). One published `{pid,start}` (`LIVENESS-PREDICATE.md`).

## 8. Leftover claim / `processing/` = in-flight **safe**

**Failure:** tick **safely skipped**. Holder dead, recycled, or never started.
**Trigger:** crash after rename; pid-only liveness (`ADVERSARIAL-PID-ONLY-LIVENESS.md`); mtime “fresh” (`LEGACY-14.md`).
**Symptom:** mailbox not empty but “owned”; A thinks B is working; days of greens.
**Cheapest guard:** skip only if `same_process`. Else stale-archive the claim file and re-offer, or ERROR. mtime is not a process.

## 9. Producer assumes taken: timeout delete = delivered

**Failure:** A’s safety of “B has it.” File unlinked; B never claimed.
**Trigger:** “if still there after 10m, they must have a copy”; drain `rm inbox/*`; antivirus quarantine (name vanishes).
**Symptom:** no drop, no claim, no Sig1; retry sees missing = complete.
**Cheapest guard:** A deletes only after Sig1∧Sig2 or a **recomputed** archive. Name gone without that ⇒ ERROR/`ORPHAN`, not delivered.

## 10. Identity from filename, date, or thread id

**Failure:** B closes the wrong WO; today’s bag **complete**.
**Trigger:** `name[:8]`; chat/mailbox subject (`SUBSTITUTE-REVIEW-QUEUE.md`); `CRON_TZ` in the name (`TASK-19-TIMESTAMP-HAZARD.md`).
**Symptom:** 74 quoted names match; 494 unquoted “done” (`RECEIPT-MATCHING.md`); UTC rollover collision.
**Cheapest guard:** `unit` is `re` inside the body. Filename is display. Canary-miss FOUND ⇒ matcher is the defect.

## 11. Mailbox is `/tmp`, chat, or model context

**Failure:** after days offline, `ls` empty = **nothing to review**.
**Trigger:** laptop drop; Cursor thread; `P`’s private inbox (`SUBSTITUTE-REVIEW-QUEUE.md`).
**Symptom:** covers invisible; `P` ACKs silence; live `unit`s untouched.
**Cheapest guard:** queue lives in the estate tree, outside the payload (`LEGACY-21.md`). Failed look ≠ empty. Chat is not a queue.

## 12. inotify / cursor / HWM ahead of the directory

**Failure:** watcher **caught up**. Files exist unread.
**Trigger:** queue overflow; cursor file stores a name never durable; `st_mtime` watermark skips a replace-in-place; lost CREATE.
**Symptom:** B idle; A “sent”; monitor quiet.
**Cheapest guard:** poll + full MEASURE of the dir (canary-hit). Cursor advances only after hash of that name. Overflow ⇒ ERROR, not 0.

## 13. Sync conflict / clone / third consumer

**Failure:** each side’s mailbox **complete**; a stranger ate the drop, or `.sync-conflict` holds the real body.
**Trigger:** Dropbox/rsync; laptop+server; agent C shares the folder; `file (1).json`.
**Symptom:** A and B both ACK; payload on a name neither lists as due; two trees (`ADVERSARIAL-LEDGER-QUEUE.md` §8).
**Cheapest guard:** one lock on the **shared** root `{pid,start,boot_id}`. Conflict/extra names ⇒ ERROR, not done. Foreign `boot_id` is not ACK.

## 14. Loopback: A reads A’s outbox as B’s ACK

**Failure:** shared dir, one namespace. A’s drop or echo is **peer complete**.
**Trigger:** inbox=outbox; `mv` into the same glob B and A both watch; test fixture writes both sides.
**Symptom:** A stops; B never started; `qid` dedup no-ops.
**Cheapest guard:** A’s success is B’s checker receipt, not a name A can create. Subtract self writes (`LEGACY-13.md`).

## 15. Dedup-by-name absorbs a never-read drop

**Failure:** second `offer` **OK/duplicate**. Completeness of submit.
**Trigger:** first file ACKed empty (§1,§5); same `unit.json` retried; `released` analogue is “name was seen.”
**Symptom:** producer stops; work never ran (`ORPHAN-RECLAMATION.md`).
**Cheapest guard:** duplicate only if last ACK has recomputed Sig1∧Sig2. Else `ORPHAN`. Name-seen is not delivery.

## 16. Partial listing treated as the whole bag

**Failure:** first K ACKed; rest unlisted; tick **complete**.
**Trigger:** `readdir` cap; `max_batch=1` then exit 0; newest-first (`LEGACY-35.md`); `break` on first 0.
**Symptom:** tail sits; inbox “quiet” after the K names vanish.
**Cheapest guard:** exit complete only if a finished MEASURE shows no due names **and** canary-hit. Else typed `REMAINING` ≠ 0.

## 17. Path traversal / absolute drop name “delivered”

**Failure:** write **succeeded** outside the mailbox. Estate inbox looks empty/safe.
**Trigger:** `unit=../out`; Windows `\` (`TASK-32-PATH-SAFETY.md`); symlink inbox entry.
**Symptom:** artifacts under the payload or `/`; B sees nothing due.
**Cheapest guard:** lexical validate the name **before** join. Escape ⇒ refuse, not sent.

## 18. Case-fold / NFC: two ids, one inode (or “missing” after write)

**Failure:** collision **absorbed**; or A sent and B’s lookup **NONE**.
**Trigger:** `Unit.json` vs `unit.json` on macOS/Windows; composed vs decomposed Unicode.
**Symptom:** one body; the other “already there” or “never arrived.”
**Cheapest guard:** mailbox names are `[0-9a-f]{64}` hex of the body hash. Reject any other spelling.

## 19. Canary drop processed or ACKed

**Failure:** controls gone; next empty `ls` is **real idle**.
**Trigger:** B’s glob is `*.json` including `canary-hit`; “ACK everything then work.”
**Symptom:** hit missing ⇒ should be ERROR, treated as NONE (`LEGACY-39.md`).
**Cheapest guard:** canaries are not work. Missing hit after a completed look ⇒ mailbox ERROR. Do not unlink controls.

## 20. 0-byte create on `ENOSPC` / quota = sent

**Failure:** A **delivered** an empty candidate. B ACKs exists (§4) or NONE.
**Trigger:** disk full mid-write; user quota; tmpfs inbox.
**Symptom:** stub COMPLETED; `expected` rewritten to empty (`LEGACY-32.md`) if the gate is captured.
**Cheapest guard:** sent iff size>0 ∧ `sha256==declared`. ENOSPC / 0-byte ⇒ ≠ delivered. Never copy empty into `expected`.

## 21. Backup / snapshot restore

**Failure:** restored `done/` or vanished inbox = **already closed** or **nothing due**.
**Trigger:** Time Machine; AMI rollback; `git checkout` of the drop dir.
**Symptom:** live work re-ACKed or wiped; `open_units` 0 (`TASK-34-METRIC-THAT-LIES.md`).
**Cheapest guard:** restore is not ACK. Re-MEASURE A∧B; names without Sig2 stay UNVERIFIED. Do not treat snapshot absence as delivery.

## 22. One reply / batch ACK closes every outstanding name

**Failure:** safety of the **directory**, not of each `enqueue_id`.
**Trigger:** `ACK all`; outbox `ok` with no id; `rm inbox/*` after one success; owner chat “cleared.”
**Symptom:** unopened drops gone; lane leapfrog (`ORDERED-RELEASE-QUEUE.md`).
**Cheapest guard:** one ACK cites one `enqueue_id` + digest just hashed. Directory wipe ⇒ ERROR.

## 23. Model / `W` writes the ACK file

**Failure:** author of the work is author of **closed** (`LEGACY-17.md`).
**Trigger:** seat dumps `DONE`; wrapper copies `status=completed` into outbox (`DEPENDENCY-INVERSION.md`).
**Symptom:** COMPLETED; file empty; next drops skip (`LEGACY-31.md`).
**Cheapest guard:** only the checker process may create ACK names. Wrapper may record disposition, not close.

## 24. `I` / monitor decremented on drop or unlink

**Failure:** quota **spent**; dashboard **no errors** because the mailbox emitted success.
**Trigger:** billing on `rename` to `new/`; `M` counts ERROR files only (`BUDGET-AWARE-ROUTING.md`).
**Symptom:** included gone; work UNVERIFIED; greens (`SILENT-FAILURE-DETECTION.md`).
**Cheapest guard:** decrement `I` only after this process’s Sig1∧Sig2. Alert on stale `last_successful_scan`, not on empty inbox.

## 25. Cross-device / hardlink ACK: name gone, bytes or a twin remain

**Failure:** unlink of one link is **consumed**. Another name still due — or the only copy was the unlinked link and the peer never had bytes.
**Trigger:** `mv` across mounts (copy+unlink); `ln` then `rm`; bind-mount alias.
**Symptom:** A complete; B sees a leftover or nothing; two names one inode processed twice or zero times.
**Cheapest guard:** refuse drop unless `stat(inbox)` device == `stat(tmp)` device. ACK unlinks only after archive digest match. Hardlink count ≠ 1 ⇒ ERROR.

## 26. Clock-step / civil name: `last_ok` or day’s folder looks finished

**Failure:** schedule or partition **healthy**. Drops live under another civil hour.
**Trigger:** `inbox/YYYYMMDD/`; NTP step (`TASK-19-TIMESTAMP-HAZARD.md`); `last_ok` touched at drop time (launch ≠ MEASURE).
**Symptom:** today’s dir empty; yesterday’s full; age(`last_ok`) small.
**Cheapest guard:** one UTC mailbox, no day folder. `last_ok` only after completed MEASURE. `last_ok > now` or gap > interval ⇒ ERROR.
