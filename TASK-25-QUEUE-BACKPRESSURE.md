# TASK-25-QUEUE-BACKPRESSURE

Design only. This checkout has no agent runtime or command runner. Producers are LLM seats; consumers are **shell commands** (one process per job). The queue is a **bounded** buffer of declared units (`TASK-24-ARTIFACT-DECLARATION.md`), not an unbounded list of prompts.

## Shape

```
seat --admit--> enqueue(unit) --N slots--> worker --shell--> declare/verify
```

- **Capacity N = 32** in-flight units (stated estate, not measured here). Each slot holds a handoff id, declared path+digest, seat id, and `pid+start` of the consumer if running (`TASK-16-ONE-LEAD-LOCK.md`).
- One consumer pool: **K = 4** shell workers. A unit is dequeued only when a worker is free.
- Enqueue is **synchronous** for the producer: the seat’s `enqueue` either accepts or refuses. There is no “fire and forget” list.

Backpressure is the refuse (or block-with-timeout) path. It is not a larger N.

## What the producer does when the queue is full

When `len(queue) == N`:

1. **Do not enqueue.** Do not drop the unit on the floor as “success.”
2. **Do not write `COMPLETED`.** Full is not a checkable artifact (`TASK-10-SELF-REPORT.md`).
3. Return **`UNKNOWN` / `queue_full`** to the wrapper (`TASK-21-WATCHDOG-SILENCE.md`). Disposition is recorded by the wrapper, never by the model (`TASK-05-DOCTRINE-DRAFT.md`).
4. The seat **retries with the same handoff id** after a bounded wait, or the operator resubmits. The receipt writer is idempotent (`TASK-15-IDEMPOTENT-RECEIPT.md`): a later accept must not create a second logical unit.
5. Optional short **block ≤ 2 s** for a slot. On timeout: same `queue_full`, no silent accept. Do not block the model context for minutes (that looks like a hang and invites a second seat to double-submit).

```python
def enqueue(unit, queue, n=32, wait_s=2.0):
    if queue.offer(unit, timeout=wait_s):
        return "accepted"
    raise QueueFull("queue_full")   # TASK-20: do not return []
```

A second seat that sees `queue_full` must not open a parallel unbounded side channel (“just run the shell”). That bypasses the bound and the one-lead lock.

## Why unbounded queues turn a slow consumer into an outage

Consumers are shells: they wait on CPU, disk, locks, and the artifact checker. Their rate **C** (units/hour) is small and bursty. Producers (agents) can emit faster than C as soon as a fan-out or retry storm starts (`TASK-22-ROLLING-WINDOW-AUDIT.md` 10,000 events/day).

If the queue has no cap:

- Each refused-looking success is actually a **list append**. Memory and the rolling event table grow with `P − C`.
- The job store and lock table fill; wrap evicts audit rows **before** the shells finish. Incidents become unreconstructable while work is still “queued.”
- Agents treat “accepted” as progress, spawn more seats, and retry. That **increases P**. The consumer is unchanged. The gap `P − C` widens.
- Host RAM / fd / tmpfs for pending prompts and half-written artifacts goes to OOM or disk-full. The watchdog may time out and report UNKNOWN; if the queue still accepts, UNKNOWN episodes stack (`TASK-08-WATCHDOG-PATTERN.md`).
- A hard kill leaves thousands of queued units with no `pid+start` (`TASK-16-ONE-LEAD-LOCK.md` empty-lock class): restart thinks they are live work. Reboot sweep cannot tell claimed-from-queue from running.

A slow consumer should **stall producers**. Unbounded accept turns that stall into **memory, audit wrap, and duplicate seats** — an outage that looks like “the queue is healthy; length is just high.”

Length-first health (`len > 0` means busy, `len == 0` means idle) is the same bad shortcut as `TASK-18-COUNT-RECONCILIATION.md`: a huge queue is not success.

## Bound vs outage (stated numbers)

| | Unbounded | Bounded N = 32, K = 4 |
|---|---|---|
| Producer when shells are stuck | keeps appending | `queue_full` after 32 |
| RAM for 10,000 queued prompts ≈ 20 KiB | ~200 MiB plus artifacts | ≤ 32 × 20 KiB ≈ **640 KiB** in queue |
| Event rows at 10,000/day vs cap 2,048 | wrap in 4.9 h | enqueue failures do not mint 10,000 job rows |
| Slow consumer | OOM / wrap / split-brain | seats wait or get UNKNOWN; LKG not overwritten |

Cost of the bound: some seats see `queue_full` and must retry. That is backpressure. Cost of no bound: the consumer’s slowness becomes everyone’s outage.

## Rule

**Accept at most N units. Full → raise `queue_full` / UNKNOWN. Never grow the list to hide a slow shell.**
