# TASK-27-RETRY-SEMANTICS

Which operations may be retried after timeout or UNKNOWN (`TASK-20-FAIL-LOUD.md`, `TASK-21-WATCHDOG-SILENCE.md`). This checkout has no message bus; the table is the contract. Safe means: a second attempt cannot create a second **effect** the store or peer will treat as new work.

Idempotent here is **byte-identical or no extra side effect** (`TASK-15-IDEMPOTENT-RECEIPT.md`), not “we meant the same thing.”

## Classification

| Operation | Retry after timeout? | Condition that makes it safe | If retried unsafely |
|---|---|---|---|
| **Read** | Yes | Pure query; no HWM move, no lock publish | Usually none. A failed read that returns `[]` is the other bug (`TASK-20-FAIL-LOUD.md`) — raise, then retry the **read**, do not invent empty. |
| **Create** | Only if keyed | Safe when create is “write this identity if absent” (handoff id, declared path, lock rename). Second call is no-op or byte-identical. | Unkeyed `INSERT` / `O_EXCL` with a new random id: two rows, two leads, two receipts for one unit (`TASK-26-DEDUPE-BY-CONTENT.md` will not save you if bodies differ by id-only fields you hashed). |
| **Update** | Only if overwrite is last-write-wins **and** the body is canonical | `write_receipt(id, outcome)` twice → same bytes. Compare-and-set on `pid+start` must retry only with the same expected incarnation. | Blind `UPDATE status=completed` or increment: over-claim or double increment. |
| **Delete** | Yes if delete-by-key | `unlink(path)` or evict-by-id: second delete is already-gone. Archive-before-evict (`TASK-22-ROLLING-WINDOW-AUDIT.md`) must not retry **evict** unless archive verify already succeeded. | Delete-next / pop-queue: two retries remove two units. |
| **Send-message** | **Not** unless the send is exactly-once or idempotent at the peer | Safe only with a **dedupe key** the consumer stores (`re` / content hash) and ignores duplicates. | Timeout then send again → **duplicate delivery** (below). |

Default for UNKNOWN: retry **reads** and **idempotent puts**. Do not retry **send** or unkeyed **create** until a check proves the first attempt had no effect.

## Why timeout + resend is the classic duplicate-delivery bug

Send has two steps the producer cannot see as one:

```
t0  producer  send(msg)          # bytes leave the seat
t1  network / consumer  accept   # effect happens
t2  ack  lost or probe times out
t3  producer  sees UNKNOWN
t4  producer  send(msg) again    # second effect
```

At t3 the producer knows **nothing**: the message may already be in the consumer’s queue (shell started, lock published, receipt written). `[]` or timeout is not “it never arrived.”

A second send with a **new** transport id is a second unit. The consumer runs two shells, two artifact writes, or two alerts. Content-hash dedupe (`TASK-26-DEDUPE-BY-CONTENT.md`) can collapse **records** after the fact; it does not un-run the first shell.

```python
def send_or_retry(bus, msg, timeout_s=2.0):
    try:
        return bus.send(msg, timeout=timeout_s)
    except TimeoutError:
        return bus.send(msg)   # WRONG: t1 may already have happened
```

What is allowed instead:

```python
def send_once(bus, msg):
    # msg.id is the handoff id / content hash, not a new UUID per attempt.
    try:
        return bus.send(msg)          # peer: if seen(msg.id): ack, do not run
    except TimeoutError:
        raise SendUnknown(msg.id)     # UNKNOWN; do not send a second body
```

The producer then **reads** (safe): “does a receipt for `re=msg.id` exist?” If yes, do not send. If the read fails, raise — do not send (`TASK-20-FAIL-LOUD.md`).

Queue full (`TASK-25-QUEUE-BACKPRESSURE.md`) is the same shape: `queue_full` is UNKNOWN, not “never queued.” Resubmit the **same** handoff id; do not mint a new send.

## Rule

**Timeout means UNKNOWN, not “no effect.” Retry only operations whose second attempt cannot add a second effect. Send is not one of them unless the consumer keys on an idempotency token.**
