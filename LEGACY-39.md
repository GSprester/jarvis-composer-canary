# LEGACY-39

A gate that only tests **production units** can be wrong in a way no ticket will show: prefix match, empty-as-none, `exists` as done (`LEGACY-10.md`, `LEGACY-03.md`, `LEGACY-05.md`). Embedding a **canary** means every invoke of the gate also evaluates objects whose outcomes are **known before the call**. The gate’s correctness is then a continuous environment probe (`LEGACY-38.md`), not a weekly suite.

The gate may **read** the canary. It must not **create, repair, or re-hash** it so the look succeeds (`LEGACY-17.md`). Author is the issuer at harness install (facts that survive retirement, `LEGACY-29.md`). Observer excludes itself (`LEGACY-13.md`).

## What is embedded

Two reserved units, same store, same matcher, same `order_key` as live work (`LEGACY-33.md`, `LEGACY-36.md`):

| Id | Role | Known outcome |
|---|---|---|
| `canary-hit` | Positive control | **Must** satisfy the predicate (two-factor done, `LEGACY-32.md`) |
| `canary-miss` | Negative control | **Must not** satisfy: `re` is not this handoff; digest is not `expected`; or class above `seat.ceiling` |

`canary-miss` is how you catch a gate that always returns true (filename prefix, `len==0` shortcut, 401→invalid everywhere). A hit-only canary cannot detect that.

Path, digest, and `re` are declared on a handoff the worker API cannot write. Copy the hit object into every partition the gate might query (UTC day, hot table). Eviction of the canary is UNKNOWN, not “no canary needed.”

## Where it sits in the gate

Same function, same code path — not a sidecar cron that the bypass can skip (`LEGACY-15.md`).

```
gate(ask):
    look_hit  = predicate(canary-hit)    # first
    look_miss = predicate(canary-miss)
    if either look failed to complete:
        return UNKNOWN                   # block; do not answer ask
    if look_hit is false OR look_miss is true:
        return unhealthy                 # gate is wrong; block
    return predicate(ask)                # now [] / deny / verify may stand
```

Apply to `verify`, `find_receipts`, `admit`, `peek` / `list_next`, `alive`. For `admit`, the hit canary is a provider at or under ceiling; the miss is a wider class that must refuse (`TASK-23-POLICY-INHERITANCE.md`). For `peek`, `canary-hit` must appear in `list_next` in `order_key` position (`LEGACY-35.md`). For three-endpoint bind, hit is the declared region; miss is a foreign URL that must stay `denied_here` (`LEGACY-24.md`).

Do not short-circuit: “prod looked fine, skip canary.” The canary **is** the proof the prod look is a look. Do not cache a green canary across process start; resume is cold (`TASK-05-DOCTRINE-DRAFT.md`). Do not log secrets. Rows record `canary_hit`, `canary_miss`, `probe` — not payload bytes.

## Continuous

Every admit, every `find_due`, every verify. Rate is the estate’s rate. A change-detector that skips the job because LKG includes “canary ok” the **job wrote** is the deadlock (`LEGACY-09.md`). Hash the **declared** canary digest, not the gate’s last sentence. CI still ships matcher tests (`LEGACY-16.md`); the embed is for the **live** instrument.

If `canary-hit` is missing: UNKNOWN, block (`LEGACY-38.md`). If `canary-miss` matches: `unhealthy`, block — the predicate is too wide; do not pass prod. Do not insert a stub hit to go green (`LEGACY-32.md`).

**Rule:** The gate grades itself on two fixtures before it grades the unit. If it cannot, it does not speak for the unit.
