# TASK-05-DOCTRINE-DRAFT

Draft only. This checkout has no runtime, job store, or seat wrapper. The text is the resume-on-restart contract for a multi-model estate (several seats, vendors, and harnesses sharing one store).

## Central invariant

**No completion claim exists without an artifact a third party can check; anything else is labelled UNVERIFIED.**

A model saying “done,” a log line, a status bit, a resumed session, or another model’s summary of the first is not a completion claim. Only a durable artifact (receipt, bundle, hash-addressed file, or equivalent) that a party who was not in the seat can inspect may close work. Missing, unverifiable, or self-attested completions stay UNVERIFIED.

`COMPLETED` is a checker view, not a field any seat can write (`TASK-10-SELF-REPORT.md`, `TASK-17-SEPARATION-OF-DUTIES.md`).

## Resume takes the same writer admission as a cold start

Restart does not mint trust. A resumed seat is a new writer walking into the same store, identical to a process that never ran:

- Same admit, identity, ceiling, and allow-list as a cold start (`TASK-13-FAILOVER-CEILING.md`).
- No carry-forward of in-seat assertions, partial traces, or “we were about to finish.”
- Prior UNVERIFIED labels remain UNVERIFIED until a checkable artifact appears.
- Prior completion claims remain claims only if their artifacts still verify; otherwise they are relabelled UNVERIFIED.
- A successor harness or a different model on the same unit is still a cold-start admit (`LEGACY-29.md`). In-seat memory is not a surviving fact.

Resume is not a continuation of the model’s word.

## Non-cooperative seats: the wrapper records the disposition, never the model

When a seat will not cooperate (timeout, refuse, crash, ignore wrapper protocol, drain-deadline kill):

- The **wrapper** records the **disposition** (refused, timed out, crashed, protocol-break, reboot_interrupted, or equivalent).
- The **model** is never the recorder of that disposition and never the source of a completion claim for that turn.
- No model-authored “I failed cleanly” or “treat this as done” text is admitted as state.

The wrapper’s disposition is operational history, not a completion artifact. Unless it is itself a third-party-checkable artifact that meets the invariant, the work stays UNVERIFIED.

## Labels

| Label | Meaning |
|---|---|
| Completion claim | Allowed only when a third party can check an artifact that closes the obligation |
| UNVERIFIED | Default for all other outcomes, including resume without a new checkable artifact |
| Disposition | Wrapper-recorded seat outcome; not a completion claim; never written by the model |
