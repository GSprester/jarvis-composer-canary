# LEGACY-11

Handoff **packet** = durable bytes + a **display name**. Identity and policy live in the **body**. The name may *echo* priority, UTC date, and topic for humans and `ls`. If priority exists **only** in the filename, every consumer that does not parse the name drops or reorders work.

## Filename (display)

```
p{0-3}-{utc_compact}-{topic}-{unit}
```

| Part | Rule |
|---|---|
| `p{n}` | `p0` highest … `p3` lowest. Digit, not words. |
| `utc_compact` | `YYYYMMDDTHHMMSSZ` only — timezone-aware UTC (`TASK-19-TIMESTAMP-HAZARD.md`). Never local civil time. |
| `topic` | `[a-z0-9]+`, no separators that look like dates. |
| `unit` | Opaque id (same as `re` / lock unit). **Not** derived from local date. |

Example: `p1-20260911T063000Z-helix-alpha` for a packet whose body `unit` is `alpha` (scheduled 23:30 PDT 2026-09-10 = 06:30Z the 11th).

Lexical path safety still applies (`TASK-32-PATH-SAFETY.md`). The name is not a path traversal and not a Windows reserved device.

## Body (source of truth)

Canonical JSON (`TASK-15-IDEMPOTENT-RECEIPT.md`, `TASK-33-SCHEMA-EVOLUTION.md`): required `unit`, `priority` (0–3), `topic`, `scheduled_utc`. Readers ignore unknown fields. Writers never omit `priority` once shipped. Matcher and lock use `unit` / `re`, not `name[:8]` (`TASK-04-MATCHER-TESTS.md`).

## What breaks if priority is filename-only

1. **Content-hash dedupe** (`TASK-26-DEDUPE-BY-CONTENT.md`) hashes the body minus `id`. Two names `p0-…-alpha` and `p3-…-alpha` with the same body **collapse**. The high-priority packet disappears; the queue looks like one unit (`TASK-18-COUNT-RECONCILIATION.md`).

2. **Prefix / date matchers** (`TASK-07-DATE-ROLLOVER.md`) join on `YYYYMMDD`. Priority is left of the date or ignored. A `p0` packet on the next UTC day never matches last night’s handoff; a `p3` sibling on the same date sorts first if someone sorts the whole string wrong (`p10` vs `p2` if you ever leave the single digit).

3. **Rename, rotate, archive.** Rotator and monthly evict (`TASK-30-LOG-ROTATION.md`, `TASK-22-ROLLING-WINDOW-AUDIT.md`) keep **bytes**. The filename is gone or generation-replaced. A reader of the archive has no priority → FIFO or skip. Change-detector that hashes names then skips the job deadlocks (`LEGACY-09.md`).

4. **Windows equality / copy.** Case fold or `/` vs `\` (`LEGACY-02.md`) can make `P0-…` and `p0-…` two writers or one. A copy into another repo drops the name; failover keeps `unit` and loses the only priority signal (`TASK-13-FAILOVER-CEILING.md`).

5. **Lock and admit.** One-lead lock is `.lock/writer-<unit>` (`LEGACY-08.md`). Filename priority is not on the lock. A `p3` steal and a `p0` wait look the same. Seat ceiling is a body/admission field, not a prefix (`TASK-23-POLICY-INHERITANCE.md`).

**Rule:** echo priority in the name if you want; **store it in the packet**. Sort and admit from the body. The name is display, like local time.
