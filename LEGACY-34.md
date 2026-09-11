# LEGACY-34

Bounded queue (`N = 32`, `TASK-25-QUEUE-BACKPRESSURE.md`) that honours body **priority** (`0` highest … `3` lowest, `LEGACY-11.md`) without letting a stream of `p0` starve an older `p3`. Aging is a **dequeue view**. It does not rewrite the handoff (`LEGACY-21.md`). Filename `p{n}` is display only.

Missing or unknown `priority` / `scheduled_utc` **raises** (`TASK-20-FAIL-LOUD.md`). Empty queue after a failed look is UNKNOWN, not “idle” (`LEGACY-33.md`). `queue_full` is refuse, not COMPLETED. Same `unit` / content hash is one slot (`TASK-26-DEDUPE-BY-CONTENT.md`).

## Slot

`unit`, `priority` (body), `scheduled_utc` (UTC), declared `(path, expected)`, `seat`. Enqueue time is **not** identity; it may break ties only after `scheduled_utc`.

## Effective rank (aging)

```
age_s              = now_utc - scheduled_utc          # never local civil time
effective_priority = max(0, priority - floor(age_s / QUANTUM_S))
```

`QUANTUM_S = 60`. Each minute waiting promotes one class toward `p0`. A `p3` scheduled 180 s ago ties a fresh `p0`. Do not write `effective_priority` onto the packet.

## Dequeue

Among admitted slots, pick the minimum of:

```
(effective_priority, scheduled_utc, unit)
```

Then require the one-lead lock for that unit (`LEGACY-08.md`). If lock steal is not allowed (merely old, `LEGACY-14.md`), skip that slot and pick the next minimum — do not drop it as done.

Within one `effective_priority`, this is **oldest-first**. Across classes, urgency still wins until age catches up.

## Starvation bound

Worst wait for a live slot before it shares the top class:

```
max_wait_s ≤ priority * QUANTUM_S     # p3 ≤ 180 s
```

A test that enqueues one `p3` then an infinite `p0` stream must dequeue the `p3` on or before the first pop after `scheduled_utc + 180s` (`LEGACY-30.md`: if the worker never `admit`s, that is never-tried, not “low priority”). Continuous `p0` with `scheduled_utc` *older* than the `p3` may still go first — that is oldest-first, not starvation.

## Out of spec

Sort by filename. Sort by mtime. Promote by mutating `priority`. Unbounded side channel when `p0` is waiting (`LEGACY-01.md`). Treat wrap-evicted due rows as not queued (`TASK-22-ROLLING-WINDOW-AUDIT.md`).

**Rule:** Priority chooses the lane. Age walks every lane toward `p0`. Time on the packet is `scheduled_utc`. The body is not edited to make the queue look fair.
