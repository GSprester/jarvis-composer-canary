# acceptance-rate metric

```
A  =  n_accept / n_look
```

`n_look` is completed verifier decisions in a UTC window. `n_accept` is those whose verdict was allow / CONFIRMED / COMPLETED / LIVE. This checkout has no metrics backend. The design is why **A near 0 and A near 1 are both uninterpretable**, the **control condition** that makes A a number about the estate, and the **seeded-fault** that proves the verifier can reject (`POSITIVE-CONTROL.md`, `LEGACY-39.md`, `TASK-34-METRIC-THAT-LIES.md`).

## Uninterpretable in both directions

**A → 0** is read as “strict verifier” or “nothing is good.” It is also:

| Mechanism | What actually happened |
|---|---|
| Look failed, counted as deny | Timeout / `[]` / hook deadlock mapped to refuse (`TASK-20-FAIL-LOUD.md`, `GATE-DEADLOCK.md`) |
| Population is all seeded rejects | Instrument is fine; the bag was planted |
| Rubric too wide on refuse | Definitional kill; two models will agree (`RUBRIC-DECORRELATION.md`) |
| Denominator is empty then coerced | `n_look = 0` displayed as 0% (`len==0` shortcut, `TASK-18-COUNT-RECONCILIATION.md`) |

**A → 1** is read as “quality is high” or “verifier works.” It is also:

| Mechanism | What actually happened |
|---|---|
| Rubber-stamp | Verifier cannot or will not refuse (status bit, exists-as-done) |
| No rejectables in the bag | Template mass / one digest, all copies “accept” (`BACKLOG-TEMPLATE-CLASS.md`) |
| Seeded faults never presented | Window wrapped them out (`TASK-22-ROLLING-WINDOW-AUDIT.md`) or they were filtered from `n_look` |
| Failed look mapped to accept | UNKNOWN → healthy; LKG overwritten (`TASK-08-WATCHDOG-PATTERN.md`) |

A is a **ratio of verdicts**, not a MEASURE of whether the verdict function is a gate. Near either rail, the same number is consistent with a dead instrument and with a perfect one. Do not page on “A dropped” or celebrate “A = 99%” without the controls below.

## Control condition that makes A meaningful

A is publishable only when **both** controls were **completed looks** in the **same** window, on the **same** code path as production (`LEGACY-39.md`: do not sidecar).

```
canary-hit   must be ACCEPTED     # positive: issuer-declared good item
canary-miss  must be REJECTED     # negative: declared non-match / seeded fault

A_ok  iff  n_look ≥ 1
           AND look(canary-hit) completed AND accepted
           AND look(canary-miss) completed AND rejected
           AND n_unknown == 0
```

If either control look fails to complete → **UNKNOWN**, not a new A. If hit is rejected or miss is accepted → **instrument unhealthy**; do not publish A for production items. `control.hit` and `control.miss` are required on every window, same as NONE (`POSITIVE-CONTROL.md`). Do not cache a green control across process start.

Under `A_ok`, A describes the **non-control** bag. Still quote **(A, n_look, n_accept, n_reject, n_unknown)** as a bag, not a lone percentage.

Worked rails:

| Observed A | hit | miss | Meaning |
|---:|---|---|---|
| 0.00 | fail / reject | — | uninterpretable (or instrument dead) |
| 0.00 | accept | reject | estate produced no accepts; instrument still a gate |
| 0.99 | accept | not presented | uninterpretable |
| 0.99 | accept | reject | high accept among prod, and the verifier still refuses the miss |
| 1.00 | accept | accept | **unhealthy** — cannot reject |

## Seeded-fault technique

A **seeded fault** is a row the issuer planted whose correct verdict is **reject**, and that a **broken** verifier will **accept**. It is `canary-miss` with a body a prefix matcher, exists-gate, or rubber-stamp will swallow.

Plant (wrapper / issuer, not the verifier, `LEGACY-17.md`):

| Seed | Why a broken verifier accepts it |
|---|---|
| Valid `re` / filename prefix, digest ≠ `expected` | Prefix join, `exists(path)` (`TASK-04-MATCHER-TESTS.md`, `LEGACY-05.md`) |
| Empty or public template digest | Template-closes-many (`RECEIPT-MATCHING.md`) |
| `status=completed` and no file | Self-report (`TASK-10-SELF-REPORT.md`) |
| Same basename, out-of-repo path | Resolve-before-validate (`TASK-32-PATH-SAFETY.md`) |
| Wider `provider.class` than `seat.ceiling` | Name-keyed failover (`TASK-12-POLICY-PARITY.md`) |

Procedure, every window (or every gate invoke, same function):

1. Insert or archive-restore the seed in the **same store and partition** the verifier will query. A seed that is not in the index is not a test (`LEGACY-39.md`: miss must be a real row a prefix would steal).
2. Run the verifier. Do not strip seeds from `n_look` after the fact.
3. **Must reject.** Accept → instrument cannot reject; halt prod admits; do not “tune A.”
4. Eviction of the seed (cap wrap) → UNKNOWN, restore, then look — do not report last A (`TASK-09-RETENTION-RISK.md`).
5. Zero-acceptance on seeds: **one** accepted seed fails the verifier for the window (`CLASSIFIER-ZERO-ACCEPT.md`). Prod A is unpublished.

The seed proves **reject capability**. The hit proves **accept capability**. A needs both. Near-zero without a passed hit is “maybe refuse-all.” Near-total without a rejected seed is “maybe accept-all.”

**Rule:** A is uninterpretable at both rails unless `canary-hit` was accepted and a seeded fault was rejected on the same completed path. If the verifier cannot fail the seed, it is not a verifier.
