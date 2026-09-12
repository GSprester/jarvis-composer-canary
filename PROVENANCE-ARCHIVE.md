# provenance archive

Consumer `C` gates on **hot-path presence** (`exists(path)`), not digest (`LEGACY-05.md`). Leftover evidence blocks relaunch. **Do not delete** (`STALE-CLAIM-RECOVERY.md` — **fail:** next find is `never_started`). **Do not rewrite in place** (`LEGACY-21.md` — **fail:** citations orphan). Copy, verify, unlink hot.

`C` globs **only** the declared hot path. **Fail if `C` also globs `archive/`:** deadlock remains. **Fail if archive is a symlink/hardlink to hot:** `C` still sees the inode.

## Layout

Declared hot path stays the WO path (e.g. `out/<unit>`, `.receipt/<unit>`). Archive is **outside** that glob:

```
archive/<unit>/<start>-<sha256>/
  payload            # exact bytes, no appended note
  manifest.json      # schema below
```

`<start>` = dead claim OS start, or `0` if never-started. `<sha256>` = digest of **payload**. **Fail if the dir is `…/latest`:** second archive clobbers. **Fail if payload sits on a path `C` still walks.** **Fail if `manifest` is appended onto payload.** Month JSONL indexes `unit,sha256,relpath` (`TASK-22-ROLLING-WINDOW-AUDIT.md`). **Fail if the index is the only copy:** wrap drops the pointer.

Lexical validate `unit` before join (`TASK-32-PATH-SAFETY.md` — **fail:** `unit=../out` writes back onto hot).

## Manifest

Canonical JSON, `sort_keys`, one trailing newline (`TASK-15-IDEMPOTENT-RECEIPT.md`). `v>=1`; extra keys ignored.

| Key | Type | Meaning |
|---|---|---|
| `v` | number | 1 |
| `type` | string | `provenance_archive` |
| `unit` | string | WO `re`, not `name[:8]` |
| `hot_path` | string | declared relative path |
| `sha256` | string | digest of `payload` |
| `bytes` | number | length |
| `reason` | string | `stale_claim` \| `presence_gate` \| `unmatched` |
| `dead_pid` | number \| null | prior claim |
| `dead_start` | number \| null | prior start |
| `archiver_pid` | number | this incarnation |
| `archiver_start` | number | OS start |
| `at_utc` | string | `YYYY-MM-DDTHH:MM:SSZ` |
| `predecessor` | string \| null | prior archive sha256 for this unit |

**Fail if `sha256` is hashed after adding a footer.** **Fail if `reason` is prose only:** replay cannot filter. `at_utc` is not identity (`TASK-19-TIMESTAMP-HAZARD.md`).

## Reversibility

Restore: archive payload → hot tmp → fsync → digest==`manifest.sha256` → rename. Only if hot is **absent** or already that digest.

**Fail if restore overwrites other hot bytes:** live attempt lost. **Fail if restore is automatic after archive:** `C` blocks again. Restore is audit or two-factor `verify` of real completion, not “put the stub back.” **Fail if reversibility is edit-manifest.** **Fail if restore is a pointer from `hot_path` to `archive/`:** presence still fires.

## Concurrency

One MUTATE (`GATE-DEADLOCK.md`). Steps:

1. `same_process` / lock: need live claim on `unit` **or** proven stale/absent (`LIVENESS-PREDICATE.md`). **Fail if steal on UNKNOWN.**
2. Hash hot bytes `d`. Copy to `archive/…/d/payload.tmp`, fsync, rename `payload`.
3. Write `manifest.json.tmp` with `sha256=d`, fsync, rename.
4. Re-hash archive payload; must equal `d` **and** re-hash of hot (hot must not have changed). **Fail if you unlink first:** crash = delete. **Fail if hot changed mid-copy:** two writers; abort, do not unlink (`LEGACY-01.md`).
5. Unlink **only** the hot inode you hashed (CAS: size+digest, or rename hot aside then unlink aside).
6. Append index JSONL, fsync, hwm (`IDEMPOTENT-EVENT-LEDGER.md`). **Fail if index write is skipped:** restore has no pointer.

Two archivists: EEXIST + same digest → done, still CAS-unlink hot. **Fail if EEXIST ⇒ skip unlink:** `C` stays blocked. **Fail if EEXIST and digest differs:** refuse.

Do not `touch` an empty hot file after unlink. **Fail:** `C` gates on presence again.

**Rule:** Copy, verify, then drop hot. Archive is evidence. Hot presence is the gate. They must not be the same path.
