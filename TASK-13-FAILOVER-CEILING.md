# TASK-13-FAILOVER-CEILING

Proposal only. This checkout has no seat runtime and no `admit()` implementation. The design closes the silent-widen path in `TASK-12-POLICY-PARITY.md`: failover must not raise the sensitivity class a seat may handle.

## Seat-owned ceiling

Sensitivity authorization is a property of the **seat**, fixed at writer admission (same check as a cold start, `TASK-05-DOCTRINE-DRAFT.md`):

```
seat.ceiling ∈ { public, internal, restricted, ... }
```

It is not read back from whichever provider answered. A provider entry may advertise `class`. That is a **vendor capability**, not this seat’s authorization.

On every dispatch, including the first hop and every failover hop:

```
admit(seat, provider)  iff  rank(provider.class) ≤ rank(seat.ceiling)
```

Missing or malformed `provider.class` is deny (no fail-open). If the destination ranks above `seat.ceiling`, refuse. Record a wrapper disposition (`failover_denied_ceiling`). Do not call the provider. Do not ask the model. Do not assign `seat.ceiling = provider.class`. The seat object is unchanged.

Failover may change **who executes** the call. It must not change **what the seat is allowed to handle**. A provider that can do `restricted` is unusable to a seat whose ceiling is `internal`, even if it is the only remaining vendor.

Clamping after the hop (`effective = min(seat, provider)`, `TASK-23-POLICY-INHERITANCE.md`) is not enough. The wider provider still received the request. Refuse before the call.

Worked hop from the parity example: seat `ceiling=internal` on `alpha` (`class:internal`). Failover target `beta` (`class:restricted`). `admit` is false. Disposition `failover_denied_ceiling`. Work stays UNVERIFIED.

## Smallest enforcement point

One predicate, one call site:

```
admit(seat, provider) -> bool
```

implemented as `rank(provider.class) ≤ rank(seat.ceiling)`.

That is the smallest enforcement point: a single boolean used as a gate. Not a policy engine, not a per-SDK hook, not a document diff. `policy_parity.py` shows that `alpha` and `beta` differ by one class; `admit` is what stops that delta at the seat.

## Where it lives

It lives in the **wrapper’s provider-select / failover function** — the only code that can change the provider name on an in-flight seat.

- Not in the model (the model is not the recorder; `TASK-05-DOCTRINE-DRAFT.md`).
- Not in the provider SDK (the destination vendor has no copy of `seat.ceiling`).
- Not in a post-hoc log scrape (that is after the widen).
- Not on the provider document alone (name-keyed rows do not bind a seat; `TASK-12-POLICY-PARITY.md`).

Concrete home: the function that today does `provider = next_failover(name)`. Replace with:

```
provider = next_failover(name)
if not admit(seat, provider):
    record_disposition("failover_denied_ceiling")
    raise FailoverDenied
dispatch(seat, provider)   # seat.ceiling still the request ceiling
```

No other site may assign `current_provider`. A later second failover path must call the same `admit`.

## What this does not do

- It does not lower a provider’s advertised class. `beta` may still be `restricted` for seats whose ceiling is `restricted`.
- It does not invent a completion claim. A denied failover is UNVERIFIED work plus a wrapper disposition, not `COMPLETED` (`TASK-10-SELF-REPORT.md`).
- It does not replace `policy_parity.py`. Parity remains the operator diff; `admit` remains the seat gate.
