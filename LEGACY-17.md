# LEGACY-17

**The gate must never arbitrate the command that satisfies it.** The gate is the predicate (admit, lock alive, `verify`, `re` match). The command is the work that is supposed to **meet** that predicate (write bytes, acquire lock, emit a receipt). If the same function **chooses or authors** the evidence it then accepts, the check is a self-report (`TASK-10-SELF-REPORT.md`, `TASK-17-SEPARATION-OF-DUTIES.md`). A third party cannot re-run it. That is why the rule is sound.

A gate may **read**. It may **refuse**. It must not **mint** the artifact, the lock identity, or the success bit that would make it return true. Completion is a **view** over bytes the worker already wrote (`TASK-24-ARTIFACT-DECLARATION.md`). Admit is `min(seat, provider)` (`TASK-23-POLICY-INHERITANCE.md`), not “set seat = provider so we can hop.” Alive is OS `pid+start` (`LEGACY-08.md`), not a sentence the holder writes after the gate asks.

## Counterexample

`verify` is supposed to close the unit only if the **declared** path matches the **declared** digest. A “helpful” gate:

```python
def verify(decl, root):
    p = decl.path
    if not p.exists():
        p.write_bytes(b"")          # command that "satisfies" exists()
    if p.exists():
        return "COMPLETED"          # gate arbitrates the write
```

The file now exists (`LEGACY-05.md`). The digest is empty, not `decl.expected`. The gate **ran the command** (create empty) that its own exists-check accepts. Drain and retry see COMPLETED; a third-party hash does not (`TASK-06-ARTIFACT-CHECK.md`). Same shape: `find_receipts` inserts a dummy row so `[]` is avoided (`LEGACY-03.md`); `admit` writes `seat.ceiling = provider.class` so failover passes (`TASK-12-POLICY-PARITY.md`); a change-detector **runs the job** to refresh the hash it uses to skip the job (`LEGACY-09.md`).

In each case the instrument **produces** the measurement. Deterministic, inaccurate (`LEGACY-10.md`). Fail open (`LEGACY-07.md`).

**Rule:** Gate = measure what is already there, fail closed if it cannot. Command = a different seat or a prior step. If the gate has to act to become true, it was not a gate.
