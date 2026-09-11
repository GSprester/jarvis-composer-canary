# LEGACY-36

The **priority convention** is: body field `priority ∈ {0,1,2,3}` (`0` highest), compare key `(effective_priority, scheduled_utc, unit)`, aging `effective = max(0, priority - floor(age_s / 60))` (`LEGACY-11.md`, `LEGACY-34.md`). The **value** lives once, on the packet. The **algorithm** is what gets copied. Five copies are five predicates. They drift. Operators and workers then see different “next” (`LEGACY-35.md`, `LEGACY-10.md`).

## Five places it is duplicated

### 1. Filename prefix `p{n}-`

Display echo. A second writer treats `P0` / `p0` / `p10` as the rank (`LEGACY-02.md`). Sort-by-name becomes a shadow queue. Dedupe on body collapses two names and the prefix “priority” disappears (`TASK-26-DEDUPE-BY-CONTENT.md`).

### 2. Packet body (and a second schema)

`priority` on the handoff is the source of truth. A wrapper, gold-set, or v2 field `urgency` dual-writes without add-only (`TASK-33-SCHEMA-EVOLUTION.md`). Two keys, two ranks, one `unit`.

### 3. Queue dequeue

`offer` / `pop` reimplements rank and aging — or skips aging and is pure `p0`-first. Starvation returns (`LEGACY-34.md`). A side channel “just run the shell” is a third queue (`TASK-25-QUEUE-BACKPRESSURE.md`).

### 4. Listing / peek / cancel-by-index

`list_next` sorts mtime or `YYYYMMDD` while `dequeue` uses the body key (`LEGACY-35.md`). The human’s index is not the worker’s pop.

### 5. Archive, cron, and N wrappers

Monthly evict and rotators keep bytes and lose the name (`TASK-22-ROLLING-WINDOW-AUDIT.md`, `TASK-30-LOG-ROTATION.md`). Replay sorts FIFO or by generation. Each model SDK maps “high” to a vendor enum (`LEGACY-28.md`). After harness retirement the successor guesses the scale (`LEGACY-29.md`). Silent config reset restores `priority=3` as default and looks like an edit (`LEGACY-27.md`).

## Why centralise

One module, one compare, one parse:

```
parse_priority(body) -> 0..3          # raise if missing / unknown
effective_priority(priority, scheduled_utc, now_utc) -> 0..3
order_key(slot, now_utc) -> (effective, scheduled_utc, unit)
```

Queue, `list_next`, archive replay, and the router **import this**. Filename may echo `p{n}` after `order_key`; it must not compute rank. No `urgency` without writing `priority` too. Tombstone any `sort_by_name` / `priority_from_path` so it raises (`LEGACY-15.md`, `LEGACY-16.md`). CI asserts `list_next()[0] == peek()` and `order_key` is the only symbol that mentions `QUANTUM_S`.

Copies fail open in different directions: one starves, one kills the wrong `unit`, one archives `p0` as FIFO. A third party cannot re-run five ranks. Centralising is the same move as one admit gate: the convention is a **predicate**, not a comment in five READMEs.

**Rule:** Store the number on the packet. Compute the order in one function. Everything that lines up work calls that function or it is not lining up work.
