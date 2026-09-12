# adversarial review: a manifest recording an archive operation

Mechanism: `manifest.json` (plus optional index JSONL) asserts that hot bytes were **copied, verified, and unlinked** (`PROVENANCE-ARCHIVE.md`). Success is typically **manifest exists**, **`sha256` field set**, **`stale_archived`**, or **index row**. Confident wrong = **success, completeness, or safety** that does not hold.

## 1. Manifest written; archive payload never re-hashed

**Failure:** a row says **archived**. Bytes missing, empty, symlink, or ≠ `d` (`STALE-CLAIM-RECOVERY.md`).
**Trigger:** hash hot only; copy `W`’s digest; write manifest then skip payload; `kind=stale_archived` on exists (`ADVERSARIAL-LEDGER-QUEUE.md` §7).
**Symptom:** recovery “complete”; next find `[]` = never-tried (`LEGACY-03.md`); `R` uncomputable.
**Cheapest guard:** `stale_archived` / success only if this process just hashed **archive payload** and `d` equals hot hash from **before** copy. Manifest field is not the digest.

## 2. Unlink first (or delete-only) then manifest

**Failure:** hot gone; row **safe**. Crash mid-copy = delete. Same as never archived.
**Trigger:** `rm` then write row; “save time”; ENOSPC after unlink.
**Symptom:** `interrupted` with no payload; crash-loop ≡ first-run; late writer re-blocks (`LEGACY-05.md`).
**Cheapest guard:** copy → fsync payload → fsync manifest → re-hash both → CAS unlink only that inode. Unlink-first ⇒ ERROR, not archived.

## 3. `sha256` copied from `W` / `expected` / hot after a footer

**Failure:** gate authored the evidence (`LEGACY-17.md`). Manifest **matches**.
**Trigger:** hash after note (`LEGACY-21.md`); `expected` post-write; empty-hash declared (`LEGACY-32.md`).
**Symptom:** COMPLETED/`stale_archived` on a stub; citations orphan.
**Cheapest guard:** `d = sha256(canonical payload)` only. Manifest must not be appended onto payload. `expected` never read from `W`.

## 4. Hot changed mid-copy; still unlink / still “done”

**Failure:** two writers. Manifest `d` is the old inode; new hot dropped or leftover (`LEGACY-01.md`).
**Trigger:** no CAS; steal on UNKNOWN (`LIVENESS-PREDICATE.md`); second seat “just archives.”
**Symptom:** live attempt deleted; or `C` still blocked on new stub; both sides **archived**.
**Cheapest guard:** re-hash hot after copy; mismatch ⇒ abort, do not unlink. UNKNOWN `same` ⇒ no archive.

## 5. Symlink / hardlink archive → hot (or reverse)

**Failure:** one inode. Manifest **copied**. `C` still sees presence — or unlink deletes both.
**Trigger:** `ln -s`; restore pointer (`PROVENANCE-ARCHIVE.md`); cross-device “copy.”
**Symptom:** archive OK + still blocked; or payload vanishes with hot.
**Cheapest guard:** refuse unless regular file, `st_nlink==1`, distinct device+inode from hot, then hash. Link ⇒ ERROR.

## 6. `…/latest` or same dir clobbers the prior archive

**Failure:** second write **succeeded**. First attempt’s bytes gone; its manifest overwritten.
**Trigger:** `archive/<unit>/latest`; `start=0` for every never-started; case-fold two `unit`s.
**Symptom:** chain looks one-deep; incident review missing; predecessor dangles.
**Cheapest guard:** dir is `<start>-<sha256>`, never `latest`. EEXIST + different digest ⇒ refuse. Same digest ⇒ CAS-unlink only.

## 7. EEXIST ⇒ skip unlink (or different digest ⇒ skip)

**Failure:** archive **already done**. `C` still gates on hot — or the other digest is the real body.
**Trigger:** two archivists; “idempotent”; sync-conflict copy.
**Symptom:** dispatcher relaunches into a leftover; or wrong payload is the cited archive.
**Cheapest guard:** EEXIST + same digest ⇒ still CAS-unlink hot. EEXIST + different digest ⇒ REFUSE, not done.

## 8. `released.kind=stale_archived` on manifest exists

**Failure:** lane head **popped**. Completeness of the queue (`ORDERED-RELEASE-QUEUE.md`).
**Trigger:** `exists(manifest.json)`; index row; `P` ACK (`BURST-REVIEW.md`).
**Symptom:** successors release; consumer never sees payload; `offer` duplicate.
**Cheapest guard:** `stale_archived` only after this process’s payload hash == `evidence_sha256`. Exists is a candidate (`ADVERSARIAL-EXISTS-SCANNER.md`).

## 9. Steal / archive a live holder

**Failure:** safety of **stale**. Live drain archived; two leads or work deleted.
**Trigger:** mtime (`LEGACY-14.md`); pid-only (`ADVERSARIAL-PID-ONLY-LIVENESS.md`); lock UNKNOWN ⇒ stale.
**Symptom:** holder still writing; archive of a prefix; `C` races restore.
**Cheapest guard:** archive only if `same_process` is false after a completed probe, or claim absent. UNKNOWN ⇒ stop.

## 10. `unit` from filename / prefix / `../out`

**Failure:** **that** name archived. This `re` untouched or hot escaped (`TASK-32-PATH-SAFETY.md`).
**Trigger:** `name[:8]` (`RECEIPT-MATCHING.md`); `unit=../out`; Windows `\`.
**Symptom:** manifest OK under the wrong tree; live `open_units` unchanged; payload on the glob `C` walks.
**Cheapest guard:** lexical validate `unit` **before** join. `unit` is `re`. Escape ⇒ not archived.

## 11. Index JSONL is the only copy; wrap / skip index

**Failure:** pointer **complete** or list **empty** = nothing left to archive (`TASK-22-ROLLING-WINDOW-AUDIT.md`).
**Trigger:** index write skipped; rotate without archive-before-evict; HWM ahead of bytes (`ADVERSARIAL-LEDGER-QUEUE.md` §4).
**Symptom:** restore has no path; `list=[]` ⇒ recovery done; payload orphaned on disk.
**Cheapest guard:** payload+manifest are the evidence. Index is a pointer after fsync. Replay without canary-hit ⇒ ERROR. Skip index ⇒ ≠ done.

## 12. Restore by editing the manifest (or auto-restore / pointer)

**Failure:** reversibility **succeeded**. Bytes not at hot, or stub put back so `C` blocks (`PROVENANCE-ARCHIVE.md`).
**Trigger:** rewrite `sha256`; symlink `hot→archive`; restore immediately after archive.
**Symptom:** “restored”; live attempt overwritten; presence deadlock returns.
**Cheapest guard:** restore = copy payload → tmp → fsync → hash==`manifest.sha256` → rename iff hot absent or already `d`. Edit-manifest is not restore. No auto-restore.

## 13. Failed look at `archive/` = nothing archived / review complete

**Failure:** timeout/`[]` ⇒ bag **caught up** (`POSITIVE-CONTROL.md`).
**Trigger:** hung NFS; `EACCES`; `C` also globs `archive/` and deadlocks MEASURE.
**Symptom:** burst “nothing to review”; `R` truncated (`BLAST-RADIUS-ESTIMATE.md`).
**Cheapest guard:** look ERROR ⇒ not empty. Canary-hit missing in archive ⇒ ERROR. Include archive in the window.

## 14. Two archivists, two manifests, each **valid**

**Failure:** split `d`. Each side **safe**. Unlink races (`LEGACY-12.md`).
**Trigger:** substitute + primary; clone; no lock on `archive/<unit>`.
**Symptom:** two dirs; one unlinked the other’s hot; index cites both or neither.
**Cheapest guard:** one MUTATE lock on the unit. Second archivist: EEXIST rules in §7 only.

## 15. Copy without fsync; manifest durable

**Failure:** row fsynced. Payload zeros/torn after crash. Manifest **matches** a future read of junk or misses.
**Trigger:** rename without `fsync` file+dir; NFS; `payload.tmp` lost.
**Symptom:** archived; open of payload ERROR mapped to NONE; or hash of zeros “equals” empty `expected`.
**Cheapest guard:** fsync payload and dir **before** manifest rename. Torn/missing payload ⇒ ERROR, not archived.

## 16. `reason` prose / unknown → row skipped as success

**Failure:** replay cannot filter; skipped line **handled** (`TASK-33-SCHEMA-EVOLUTION.md`).
**Trigger:** `reason="worker died"`; extra keys ignored ⇒ whole object ignored; unknown `type`.
**Symptom:** `stale_claim` bag empty; crash-loop count 0; work still blocked.
**Cheapest guard:** `reason` enum only. Unknown `type`/`reason` ⇒ replay ERROR, not skip.

## 17. `predecessor` / chain looks complete

**Failure:** pointer to a digest that was never a payload, or clobbered (§6). History **intact**.
**Trigger:** `predecessor` copied from `W`; omitted; points at `latest`.
**Symptom:** audit walks a lie; first attempt invisible; `n` crash-loop wrong.
**Cheapest guard:** `predecessor` must be a dir whose payload **this process** hashed. Missing/broken ⇒ ERROR, not a full chain.

## 18. Model / wrapper copies a sentence into `manifest.json`

**Failure:** author of the work is author of **archived** (`TASK-10-SELF-REPORT.md`).
**Trigger:** `W` writes the file; “DONE” footer; `archiver_pid` = worker (`ADVERSARIAL-SELF-SUMMARY.md`).
**Symptom:** COMPLETED/`interrupted` from prose; bytes unverified.
**Cheapest guard:** only the archivist wrapper after MEASURE. `W` cannot create `type=provenance_archive`. `archiver` ≠ worker `pid+start`.

## 19. `bytes` / mtime / exists without hash

**Failure:** size or name **is** the copy (`ADVERSARIAL-EXISTS-SCANNER.md` §9).
**Trigger:** `st_size` match; `touch` manifest; sparse hole (size full, read zeros).
**Symptom:** archived; digest wrong; `touch` after fail looks fresh.
**Cheapest guard:** hash the fd. Size/mtime are not A. `bytes` must equal `len` of the hashed buffer.

## 20. `start=0` / case-fold / UTC day folder: collision **absorbed**

**Failure:** two attempts, one dir. Second **OK**; first gone or “duplicate.”
**Trigger:** never-started always `0`; `Unit` vs `unit`; `archive/YYYYMMDD/` (`TASK-19-TIMESTAMP-HAZARD.md`).
**Symptom:** one payload; `qid` dedup no-ops (`ORPHAN-RECLAMATION.md`).
**Cheapest guard:** identity is `<os_start>-<sha256>` plus `re`. Reject non-hex / civil folders. Collision + different `d` ⇒ refuse.

## 21. `C` globs `archive/`: leftover presence = still done **and** still blocked

**Failure:** consumer and dispatcher disagree. Manifest **archived**; `C` FOUND on archive path.
**Trigger:** glob `**`; restore pointer; payload on a path `C` walks.
**Symptom:** relaunch refused forever (accident brake) **or** `C` treats archive as COMPLETED.
**Cheapest guard:** `C` globs **only** declared hot. Archive outside that glob. FOUND on archive path ⇒ ERROR.

## 22. `touch` empty hot after unlink

**Failure:** archive **finished**. Presence gate returns (`PROVENANCE-ARCHIVE.md`).
**Trigger:** “keep the path”; create so `exists` for the next job; LKG.
**Symptom:** `C` blocked; dispatcher thinks `interrupted`/`open`; two stories.
**Cheapest guard:** do not create hot after unlink. Next write is a new claim’s payload only.

## 23. `archiver_pid` only / recycled incarnation

**Failure:** record names a live stranger as the archivist. Sweep **trusts** the row.
**Trigger:** no `start`; reboot (`ADVERSARIAL-PID-ONLY-LIVENESS.md` §1–2).
**Symptom:** “we archived”; payload never durable; steal under a false author.
**Cheapest guard:** `{archiver_pid, archiver_start}` and `same` to re-run. Pid-only is not authorship.

## 24. Torn / pretty / no-newline manifest parsed as success

**Failure:** prefix keys present (`sha256`, `unit`) ⇒ **valid**. Or last line invisible ⇒ “not archived” then sudden done (`ADVERSARIAL-LEDGER-QUEUE.md` §17).
**Trigger:** crash mid-JSON; editor; CRLF; `json.dumps` default.
**Symptom:** flapping completeness; hash of a truncated object.
**Cheapest guard:** canonical dumps; refuse if no trailing newline or missing required keys. Partial object ⇒ ERROR (`LEGACY-26.md`).

## 25. Cross-device copy+unlink before verify

**Failure:** `mv` across mounts is copy+unlink. Source gone; dest unverified; manifest **sent**.
**Trigger:** archive on another FS; bind-mount alias (`ADVERSARIAL-MAILBOX-QUEUE.md` §25).
**Symptom:** ENOSPC mid-copy; half file; hot already unlinked.
**Cheapest guard:** `stat` same device or copy-verify-then-unlink source. Verify dest hash **before** source unlink.

## 26. `I` / monitor / `open_units` moved on manifest write

**Failure:** quota **spent**; dashboard **archived=N** or `open_units=0` because the row exists.
**Trigger:** billing on rename; hot wrap of OPEN (`TASK-34-METRIC-THAT-LIES.md`); `M` skips archive (`SILENT-FAILURE-DETECTION.md`).
**Symptom:** included gone; work UNVERIFIED; greens; incident evicted from hot.
**Cheapest guard:** decrement `I` only after Sig1∧Sig2. Count archive only after payload hash. Pair `open_units` with archive count + wraps.
