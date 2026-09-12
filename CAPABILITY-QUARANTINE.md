# Capability quarantine

**Status:** design only.  
**Depends on:** `PROVENANCE-ARCHIVE.md`, `REMOVAL-FIX-EVIDENCE.md`, `LEGACY-16.md`, `OUTWARD-JOB-AUDIT.md`.  
**Does not exist in this tree:** live capability, stub module, quarantine store.

A **dangerous capability** `X` (hook, binary, import, register) must leave the run path **without** being deleted. Delete is `never_present` or unrecoverable (`REMOVAL-FIX-EVIDENCE.md`). Quarantine is: original bytes in a store a caller cannot exec, stub on the old name, restore that a third party can prove **byte-for-byte**.

This is not retire. Retire names a **different** successor (`LEGACY-15.md`). Quarantine’s successor on the path is a **stub**. The original remains the restore target.

## Declare before mutate

Issuer, before any rename or disable:

```
quarantine = {
  symbol,                 # argv[0] / import / register_id
  hot_path,               # lexical, under root (TASK-32)
  before_digest,          # sha256 of live X now; ≠ sha256(b"")
  archive_path,           # outside any hot glob
  stub_digest,            # sha256 of the stub that will replace X; ≠ before_digest
  tombstone_class         # CapabilityQuarantined
}
```

`before_digest` is hashed from **this** process’s read of `hot_path` (`TASK-06-ARTIFACT-CHECK.md`). Do not copy a worker self-hash (`LEGACY-17.md`). `stub_digest` is declared **before** the stub is written to hot.

## Rename-or-disable protocol

One lock on `symbol` (`TASK-16`). Order is the archive order (`PROVENANCE-ARCHIVE.md`, `ADVERSARIAL-ARCHIVE-MANIFEST.md`): **copy → fsync → re-hash → then** move hot. Unlink-first is ERROR, not quarantined.

```
on_quarantine(X):
    if lock_read UNKNOWN: stop
    d0 = sha256(hot bytes)                 # completed look
    if d0 != before_digest: abort          # hot moved; do not archive a stranger
    copy regular file → archive/payload    # refuse symlink/hardlink; distinct inode
    fsync payload; write manifest; fsync
    if sha256(archive payload) != d0: abort
    rename hot → hot.quarantine-tmp        # still not the stub
    write stub at hot_path; fsync
    if sha256(hot) != stub_digest: abort; do not unlink tmp
    unlink tmp only if it still hashes to d0
    disable every outward register whose resolved digest was d0
    append type=quarantined {symbol, before_digest, stub_digest, archive_path, pid, start}
```

**Rename** (preferred): the original name on the run path is now the stub. Archived bytes live off PATH / off import / outside the glob `C` walks.

**Disable** (registers): `systemctl disable`, crontab comment, chmod −x is **not** enough alone (`SCHEDULER-POLICY-ENFORCE.md`). A second register can still exec `d0`. Disable is **in addition** to rename: every enumerator that listed `d0` must show DISABLED or a path whose digest is `stub_digest` (`OUTWARD-JOB-AUDIT.md`). Disable-without-rename leaves `d0` BOUND on disk.

Do not `chmod −x` the archive copy as the only hold. Restore would need those bits; the hold is **location and register**, not mode bits a later umask can flip.

Crash mid-protocol: tmp + archive + no stub → recover by finishing stub write if archive verifies; never delete archive. Stub on hot and no archive → **broken quarantine**, not “X is gone.”

## The stub

Same `symbol` / `hot_path`. Different bytes.

| Property | Hold |
|---|---|
| Call / import / exec | Raises `CapabilityQuarantined` (or the declared `tombstone_class`) |
| Exit | Non-zero. Not `[]`, not `True`, not 0 (`LEGACY-16.md`) |
| Body | Must not contain `d0` bytes or a decoder of them |
| Digest | `sha256(stub) == stub_digest ≠ before_digest` |
| Registers | Cron that still names the symbol goes UNKNOWN, not green |

A stub that returns “disabled, ok” is **removed and broken**. Callers treat 0 as success and skip the successor gate.

Canary-miss: planted exec of `symbol` must raise. Canary-hit for the **estate**: whatever gate still admits work (not `X`) still accepts a hit and rejects a miss (`ACCEPTANCE-RATE.md`). Quarantine of the only rejector is a broken estate (`REMOVAL-FIX-EVIDENCE.md` `successor_ok`).

## Proof the original is recoverable byte-for-byte

`E` is the archive payload. Restore is a **copy from `E` only**, never from the stub, never from `hot.quarantine-tmp` after unlink, never from a worker-supplied blob.

```
recoverable(X)  iff
    verify(archive_path, before_digest) == COMPLETED     # this process hashed E
    AND ledger quarantined row cites that digest
    AND sha256(hot) == stub_digest                       # live is not X
    AND E is a regular file, st_nlink==1, inode ≠ hot
```

```
on_restore(X):
    if not recoverable(X): stop
    copy E → hot.tmp; fsync
    if sha256(hot.tmp) != before_digest: abort           # not byte-identical
    rename onto hot_path
    if sha256(hot) != before_digest: ERROR               # torn replace
    append type=restored {before_digest, pid, start}
    re-enable registers only via a new grant (LEGACY-22), not because restore succeeded
```

**Byte-for-byte** is `sha256(restored) == before_digest == sha256(E)` on completed looks, same canonical bytes, no appended note (`LEGACY-21.md`). Size, mtime, `exists`, or “it runs” are not the proof. A restore whose digest equals `stub_digest` is the stub put back — fail.

| Observation | Not recoverable |
|---|---|
| Archive missing / empty / digest ≠ `before_digest` | Bytes lost or swapped |
| Archive is symlink to hot (stub) | One inode; restore is the stub |
| Only manifest `sha256` field, payload not re-hashed | Gate-authored evidence |
| `before_digest` taken from the worker after quarantine | Self-hash |
| Disable-only; `d0` still at `hot_path` | Nothing to restore; X never left |

`recoverable` can hold while X stays quarantined. That is the point: proof of restore **capability** is not restore **execution**. Do not exec `E` to prove the hash.

## Verdicts

| `recoverable` | stub raise | registers vs `d0` | Estate |
|---|---|---|---|
| yes | yes | no BOUND `d0` | **quarantined and restorable** |
| yes | `[]` / 0 | — | stub broken; X may still be off-path |
| no | yes | no `d0` | **removed, not quarantined** (`REMOVAL-FIX-EVIDENCE.md`) |
| `d0` still on hot | — | BOUND | **not quarantined** |

## What this must not do

- `rm` X and keep a comment.
- Unlink hot before archive verify.
- Leave `d0` executable beside a stub.
- Restore from the stub or from a new “fixed” rewrite.
- Treat disable bits or a README as the archive.
- Re-enable every register automatically on restore.

**Rule:** Copy-hash-then-rename `X` into an off-path archive; put a raising stub (`stub_digest ≠ before_digest`) on the old name; disable registers that pointed at `d0`. Recoverable iff this process hashes the archive to `before_digest`. Restore is copy-from-archive only, proved by that same digest on hot.
