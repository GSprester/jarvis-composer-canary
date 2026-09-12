# classifier zero-acceptance sample

Bound the error rate of an automated classifier when only **B** human reviews are affordable. Plan is **zero-acceptance** (c = 0): inspect **n** items in a stratum; **zero** human-confirmed errors ⇒ the bound holds at the stated consumer risk; **one** confirmed error ⇒ the **whole stratum fails**. This checkout has no classifier. The design is the contract (`BACKLOG-TEMPLATE-CLASS.md`, `TASK-10-SELF-REPORT.md`).

Human review is MEASURE against a checkable predicate (`re` identity, declared digest, path), not a second model. Failed look is UNKNOWN, not “no error” (`TASK-20-FAIL-LOUD.md`).

## Sample size formula

Lot (stratum) size **N_s**. Unacceptable error rate **p0** (LTPD): if the true rate is at least p0, the chance of seeing **zero** errors in the sample must be at most **β** (consumer risk). Acceptance number **c = 0**.

Infinite / large N_s (Bernoulli):

```
P(accept | p) = (1 − p)^n
n = ceil( ln(β) / ln(1 − p0) )
```

Rule of three (β = 0.05, small p0): **n ≈ 3 / p0**. Use the log form when p0 ≥ 0.05.

Finite N_s (sample without replacement). Let **K = max(1, ceil(p0 · N_s))**. Smallest n such that

```
C(N_s − K, n) / C(N_s, n)  ≤  β
```

First-order cap: `n_fpc = ceil( n_inf / (1 + (n_inf − 1) / N_s) )`, then verify the hypergeometric inequality. If N_s ≤ n, **census** the stratum.

Worked n (infinite, then FPC at N_s = 355 tail):

| p0 | β | n_inf | n at N_s = 355 |
|---:|---:|---:|---:|
| 0.01 | 0.05 | 299 | 163 |
| 0.05 | 0.05 | 59 | 51 |
| 0.05 | 0.10 | 45 | 40 |
| 0.10 | 0.05 | 29 | 27 |
| 0.20 | 0.05 | 14 | 14 |

Zero errors in n is **not** “0% error.” It is: we did not reject the claim “p < p0” at consumer risk β. Quote **(p0, β, n, c=0)**. A dashboard `errors=0` is a lying zero (`TASK-34-METRIC-THAT-LIES.md`).

## Fixed budget B

```
Σ_s n_s  ≤  B
```

If the n required for the (p0, β) you want to **publish** exceeds B on that stratum, you **cannot publish that bound**. Status is UNKNOWN. Do not shrink p0 after the reviews to make the formula fit (`LEGACY-17.md`: the gate must not author the evidence it accepts).

Allocation: spend reviews where a wrong label **changes action** (kill of live work, LIVE that pages a human). Do not spend c=0 samples on byte-identical template copies — those are one digest, one look (`BACKLOG-TEMPLATE-CLASS.md`).

Example: B = 60, β = 0.05.

| Choice | What you can claim |
|---|---|
| One kill stratum, p0 = 0.05, n = 59 | That stratum only; others UNKNOWN |
| Two strata, p0 = 0.10, n = 29 + 29 | Looser bound, two action classes |
| Four strata, p0 = 0.20, n = 14 × 4 = 56 | Wide bound; almost all B used |
| p0 = 0.05 on four strata (n = 236) | **Refuse the claim.** B is not enough |

## Strata

Build strata from **deterministic fields and the classifier’s emitted label**, not from model confidence.

After template collapse (templates are **not** a c=0 lot of copies):

| Stratum | Population | Error that matters |
|---|---|---|
| `LIVE` | Tail items the classifier marked LIVE | False LIVE (pages a human / forks work) |
| `SUPERSEDED` | Tail marked SUPERSEDED | False kill — live work buried |
| `DUPLICATE` | Tail marked DUPLICATE | Wrong sibling closed; unique constraint lost |
| `INFO` | Tail marked INFO | Blocker or decision labelled non-actionable |
| `UNLABELLED` / UNKNOWN | Tail the classifier skipped or errored | Silent leftover; do not treat as “no class” |

Optional cheap cross-cut **inside** a label, only if B remains after the table above: `has re` vs `no re`. Do not stratify on filename date (`TASK-07-DATE-ROLLOVER.md`) or on “high confidence.”

Draw a **simple random sample without replacement** inside each stratum. Do not sample the 43% template mass as if they were independent trials.

## Single-observation rule

**c = 0.** The first human-confirmed misclassification **fails the whole stratum**.

```
review(item):
    if look fails: UNKNOWN; do not count as pass
    if human predicate ≠ classifier label:
        STRATUM_FAIL
        stop trusting the classifier as a gate on this stratum
```

- One miss is enough. Do not “finish n to estimate p̂ and ship anyway.”
- Remaining items in that stratum stay UNVERIFIED. Action path is census, a deterministic rule, or a new classifier — not the failed model.
- A planted canary (known-wrong label) that the reviewer **agrees** with the classifier is instrument failure, not a pass (`LEGACY-39.md`).
- Zero misses after n ⇒ publish the bound (p0, β), not a green “validated.” The classifier may gate that stratum **only** at that bound.

**Rule:** n = ceil(ln(β)/ln(1−p0)) (or the hypergeometric twin). Strata are the action labels on the bespoke tail. One confirmed error fails the stratum. If B cannot buy that n, you do not have a bound.
