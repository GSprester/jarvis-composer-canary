# TASK-16-ONE-LEAD-LOCK

Proposal only. This checkout has no multi-seat runtime. The lock is a **repo-level writer lead**: at most one seat may mutate the same unit of work (one handoff, one receipt path, one job row) at a time.

## Shape

One lock object per unit, path derived from the work id, never from the UTC date prefix (`TASK-07-DATE-ROLLOVER.md`):

```
.lock/writer-<handoff_id>
```

A published lock is a **complete identity record**, not an empty exclusive file:

```json
{"pid": 4412, "start": 1757600001.25, "seat": "seat-a", "unit": "20260910-alpha"}
```

`pid` + `start` name a process incarnation. After exec or pid reuse, `start` does not match `/proc/<pid>` (or equivalent). Same pair as `TASK-03-REBOOT-SWEEP.md`: a live claim is a process that still has that start time.

### Publish order (atomic visibility)

1. Write the identity to `writer-<id>.tmp`.
2. Flush / fsync that file (`TASK-11-IDEMPOTENCE.md`: bytes, then flush, then mark).
3. `rename` onto `writer-<id>` (atomic on the same filesystem).

Readers treat **only the renamed path** as held. A tmp file is not a lock. There is no window where “I hold the lock” is visible without identity.

## Stale-lock recovery after a hard kill

A later seat (or reboot sweep) loads the published record and classifies:

| Observation | Action |
|---|---|
| No `writer-<id>` | acquire (write tmp → fsync → rename) |
| Record well-formed AND `pid` live AND `/proc/pid` start == `start` | lock live; do not steal |
| Record well-formed AND (`pid` dead OR start mismatch OR `start` < host `boot_at`) | stale; unlink **only if** the file still hashes to that record, then acquire |
| Record missing, empty, or unparseable | **not live** (fail closed on “held”); treat as interrupted identity, same as missing `process_started_at` in TASK-03; unlink if still empty/unparseable, then acquire |

Hard kill: kernel drops the process; the file remains. Recovery is **identity vs the OS**, not “the file is old.” After host reboot, every published `start < boot_at` is stale (`reboot_interrupted` for the lock). Do not wait for mtime.

Unlink-before-acquire must be compare-and-unlink (inode + digest of the record you judged stale). Otherwise two recoverers steal at once.

## Why mtime is weaker than pid + start

`mtime` answers “when was the inode touched?” It does not name a process.

- A dead holder can have a **fresh** mtime (unrelated `touch`, antivirus, backup, `ls` on some network FS).
- A live holder can have a **stale** mtime (acquired, then blocked in a long probe; no further writes).
- Clock step, NTP, and laptop sleep move mtime relative to “now” without a crash.
- After reboot, mtime can still look recent; pid+start vs `boot_at` cannot.

A timeout on mtime either steals from a live silent holder or never recovers a dead one whose mtime was refreshed. `pid` + `start` is checkable by a third party (`TASK-10-SELF-REPORT.md`): either that incarnation exists or it does not.

## Exact failure: killed between acquire and writing identity

If acquire is “create empty `writer-<id>` with `O_EXCL`” and identity is a **later** write:

```
t0  seat-A  O_EXCL create  writer-20260910-alpha   # empty file now “held”
t1  seat-A  hard-killed
t2  seat-A  never writes {pid, start}
```

What others see: the lock **exists** and has **no incarnation**.

- **Wait-if-exists:** seat-B blocks forever. Hard kill became a deadlock. Fail-open for liveness.
- **Steal-if-empty-immediately:** seat-B unlinks and takes the lock while seat-A is **still alive** at t0→t1 (slow write, not yet killed). Seat-A then writes its identity over seat-B, or continues mutating the unit. Two leads.
- **Steal-if-mtime-old:** same as mtime weakness; empty file’s mtime is t0, which may be 2 ms ago. You either wait forever or steal from a live acquirer.

That gap is split-brain **or** deadlock, depending on the recoverer’s guess. There is no pid to probe and no start to compare to `boot_at`.

The rename-of-a-complete-record design removes the gap: an empty exclusive create is never the published lock. Death before rename leaves only a tmp (ignored). Death after rename leaves a probeable `pid`+`start`.
