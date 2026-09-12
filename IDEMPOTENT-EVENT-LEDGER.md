# idempotent event ledger

JSONL, UTF-8, `sort_keys`, tight separators, one trailing newline (`TASK-15-IDEMPOTENT-RECEIPT.md`). Readers see `[0, hwm)` (`TASK-11-IDEMPOTENCE.md`). **Fail if visibility is file length:** torn tail replays twice or becomes `[]`.

**Invariant:** queue state is a pure function of the flushed prefix. **Fail if an in-memory queue is the source of truth:** reboot drops enqueues or double-pops. Do not append onto the unit artifact (`LEGACY-21.md` — **fail:** digest moves).

## Write path

`append(obj) → fsync(bytes) → hwm = end`. Never hwm-first. **Fail if release is logged then delivery fsynced:** crash in the gap = `released` without effect (`TASK-31-GRACEFUL-SHUTDOWN.md`).

Identity of a line: `event_id = sha256(canonical(obj minus at_utc))`. Retry of the same `(type, enqueue_id)` is byte-identical except `at_utc`. **Fail if `event_id` includes wall-clock:** every retry is a new fact. **Fail if `event_id` is a random UUID:** timeout + resend duplicates (`TASK-27-RETRY-SEMANTICS.md`).

Missing required key → do not append; raise. **Fail if you append a partial object:** replay cannot tell truncated from valid (`LEGACY-26.md`).

## Schemas (`v >= 1`; extra keys ignored — `TASK-33-SCHEMA-EVOLUTION.md`)

Common: `v`, `type`, `enqueue_id`, `unit`, `at_utc` (`YYYY-MM-DDTHH:MM:SSZ` only — **fail if local civil time:** day-skew reorder, `TASK-19-TIMESTAMP-HAZARD.md`). `enqueue_id` is stable across retry; `unit` is the handoff `re`. **Fail if identity is `name[:8]`:** UTC rollover drops the row (`TASK-07-DATE-ROLLOVER.md`).

**`enqueued`:** `priority` (0–3), `scheduled_utc`, `seat`, `path`, `expected_sha256`. **Fail if priority is filename-only:** replay and `ls` disagree (`LEGACY-35.md`, `LEGACY-36.md`). **Fail if `expected_sha256` omitted:** `exists` becomes done (`LEGACY-05.md`).

**`promoted`:** `seat`, `pid`, `start` (OS start, UTC seconds). Means: lock published for this `enqueue_id`. **Fail if you promote without a live claim:** ledger says in-flight, OS has no holder (`LEGACY-08.md`). **Fail if `start` missing:** empty-gap, cannot prove stale (`LEGACY-14.md`).

**`released`:** `pid`, `start` (same incarnation), `delivery` object:
- `kind`: `verify_completed` | `stale_archived`
- `evidence_sha256`: declared digest (verify) or archive copy digest (`STALE-CLAIM-RECOVERY.md`)

No `delivery` → refuse the write. **Fail if `kind=sent` or HTTP 200:** self-report, not a check (`TASK-17-SEPARATION-OF-DUTIES.md`).

## Replay (deterministic)

```
state = {queued: {}, inflight: {}, done: {}}   # keyed by enqueue_id
seen = set()
for line in bytes[0:hwm].splitlines():
    if line empty: continue
    if JSON fails: UNKNOWN; stop          # fail if skip: hole looks like never enqueued
    ev = parse(line)                      # require keys; ignore unknown
    eid = event_id(ev)
    if eid in seen: continue              # fail if apply twice: two inflight slots
    seen.add(eid)
    apply(state, ev)                      # file order only
return state
```

`apply`:
- `enqueued`: if `enqueue_id` in done or inflight: no-op; else `queued[id]=ev`. **Fail if you re-queue a done id:** duplicate shell.
- `promoted`: if id not in queued: UNKNOWN (log hole) — **fail if you invent queued:** ghost claim. Else move to `inflight` with `{pid,start}`.
- `released`: if `delivery` missing or `evidence_sha256` empty: UNKNOWN. If id not in inflight: no-op only when already `done` (dup); else UNKNOWN. Else drop inflight, `done[id]=ev`.

Do not sort by `at_utc`. **Fail if you do:** clock skew applies `released` before `promoted` on disk-correct order.

Failed read → UNKNOWN, not empty (`LEGACY-03.md`). **Fail if `[]`:** dispatcher re-launches all. Replay must still surface `canary-hit` (`LEGACY-33.md`). **Fail if wrap evicts it and replay says idle.**

## `released` after confirmed delivery only

Delivery = two-factor `verify` COMPLETED, or archive-copy verify (`LEGACY-32.md`, `STALE-CLAIM-RECOVERY.md`). Then append `released`, flush, hwm.

**Fail if `released` precedes that:** replay marks `done`; `on_launch` skips; worker never ran or died pre-verify (`LEGACY-31.md`). New `enqueue_id` after that skip = second effect. Same `enqueue_id` without `released` = idempotent promote.

Promote is not delivery. **Fail if you release on `promoted`:** crash after pop, before lock rename, looks delivered (`LEGACY-30.md`).

**Rule:** Log what already happened. Replay the prefix. `released` cites evidence that exists.
