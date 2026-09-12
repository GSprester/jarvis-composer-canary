# budget-aware routing

Router (`LEGACY-28.md`) picks `(seat, provider)` per `enqueue_id`. Each pair has **included remaining** `I` and **overage permitted** `O∈{0,1}` (wrapper table). `I` is a completed quota MEASURE. **Fail if missing `I` ⇒ 0:** failed look looks like no entitlement (`POSITIVE-CONTROL.md`). **Fail if missing `I` ⇒ ∞:** you route into 402 (`TASK-28-ERROR-CLASSIFICATION.md`). Probe ERROR ⇒ ineligible this pop, not “cheap.”

`admit(seat, provider)` still `rank(provider.class)≤rank(seat.ceiling)` (`TASK-13-FAILOVER-CEILING.md`). **Fail if a cheaper wider provider lifts the ceiling.**

Work order is `order_key` (`LEGACY-36.md`). Routing does not rewrite `priority` (`LEGACY-21.md`).

## Consume included, no overage

Eligible = admitted ∧ quota probe completed ∧ (`I>0` ∨ (`O=1` ∧ no included left **anywhere** eligible)).

```
pick(unit):
    E = {pairs: admit ∧ probe≠ERROR}
    inc = {p∈E: I(p)>0}
    if inc: return argmin (I after reserve, not unit_price) among inc
    if any p∈E with O=1: escalate; do not hop  # see preserved
    else: queue / UNKNOWN — do not send
```

**Policy:** burn **included** first; **never** send a unit that will bill overage while any admitted pair still has `I>0`. **Fail if you send overage to “keep the cheap seat warm.”** **Fail if 429 on an included pair ⇒ 402/overage hop:** rate-limit is time, not entitlement. Same `enqueue_id`, backoff (`TASK-27-RETRY-SEMANTICS.md`). **Fail if 402 ⇒ retry same pair:** tight loop. **Fail if 402 ⇒ new `unit`.** Decrement `I` only after **confirmed delivery** (A∧B or `released` with evidence) (`IDEMPOTENT-EVENT-LEDGER.md`). **Fail if you decrement on `promoted`:** crash pre-verify burns quota the worker never used. **Fail if you decrement on HTTP 200.**

Quota probe: three-state (`LEGACY-38.md`). **Fail if empty body ⇒ I=0=NONE.** Canary-hit on the quota table (`LEGACY-39.md`).

## Preserved-capacity rule

Each pair declares `reserve` (integer, default 0): tokens of `I` that **must not** be used unless `effective_priority==0` (aged `p0`, `LEGACY-34.md`).

```
usable(p, prio) = max(0, I(p) - (0 if prio==0 else reserve(p)))
```

`inc` uses `usable>0`, not raw `I>0`. **Fail if reserve=0 on the only high-class seat:** a `p3` flood spends `restricted` included; overnight `p0` pays overage or waits. **Fail if reserve=I:** that pair never runs except `p0`, so `p1–p3` starve **or** leak to overage elsewhere. **Fail if reserve is a dollar amount:** `I` is counts, not price. **Fail if you preserve by holding a second lock “for later”:** deadlock (`GATE-DEADLOCK.md`).

Substitutes covering an offline primary (`SUBSTITUTE-REVIEW-QUEUE.md`) use **their** `I`/`reserve`, not `P`’s. **Fail if cover inherits `P`’s remaining included.**

After pop, `usable` is recomputed from a **new** MEASURE. **Fail if you cache `I` across days:** silent reset / rolled window (`LEGACY-27.md`).

## Cheapest-first — failure mode

Sort by `unit_price` (or “the cheap seat”) and send:

- **Included inversion:** cheapest `I` hits 0 on `p3`; next `p0` is expensive overage. Included on a dearer pair sits unused — violates the policy above.
- **Class widen:** cheapest vendor is `public` while the WO needs `internal`; a sloppy min-price hop rewrites ceiling (`TASK-12-POLICY-PARITY.md`).
- **402 as cheap:** list price is $0 but entitlement is exhausted; you spin 402 and call the key dead (`LEGACY-23.md`).
- **429 as empty:** cheapest is rate-limited; you skip its remaining `I` and burn another seat’s included.
- **Starvation of reserved:** cheapest has `reserve=0`, so it always wins `argmin(price)` and empties before `p0`.
- **Wrong `list_next`:** UI is cheap-first, dispatcher is not (`LEGACY-35.md`).

**Fail if you add random tie-break:** non-reproducible `R` (`BLAST-RADIUS-ESTIMATE.md`). Tie-break: `(usable desc, order_key, seat)`.

Overage, if ever used, is a **human exemption** (`LEGACY-22.md`): `would_deny` on `I==0`, grantor, `policy_sha256`. **Fail if the router grants itself overage to clear the queue.**

**Rule:** Admit first. Spend `usable` included. Preserve `reserve` for aged `p0`. Overage is an exemption, not a sort key. Price is not a gate.
