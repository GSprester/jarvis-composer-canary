# Shared rate-limit ceiling

**Status:** design only.  
**Depends on:** `BUDGET-AWARE-ROUTING.md`, `TASK-28-ERROR-CLASSIFICATION.md`, `TASK-13-FAILOVER-CEILING.md`, `BLACKBOX-MODEL-PROBE.md`.  
**Does not exist in this tree:** router, two provider clients, remaining-count headers.

Two providers `A` and `B` look like separate accounts (`key_id`, name, invoice line). They may still share one **429 bucket**. Name-keyed failover then spends the same ceiling twice: `A` 429 ⇒ hop `B` ⇒ 429 ⇒ storm (`TASK-12-POLICY-PARITY.md` name-keyed hole). Discovery is a **wire** MEASURE. Routing treats a shared bucket as **one** `usable`.

429 is time, not entitlement (`TASK-28`). Shared 429 does not imply shared 402. Do not probe a **wider** class to find the bucket (`TASK-13`).

## Discovery

Do not trust `T` (“we are two orgs”), dashboard org id, or a README current-claim (`STALE-DOC-CLAIMS.md`, `MODEL-ROUTE-IDENTITY.md`). Those are invisible overrides of the real limiter.

Issuer-planted canaries, **same** `admit` class, same UTC window, wrapper holds secrets (`LEGACY-24.md`):

```
discover(A, B):
    look_A, look_B completed          # else UNKNOWN; do not classify
    # 1. Drive or wait for A in 429 (Retry-After or remaining==0)
    # 2. Immediately send canary-hit on B (B advertised remaining > 0 if a header exists)
    # 3. Classify
```

| A | B (next completed send) | Verdict |
|---|---|---|
| 429 | 429, `Retry-After` overlaps A’s window, B had not spent its advertised remaining | **`shared_429`** |
| 429 | remaining-header on B equals A’s remaining **without** B having sent (if the issuer exposes a read) | **`shared_429`** |
| 429 | 2xx / bound | **`independent_429`** |
| 429 | 402 | not a rate-share; B’s entitlement (do not call it shared 429) |
| any | timeout / unreadable | **UNKNOWN** |

One accepted independent 2xx on B during A’s 429 is enough to refuse `shared_429` for that window. One correlated 429 is enough to **set** `shared_429` for that pair (zero-accept the other way: do not require a week of correlation). Re-run when keys or endpoints change (`LEGACY-27.md` silent reset).

**Not discovery:** same `M` SKU (`SAME-MODEL-PAIR.md`), same region, similar latency, or the vendor’s org string. Those are bias / marketing.

Canary-miss: plant two keys the issuer **knows** share a limiter; if `discover` returns `independent_429` and `pick` hops on 429, the instrument is always-split. Canary-hit: two keys the issuer knows are isolated; `discover` must not force-share.

Do not burn prod units for discovery. Do not decrement `I` on the probe unless delivery completed (`BUDGET-AWARE-ROUTING.md`).

## Routing rule (shared ceiling)

When `shared_429(A,B)`:

```
bucket G = {A, B}                  # maybe more keys later; one remaining
I_G      = one MEASURE of remaining in the shared window
           (min of advertised remainings if both speak; else
            “in 429 until Retry-After”)
usable(G, prio) = max(0, I_G - reserve_G)
```

`pick` sees **G**, not two pairs with two `I`s:

```
if A in G and B in G:
    if G in 429:                  # completed look
        do not send A or B
        backoff until Retry-After  # same enqueue_id
        do not failover A→B
        latch if the pool’s only hops are in G   # DISPATCH-POOL-LATCH kind≈quota-time
    else:
        pick one member of G under admit(seat, member)
        decrement I_G only after A∧B / released
```

| Forbidden | Why |
|---|---|
| `A` 429 ⇒ `B` now | Same ceiling; second send is the storm |
| Sum `I_A + I_B` | Double-counts one bucket |
| Cache `independent` across key rotation | Silent merge (`LEGACY-27`) |
| Shared 429 ⇒ treat as 402 / hop a wider vendor | Class lift (`TASK-13`); 429 ≠ 402 |
| Discover by sending `restricted` from an `internal` seat | Ceiling probe |

`admit(seat, A)` and `admit(seat, B)` still apply **per hop**. Sharing a limiter does not widen class. If only `B` admits and `G` is in 429, queue — do not send `B` anyway.

Independent pair: keep today’s `pick` (two `I`s, 429 on A backs off A only; B may still have `usable`).

`list_next` shows `G` as one column. Two green “accounts” with one 429 window is the metric that lies (`TASK-34`).

## What this must not do

- Trust account names as isolation.
- Failover on 429 inside a discovered `G`.
- Probe a wider class or a prod bag to “see if they share.”
- Collapse 429 into 402 because two keys died together.

**Rule:** `shared_429` iff a completed send to `B` is 429 in `A`’s window while `B` had not spent a separate remaining. Route `G` as one `usable`; never hop `A→B` on 429. Independent 2xx on `B` during `A`’s 429 keeps two ceilings.
