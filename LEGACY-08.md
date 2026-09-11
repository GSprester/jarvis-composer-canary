# LEGACY-08

Lock file format so **“is the holder still alive?”** is answered by **structured fields + the OS**, not by reading a sentence. Free text (`"still running"`, `"seat-a holds this"`) is not a probe (`TASK-16-ONE-LEAD-LOCK.md`, `LEGACY-01.md`).

## Publish

- Path: `.lock/writer-<unit_id>` (unit id, not a local `YYYYMMDD`, `TASK-19-TIMESTAMP-HAZARD.md`).
- Visibility: write `writer-<unit_id>.tmp` → fsync → `rename` onto the final path. An empty exclusive create is **not** a lock.
- Body: one UTF-8 JSON object, `sort_keys`, one trailing newline (`TASK-15-IDEMPOTENT-RECEIPT.md`). No comments, no prose keys.

## Schema (all keys required; extra keys ignored — `TASK-33-SCHEMA-EVOLUTION.md`)

| Key | Type | Meaning |
|---|---|---|
| `v` | number | Format version. Readers require `v >= 1`. |
| `unit` | string | Same as `<unit_id>` in the path. |
| `pid` | number | Holder pid (integer). |
| `start` | number | Process start time, **UTC seconds** (float ok), from the OS at acquire. |
| `seat` | string | Seat id (opaque token, not liveness). |

```json
{"pid":4412,"seat":"seat-a","start":1757600001.25,"unit":"20260910-alpha","v":1}
```

`pid` + `start` name an **incarnation**. After exec or pid reuse, `start` does not match.

## Alive predicate (no text)

```
alive  iff  record well-formed
       AND  pid is live
       AND  OS start(pid) == record.start
       AND  record.start >= host boot_at
```

- Missing/unparseable field → **not alive** (fail closed, `LEGACY-07.md`). Do not parse a `note` string.
- Dead pid, start mismatch, or `start < boot_at` → stale; compare-and-unlink the same bytes, then acquire.
- Do not use mtime.

## Out of spec

`message`, `status`, `hostname` as the liveness signal. A wrapper disposition may exist **elsewhere**; it does not answer this question.
