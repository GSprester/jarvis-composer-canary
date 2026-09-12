# dependency inversion

Consumer `C` embeds a **local model** of system `G`’s gate (`re` match, `admit`, `verify`, `order_key`). `C` then emits success when `G` would not (or refuses when `G` would pass). That is a second predicate (`LEGACY-10.md`, `LEGACY-36.md`). Blast radius is every WO `C` closed on the copy (`BLAST-RADIUS-ESTIMATE.md`).

**Fail if you “keep them in sync” by comment or shared README:** two run paths (`LEGACY-15.md`). **Fail if `C` copies `G` source into the repo:** the next `G` deploy drifts again.

## Interface (remove the duplicate model)

`C` depends on a **port**. `G` owns the implementation. `C` does not compute the predicate.

```
Gate.measure(ask) -> {kind: FOUND|NONE|ERROR, evidence, control}
```

`ask` is declared identity only: `unit` / `re`, `path`, `expected_sha256`, `enqueue_id`, `seat`, `provider` — not filename, not `C`’s cached class (`RECEIPT-MATCHING.md`, `TASK-24-ARTIFACT-DECLARATION.md`).

`kind` is `G`’s scanner type (`POSITIVE-CONTROL.md`). **Fail if the port returns `bool`:** timeout becomes `false`/`NONE`. **Fail if the adapter reimplements `measure` “because RPC is slow”:** the duplicate is back. Adapter = **import or call `G`** (same module `G` ships, or MEASURE HTTP that `G` serves). **Fail if `C` vendors a snapshot of that module and patches it.**

`C` may **transport** (send bytes, hold its own lock for *its* tree). It may not decide `COMPLETED`, match, or pop order (`LEGACY-28.md`, `TASK-17-SEPARATION-OF-DUTIES.md`). `list_next` in `C` is `G.order_key` applied to `G`’s measure results (`LEGACY-35.md`). **Fail if `C` sorts locally “for the UI.”**

Evidence on `FOUND` is `G`’s (rung, both digests, `n=1`). **Fail if `C` adds a similarity score.** `ERROR` from `G` ⇒ `C` ERROR; no local fallback (`LEGACY-38.md`). **Fail if `G` timeout ⇒ use last local answer:** LKG of the drift (`TASK-08-WATCHDOG-PATTERN.md`).

Canary-hit/miss stay **inside `G.measure`** (`LEGACY-39.md`). **Fail if `C` implements its own canaries against a different store.**

`C` does **not** invert by making `G` import `C`’s types. **Fail if `G` must know `C`’s WO table:** `G` becomes the second copy of `C`.

## Migration order

Do **not** cut over on a flag that leaves both predicates live (split-brain). Do **not** delete the local model before the adapter has completed looks.

1. **Ship the port + adapter** in `C`. Local model still decides. Every ask dual-calls `G.measure` (MEASURE, no mutate). Log `{local, G, ask, event_id}`. **Fail if mismatch is ignored or “prefer local”:** you automate the lie. Mismatch ⇒ `ERROR` on the **log**, not a user-visible NONE.
2. **Canary window:** `G` controls pass; mismatch count = 0 for `N` completed looks (`last_successful_scan` UTC). **Fail if `N` is wall-clock-only with zero looks:** silence (`LEGACY-37.md`). **Fail if you sample only the 74 `re`-quoted receipts:** the 494 are the drift (`RECEIPT-MATCHING.md`).
3. **Cut over:** `C` uses **only** `G.measure`. Local functions not on any run path. User-visible ERROR if `G` ERRORs. **Fail if 50/50 flag:** two truths. **Fail if cut over while mismatch>0.**
4. **Tombstone** the local matcher/admit/verify: import raises (`LEGACY-16.md`). CI greps the retired symbol. **Fail if the module remains `pip`-able.**
5. **Bound R** on the dual-run log (local FOUND ∧ `G` not FOUND) (`BLAST-RADIUS-ESTIMATE.md`). Relabel those `unit`s UNVERIFIED. **Fail if you skip R because “we’re inverted now.”**

Harness retirement of `C` keeps the port; successor calls the same `G` (`LEGACY-29.md`). **Fail if successor re-embeds “a simpler copy.”**

Policy: `admit` stays `G`’s `min(seat, provider)`. **Fail if `C` caches `seat.ceiling` from an old `G` body and hops.**

## Still refuse

`C` writing `G`’s receipts, rewriting `expected`, fuzzy join, steal on UNKNOWN, `released` before delivery, presence-only close. The port is MEASURE. **Fail if `C` uses the port to ACQUIRE `G`’s lock.**

**Rule:** One gate, `G`’s. `C` calls it. Dual-run until mismatch is 0, then tombstone the copy. ERROR, not a local guess.
