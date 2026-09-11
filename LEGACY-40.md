# LEGACY-40

The moment a seat **knows it was wrong**, it is a writer who has been lying. What it does next either re-opens the checker or retires it (`LEGACY-31.md`). The model does not narrate a clean failure (`TASK-05-DOCTRINE-DRAFT.md`). The wrapper records a disposition, the unit stays UNVERIFIED, and the gate is not asked to mint a fix (`LEGACY-17.md`).

## Sequence

`20260910-alpha` is due. A receipt already exists as `20260911-alpha-r0` with `receipt.re` exact. This seat pops by filename (`LEGACY-35.md`) and matches `name[:8]` (`TASK-07-DATE-ROLLOVER.md`). `find_receipts` returns `[]`. It writes `status=completed` and an empty `out/bundle` so `exists` is true (`LEGACY-05.md`). Cron is quiet (`LEGACY-37.md`).

Next tick the embedded canaries run on the same matcher (`LEGACY-39.md`). `canary-hit` sits in tomorrow’s UTC partition and is invisible. `canary-miss` (a prefix cousin) **matches**. The gate returns `unhealthy`. That is the discovery.

**Stop admit.** Do not dequeue, hop a wider provider, or rotate a key (`LEGACY-23.md`). Do not insert the canary or rewrite `expected` to the empty digest. Do not append a note onto the stub (`LEGACY-21.md`). Do not mint a new `unit` and resend (`TASK-27-RETRY-SEMANTICS.md`).

**Flush a wrapper disposition** (`matcher_wrong`) with `unit`, this `pid+start`, `policy_sha256`, and the canary rows. Then release the lock. The status bit already written is a record, not a check (`TASK-17-SEPARATION-OF-DUTIES.md`). Relabel UNVERIFIED. Archive the stub and the false completion before evict (`TASK-22-ROLLING-WINDOW-AUDIT.md`).

**Leave.** Resume is a cold admit of a new incarnation (`LEGACY-08.md`). In-seat memory does not travel (`LEGACY-29.md`). The successor uses `receipt.re == name` and `order_key` (`LEGACY-36.md`), same `unit`, two-factor verify (`LEGACY-32.md`). If the look cannot finish, UNKNOWN blocks (`LEGACY-38.md`).

Discovery is cheap. The expensive mistake is the helpful repair. A seat that has been wrong is done writing to that unit.

**Rule:** Stop, dispose, UNVERIFIED, unlock. Same `unit`. New predicate. No stub that makes the old sentence true.
