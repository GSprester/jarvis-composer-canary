# LEGACY-06

Ten questions before trusting a **status field written by the process it describes** (`TASK-10-SELF-REPORT.md`). If any answer is no or UNKNOWN, the field is a **record**, not a check. Treat the unit as UNVERIFIED.

1. **Can a third party re-derive this status without the writer?** If the only way to get `completed` is to read the worker’s row, the author and the verdict are the same process.

2. **Is there a declared path and digest that existed before the run?** Declare-then-verify (`TASK-24-ARTIFACT-DECLARATION.md`). A status with no prior declaration is a self-award.

3. **Does `verify(path, declared_sha256)` return COMPLETED?** Existence is not enough (`LEGACY-05.md`). Empty, stale, and wrong-hash files all exist.

4. **Was the digest computed after canonicalisation, not before?** Hash-then-normalise embeds a lie (`TASK-15-IDEMPOTENT-RECEIPT.md`). Two writes of the same handoff must be byte-identical.

5. **Did flush happen before the status bit?** Completion-before-durable inverts the guarantee (`TASK-31-GRACEFUL-SHUTDOWN.md`). A crash after `status=done` and before fsync leaves a green row and UNVERIFIED bytes.

6. **Could this status have been written on a failed look?** `[]` from timeout is not “no work left” (`LEGACY-03.md`). Is there `last_successful_scan` in UTC?

7. **Is the writer still the one-lead lock holder with matching `pid+start`?** A second seat or an empty-lock steal can write status for work it did not run (`LEGACY-01.md`).

8. **Does `receipt.re` name this handoff, not a date prefix?** A prefix match can attach another day’s receipt (`TASK-07-DATE-ROLLOVER.md`). Status “matched” is not identity.

9. **Would a later seat, with a cold-start admission, reach the same bit?** Resume must not inherit in-seat memory (`TASK-05-DOCTRINE-DRAFT.md`). If only the original process “knows” it is done, it is not done.

10. **If this row vanished from the hot table, would an archive still prove the claim?** A wrap can zero `open_units` and leave `completed` as folklore (`TASK-34-METRIC-THAT-LIES.md`). No archive-before-evict → do not trust the bit.

A yes to all ten still does not let the worker **set** COMPLETED. COMPLETED remains a view over the checker (`TASK-17-SEPARATION-OF-DUTIES.md`).
