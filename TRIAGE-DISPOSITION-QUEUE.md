# triage disposition queue

Triage (LIVE / SUPERSEDED / DUPLICATE / INFO / UNKNOWN) **creates a disposition queue by the act of labelling**. The inbox count falls. The work is **moved**, not reduced. This checkout has no triage runtime. The design is that move, and the shape that does **not** mint a second backlog for a downstream reviewer (`SUBSTITUTE-REVIEW-QUEUE.md`, `BACKLOG-TEMPLATE-CLASS.md`, `TASK-25-QUEUE-BACKPRESSURE.md`).

```
item  →  triage  →  disposition(item)
```

`disposition` is a **wrapper event** on the **same** `unit` / `enqueue_id`. It is not a new work order.

## Work is moved, not reduced

Let **N** be backlog items. After one completed triage pass you have **N dispositions** (or **C + U** after template collapse: one row per digest plus the tail). Nothing has been verified. Nothing has been actioned (`TASK-10-SELF-REPORT.md`).

| Label | Where the work went |
|---|---|
| LIVE | Still the original obligation — do the unit, then A∧B |
| SUPERSEDED | Prove a **named** survivor (`re`); else it is a buried LIVE |
| DUPLICATE | Prove the sibling `unit`; wrong sibling is a unique constraint lost |
| INFO | Holdout / zero-acceptance sample (`CLASSIFIER-ZERO-ACCEPT.md`); not “done” |
| UNKNOWN | Failed look — stays in the **same** bag (`TASK-20-FAIL-LOUD.md`) |
| TEMPLATE cluster | One MEASURE of the digest; `count` copies are not N closes |

Conservation:

```
N_open + N_dispositioned  =  N
N_completed               =  |{ unit : verify(path, expected) == 0 }|
```

`N_dispositioned` rising and `N_open` falling is a **transfer**. Dashboard “inbox 0” with `N_completed` unchanged is a lying zero (`TASK-34-METRIC-THAT-LIES.md`). A mailbox of “please review these labels” is the transfer made visible as a **second pile**.

Burst-review already names the failure: substitutes open `review_id`s until `P` returns to a bag that is **another** queue (`BURST-REVIEW.md`). Triage does the same in one sitting if each label mints a review item.

## What produces the second backlog

| Act | Second pile |
|---|---|
| New `unit` / `review_id` / ticket per label | Parallel identity (`TASK-27-RETRY-SEMANTICS.md`) |
| Copy notes into `P`’s mailbox or chat | Offline drop; ACK as close (`ADVERSARIAL-SELF-REVIEW-QUEUE.md`) |
| `review_opened` for every SUPERSEDED / INFO | Downstream must census kills |
| Model writes `review_closed` / `status=triaged` | Self-satisfied gate (`LEGACY-17.md`) |
| Filename `YYYYMMDD-triage-*` | Date-prefix join, UTC rollover (`TASK-07-DATE-ROLLOVER.md`) |

The downstream reviewer then has **two** `list_next`s: the original WOs and the disposition inbox. That is two leads on the same obligation (`LEGACY-01.md`) unless one of them is only MEASURE.

## Design that does not mint a second backlog

**One bag. One identity. Disposition is a field, not a child unit.**

```
append triage_recorded { unit, enqueue_id, label, rubric_sha256,
                         survivor_re | null, pid, start, at_utc }
flush; hwm                              # TASK-11
# no new unit, no review_id, no mailbox copy
```

1. **Same `unit`.** `list_next` for everyone is the original ledger / WO bag, filtered by label if needed. Priority stays `order_key` on the WO body (`LEGACY-36.md`). Do not sort by who triaged (`LEGACY-35.md`).
2. **Collapse before label.** Byte-identical templates are one digest, one disposition, `count=T` (`BACKLOG-TEMPLATE-CLASS.md`). Do not open 430 review rows.
3. **Kills are not a queue.** SUPERSEDED / DUPLICATE / INFO do **not** enqueue a reviewer. They stay on the unit as `triage_recorded`. Human spend on kills is a **zero-acceptance sample** of size `n(p0,β)` (`CLASSIFIER-ZERO-ACCEPT.md`), drawn from that label, not a full dump. One miss fails the stratum; it does not create N tickets.
4. **LIVE stays the only action queue.** Bounded (`TASK-25`). Downstream work is `on_launch(unit)` on LIVE (or UNVERIFIED) rows — the same path as before triage. Triage did not add a hop.
5. **Close is still A∧B**, same checker, same `expected_sha256` declared before the seat (`LEGACY-32.md`). `triage_recorded` is inferred at best (`EVIDENCE-TIERING.md`). It never writes COMPLETED.
6. **Writer split.** Wrapper (or cheap deterministic collapse) appends `triage_recorded`. The triaging model does not close. A downstream human may MEASURE; they do not ACK a mailbox to drain it.
7. **Idempotent re-triage.** Same `{unit, enqueue_id, rubric_sha256, label}` is one event (`TASK-26`). A second pass must not mint a second pile.

`list_next` after triage:

```
action   = units with label ∈ {LIVE, UNKNOWN, unlabelled}  # same bag
sample   = SRS of {SUPERSEDED, DUPLICATE, INFO}            # not a queue
closed   = A∧B only
```

The reviewer who “comes next” is either (a) doing LIVE work on the original units, or (b) scoring a **bounded sample** of kills. They are not the owner of a new inbox of N dispositions.

## What this does not do

- It does not treat `N_open → 0` as reduced work.
- It does not open `review/` for triage.
- It does not let Model-B “verify” by writing a parallel ticket.
- It does not delete dispositions to make the second pile disappear (`STALE-CLAIM-RECOVERY.md`: archive, same unit).

**Rule:** Triage moves a label onto the existing unit. It does not create a unit. The only queues that remain are the original action bag and a sized sample — never a disposition inbox of size N.
