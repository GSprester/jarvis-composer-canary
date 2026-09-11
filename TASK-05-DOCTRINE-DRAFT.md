# TASK-05-DOCTRINE-DRAFT

Draft only. This checkout has no runtime, job store, or seat wrapper. The doctrine is the contract for resume-on-restart in a multi-model estate.

## Central invariant

No completion claim exists without an artifact a third party can check. Anything else is labelled UNVERIFIED.

A model saying “done,” a log line, a status bit, or a resumed session memory is not a completion claim. Only a durable artifact (receipt, bundle, hash-addressed file, or equivalent) that a party who was not in the seat can inspect may close work. Missing, unverifiable, or self-attested completions stay UNVERIFIED.

## Resume-on-restart

Restart does not mint trust. A resumed seat is a new writer admission, identical to a cold start:

- Same admission checks, identity, and allow-list as a process that never ran.
- No carry-forward of in-seat assertions, partial traces, or “we were about to finish.”
- Prior UNVERIFIED labels remain UNVERIFIED until a checkable artifact appears.
- Prior completion claims remain claims only if their artifacts still verify; otherwise they are relabelled UNVERIFIED.

Resume is not a continuation of the model’s word. It is a fresh writer walking into the same store.

## Non-cooperative seats

When a seat will not cooperate (timeout, refuse, crash, ignore wrapper protocol):

- The **wrapper** records the **disposition** (refused, timed out, crashed, protocol-break, reboot_interrupted, or equivalent).
- The **model** is never the recorder of that disposition and never the source of a completion claim for that turn.
- No model-authored “I failed cleanly” or “treat this as done” text is admitted as state.

The wrapper’s disposition record is operational history, not a completion artifact. Unless it is itself a third-party-checkable artifact that meets the invariant, the work stays UNVERIFIED.

## Labels

| Label | Meaning |
|---|---|
| Completion claim | Allowed only when a third party can check an artifact that closes the obligation |
| UNVERIFIED | Default for all other outcomes, including resume without a new checkable artifact |
| Disposition | Wrapper-recorded seat outcome; not a completion claim; never written by the model |
