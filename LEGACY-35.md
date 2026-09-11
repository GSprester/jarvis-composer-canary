# LEGACY-35

A **listing** that claims to show “what happens next” but sorts by a **different key** than the action is a wrong response that looks complete (`LEGACY-26.md`, `LEGACY-10.md`). Operators treat line 1 as the next dequeue, the next kill, the next archive. The action then touches another `unit`. That is not a UI nit. It is a second, silent lead (`LEGACY-01.md`): the human and the worker disagree about identity.

## How the error happens

Handoffs are named `p{n}-{utc}-{topic}-{unit}` for `ls` (`LEGACY-11.md`). Lexical order is filename. The queue pops `min(effective_priority, scheduled_utc, unit)` (`LEGACY-34.md`). Those orders diverge:

- Fresh `p0-20260911T120000Z-…` sorts **before** an aged `p3-20260911T080000Z-…` on disk. After 180 s the `p3` is effective `p0` and **older**, so the worker takes the `p3`. The operator “cancels the first row” and kills the `p0`. The starved unit still runs. The cancelled unit was merely old and live (`LEGACY-14.md`).
- Local listing of `YYYYMMDD` vs UTC `scheduled_utc` (`TASK-19-TIMESTAMP-HAZARD.md`): “today’s” first line is tomorrow’s work. Drain archives the wrong day; `re` still matches last night (`TASK-07-DATE-ROLLOVER.md`).
- `ls` by mtime after a touch/reset (`LEGACY-27.md`): newest file is a silent default. The action still keys `unit`. Operator “retries the top one” and double-sends a different token (`TASK-27-RETRY-SEMANTICS.md`).
- Empty listing from a failed look (`LEGACY-03.md`, `LEGACY-33.md`): operator reads “nothing next,” action still has a canary-visible bag. They skip the checker (`LEGACY-31.md`).

The listing is an error message that does **not** say it used a display key (`LEGACY-25.md`). Silence + pretty order looks like health (`LEGACY-04.md`).

## The fix

The list command is a **view of the same predicate** as the action. Not a directory walk.

```
list_next() == the sequence dequeue() would return
              using the same compare key
              after the same completed look
```

- Sort key **is** `(effective_priority, scheduled_utc, unit)` — body fields, UTC, aging at `now_utc`. Print those columns. Filename is last and unlabeled as “order.”
- Line *i* is the *i*-th pop. A test fails the build if `list_next()[0].unit != peek()`. No filename sort, no mtime sort, no `name[:8]`.
- Failed look → raise / UNKNOWN, not `[]` that looks like “idle.” Positive control still required (`LEGACY-33.md`).
- Actions that take an operand (`cancel`, `archive`, `kill`) take **`unit`**, never “the first line” parsed from a stale table. If they accept an index, the index is from **this** `list_next` snapshot, same `pid+start` of the listing process, discarded after one use.

Do not “fix” it by renaming files so `ls` matches (`LEGACY-11.md`). That mutates display to fake the gate (`LEGACY-17.md`). Change the listing, or stop calling it a preview of the action.

**Rule:** If the command is named like the verb, its order **is** the verb. Otherwise it is decoration and must not be the operator’s index.
