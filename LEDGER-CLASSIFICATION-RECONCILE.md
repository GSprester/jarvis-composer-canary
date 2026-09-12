# ledger ↔ classification reconcile

A new classifier must not become a **second source of truth** for a question the append-only ledger already answers (`IDEMPOTENT-EVENT-LEDGER.md`, `TRIAGE-DISPOSITION-QUEUE.md`). This checkout has no live ledger. The design is one fold, one write path, and a reconcile that **appends** — it does not fork a table.

```
question Q  =  label of enqueue_id (or unit) under rubric_sha256
truth(Q)    =  apply(replay(bytes[0:hwm]))
```

`list_next`, dashboards, and Model-B read **only** `truth(Q)`. A side JSON/CSV/model dump that answers Q is a competing log.

## Two truths

| Store | What it claims | Failure |
|---|---|---|
| Ledger `[0,hwm)` | `enqueued` / `promoted` / `released` / `triage_recorded` | The estate, if replay is the fold |
| Classifier file / hot table / chat | Latest LIVE/SUPERSEDED/… | Inbox “reduced”; ledger still OPEN (`TASK-34-METRIC-THAT-LIES.md`) |
| In-memory map after classify | Labels this process believes | Reboot drops or doubles (`TASK-11-IDEMPOTENCE.md`) |

If `list_next` uses the side file and `on_launch` uses replay, the same `enqueue_id` is LIVE in one place and `released` in the other. That is two leads on one question (`LEGACY-01.md`).

Classification does **not** override `released` with `delivery.verify_completed`. A∧B remains the close (`LEGACY-32.md`). A new label on a done id is a **new event** for audit, not a reopen, unless a later `enqueued` with a new `enqueue_id` exists (`TASK-27-RETRY-SEMANTICS.md`).

## One schema, add-only

Append `triage_recorded` on the **same** JSONL. Do not start `classifications.jsonl`. Extra keys ignored; do not drop shipped keys (`TASK-33-SCHEMA-EVOLUTION.md`).

```
type: triage_recorded
unit, enqueue_id, v, at_utc
label ∈ {LIVE, SUPERSEDED, DUPLICATE, INFO, UNKNOWN, TEMPLATE}
rubric_sha256
survivor_re | null
pred_event_id | null        # event_id this row supersedes; null = first
content_h                   # sha256(canonical item bytes)
```

`event_id = sha256(canonical(obj minus at_utc))`. Same classification retried is one line (`TASK-26-DEDUPE-BY-CONTENT.md`). **Fail if `event_id` is a UUID per model run.**

Write path unchanged: `append → fsync → hwm`. Visibility is the mark, not file length.

## Replay fold

Extend `apply` (file order only; **not** `at_utc`):

```
state.labels : enqueue_id → {label, rubric_sha256, event_id, content_h}

triage_recorded:
  if enqueue_id in seen as released.verify_completed:
      record on done[id].history; do not move to queued     # no reopen
  else:
      if pred_event_id is null or pred_event_id == labels[id].event_id:
          labels[id] = ev
      else:
          UNKNOWN hole or stale supersede — do not apply
```

Idempotent: second identical `event_id` is skipped (`seen`). Two different labels for the same id without `pred_event_id` pointing at the first = **competing answers**. Do not last-write-wins on clock. Require the pointer, or refuse the append.

`truth(Q)` after fold is `labels[id]`, or “unlabelled” if no `triage_recorded`. Unlabelled is not INFO.

## Reconcile a side classification into the ledger

When a new classifier already wrote a side bag `S` (files, tickets, in-memory):

```
L = replay(ledger[0:hwm])          # completed look; fail ⇒ UNKNOWN
if canary-hit missing or canary-miss present: ERROR; stop

for row in S:
    q = (row.enqueue_id, row.rubric_sha256)
    if L.labels[q] == row.label and L.labels[q].content_h == row.content_h:
        continue                   # already the truth
    if L.done[row.enqueue_id].verify_completed:
        append triage_recorded { pred_event_id: last, note: post_close }
        do not reopen
    elif L.labels[q] exists and differs:
        # ledger wins until a human MEASURE
        do not copy S over L
        queue a zero-accept sample on the conflict bag
    elif L has enqueue_id and no label:
        append triage_recorded from row, pred_event_id=null
        flush; hwm
    else:
        # S names an id the ledger never enqueued
        REFUSE. Do not invent enqueued. Ghost LIVE (IDEMPOTENT-EVENT-LEDGER).

tombstone S after every row is either appended, skipped as equal, or REFUSED.
list_next must not open S again.
```

Ledger **wins** on conflict. The side bag may **propose** an append; it may not **replace** `[0,hwm)`. Rewriting history (edit a prior JSONL line) is forbidden. A “fix” is a new `triage_recorded` with `pred_event_id` set.

Failed list of `S` or of the ledger is UNKNOWN, not “already reconciled” (`TASK-20-FAIL-LOUD.md`).

## Readers

One function:

```
label(enqueue_id) = replay().labels.get(enqueue_id) or UNLABELLED
```

UI, Model-B, acceptance-rate (`ACCEPTANCE-RATE.md`), and kill samples (`CLASSIFIER-ZERO-ACCEPT.md`) call that. They do not `open(classifications.json)`. After tombstone, grep CI for the side path (`LEGACY-15.md`).

## What this does not do

- It does not mint a second `unit` for a label.
- It does not let the model append `released`.
- It does not sort fold by wall clock.
- It does not treat wrap-evicted `triage_recorded` as “never classified” without archive (`TASK-09-RETENTION-RISK.md`).

**Rule:** The ledger is the only answer to Q. A new classifier appends `triage_recorded` and is folded. A side store is reconciled by append-or-refuse, then tombstoned — never by becoming a second `list_next`.
