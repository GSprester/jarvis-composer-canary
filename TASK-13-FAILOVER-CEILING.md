# TASK-13-FAILOVER-CEILING

Proposal only. This checkout has no seat runtime. The design closes the silent-widen path in `TASK-12-POLICY-PARITY.md`: failover must not raise the sensitivity class a seat may handle.

## Seat-owned ceiling

Sensitivity authorization is a field on the **seat**, fixed at writer admission (same check as a cold start, `TASK-05-DOCTRINE-DRAFT.md`):

```
seat.ceiling ∈ { public, internal, restricted, ... }
```

It is not read back from whichever provider answered. Provider entries may advertise their own `class`. That advertisement is a **capability of the vendor**, not an authorization for this seat.

On every dispatch, including the first hop and every failover hop:

```
admit(seat, provider)  iff  rank(provider.class) ≤ rank(seat.ceiling)
```

If the destination ranks above `seat.ceiling`, refuse. Record a wrapper disposition (`failover_denied_ceiling`). Do not call the provider. Do not ask the model. The seat object is unchanged.

Failover may change **who executes** the call. It must not change **what the seat is allowed to handle**. A provider that can do `restricted` is unusable to a seat whose ceiling is `internal`, even if it is the only remaining vendor.

## Smallest enforcement point

One predicate, one call site:

```
admit(seat, provider) -> bool
```

implemented as `rank(provider.class) ≤ rank(seat.ceiling)`, with missing/malformed `class` treated as deny (`TASK-02-POLICY-AUDIT.md`: no fail-open).

That is the smallest enforcement point: a single boolean used as a gate, not a policy engine and not a per-SDK hook.

## Where it lives

It lives in the **wrapper’s provider-select / failover function** — the only code that can change the provider name on an in-flight seat.

- Not in the model (the model is not the recorder; `TASK-05-DOCTRINE-DRAFT.md`).
- Not in the provider SDK (the destination vendor has no copy of `seat.ceiling`).
- Not in a post-hoc log scrape (that is after the widen).
- Not on the provider document alone (`TASK-12-POLICY-PARITY.md` shows name-keyed rows will not bind a seat).

Concrete home: the function that today does `provider = next_failover(name)`. Replace with:

```
provider = next_failover(name)
if not admit(seat, provider):
    record_disposition("failover_denied_ceiling")
    raise FailoverDenied
dispatch(seat, provider)   # seat.ceiling still the request ceiling
```

No other site may assign `current_provider`. If a second failover path is added later, it must call the same `admit`.

## What this does not do

- It does not lower a provider’s advertised class. `beta` may still be `restricted` for seats whose ceiling is `restricted`.
- It does not invent a completion claim. A denied failover is UNVERIFIED work plus a wrapper disposition, not `COMPLETED` (`TASK-10-SELF-REPORT.md`).
- It does not replace `policy_parity.py`. Parity remains how operators see `alpha` vs `beta` differ by one class; `admit` is how that delta is stopped at the seat.
