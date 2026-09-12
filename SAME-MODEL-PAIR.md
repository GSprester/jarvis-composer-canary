# Same-model pair

**Status:** design only.  
**Depends on:** `RUBRIC-DECORRELATION.md` (shared predicate), `ACCEPTANCE-RATE.md` (rails), `BLACKBOX-MODEL-PROBE.md` (`M`, not `T`), `TASK-17-SEPARATION-OF-DUTIES.md`.  
**Does not exist in this tree:** ensemble runner, labeled bag, two hops.

Two **instances** of the **same** SKU (`M` equal, same rubric bytes). The question is not “is two cheaper than a second family?” It is whether the pair decorrelates **variance** or only repeats **bias**. Those are different residuals. A pair that only repeats bias is worthless as a check.

## What an instance is

An instance is one independent **draw** from the same issued hop:

- same `M` (transport identity, `MODEL-ROUTE-IDENTITY.md`)
- same prompt, same `policy_sha256`
- **new** context: no shared sample KV, no cached completion, no second read of the first body

If decoding is greedy (`temperature = 0` and the issuer is deterministic) the second call is the same draw. Variance is structurally zero. Do not run the pair. That is not a regime test; it is one instrument counted twice.

`T` (“we are two different models”) is discarded. Two seats that print different names and share `M` are still one SKU.

## Two residuals

On an issuer-labeled item `i`, write `e_X(i) = 1` iff instance `X` mismatches the **issuer** label (not the other instance).

| Residual | What is shared | What a second draw changes |
|---|---|---|
| **Variance** | Weights, rubric, `M` | The sample. `e_A(i)` and `e_B(i)` can split when the item is near a decision boundary. |
| **Bias** | Weights, rubric, `M`, training, refusal shape | Nothing that matters. `E[e(i) \| i]` is the same for both. They miss the same `i`. |

**Decorrelation of variance** = a second sample of the *same* estimator. Useful only if errors on the work you ship are mostly this residual.

**Decorrelation of bias** = a *different* estimator (other family after a rubric diff, or a deterministic MEASURE). Two instances of one SKU **cannot** do this. Family diversity already fails a shared predicate (`RUBRIC-DECORRELATION.md`). Same-`M` is that failure with the weights shared as well.

Do not call “they disagreed once” bias decorrelation. Disagreement is a variance event. Do not call “they agreed” verification. Agreement is the bias event you cannot see without a label.

## When the pair may be used

Only after the regime test below returns **variance**, on a bag of the **same class** as production, and only as a **disagreement gate**:

```
agree       →  not a close. One estimator spoke twice.
disagree    →  escalate (human, other family, or deterministic checker).
             Do not majority-vote n=2.
```

Allowed side uses in variance regime: mean of two **scores** when the issuer consumes a score, not a label. Still not a COMPLETED (`TASK-17`).

## When the pair is worthless

| Condition | Why |
|---|---|
| Bias regime on that class | Second draw repeats the miss. Agreement is the bug. |
| Greedy / cached / same session | One draw. |
| Undiffed or shared-wrong rubric | Definitional bias; both apply it (`RUBRIC-DECORRELATION.md`). |
| Agreement used as Sig2 / COMPLETED | Same author twice (`DUAL-SIGNATURE-COMPLETION.md`). |
| No issuer labels | Cannot tell bias from both-right (`ACCEPTANCE-RATE.md` A→1). |
| Pair used to “decorrelate” a ceiling or a digest check | Those are not stochastic. Run the checker once. |

Worthless means: do not spend the second call; it does not change the residual you care about. Escalate to a different estimator or a MEASURE, or do not claim a pair at all.

## Regime test

Required bag, **issuer-labeled**, same path as prod (`LEGACY-39.md`):

- **canary-hit** — declared accept
- **canary-miss** — seeded systematic fault a biased SKU will **accept** (wrong `SUPERSEDED`, prefix join, template digest — `ACCEPTANCE-RATE.md`)
- **boundary** (optional) — items that flip under temperature if the SKU is stochastic

`n` too small or `n_unknown > 0` → `UNKNOWN`, not a regime. Both instances must be completed looks. Mid-session `M` change aborts (`BLACKBOX-MODEL-PROBE.md`).

Run instance A, then B. Compute:

```
W_X     = { i : e_X(i) = 1 }
p       = |W_A| / n                         # marginal error of one draw
q       = |W_A ∩ W_B| / |W_A|               # P(B wrong | A wrong); if W_A=∅, see rails
J       = |W_A ∩ W_B| / |W_A ∪ W_B|         # Jaccard of error sets; 0 if both empty
d       = |{ i : A(i) ≠ B(i) }| / n         # disagreement, labels ignored
```

**Decision** (use `q` as the hold; `J` and `d` are diagnostics):

| Observation | Regime |
|---|---|
| `W_A` and `W_B` empty | **unidentified** — bag too easy; add misses. Not “variance.” |
| `q ≈ p` (independent: second miss is not predicted by the first) and `d` is on the errors | **variance** |
| `q ≫ p` (in particular `q → 1`) or `J → 1` with `W_*` nonempty | **bias** |
| canary-miss ∈ `W_A ∩ W_B` (both accept the seed) | **bias** (or always-yes). Fail the pair for this class. |
| canary-miss in exactly one of `W_A`, `W_B` | variance *on that seed*, or a flaky seed. Do not promote to “bias is gone.” Re-run; if it keeps splitting, you may use the pair as a disagreement gate **on that axis only**. |
| canary-hit rejected by both | instrument / hop, not a pair question (`ACCEPTANCE-RATE.md`). |
| `d = 0` and `W_*` nonempty | **bias** (or greedy). The pair never splits on the actual errors. |

`q ≈ p` is the statement that a second draw is a new sample, not a copy of the first residual. `q ≫ p` is the statement that knowing A missed `i` tells you B will miss `i` — shared bias.

Worked rails:

| `p` | `q` | `d` | Reading |
|---:|---:|---:|---|
| 0.20 | 0.22 | 0.30 | variance — pair may escalate on disagree |
| 0.20 | 0.95 | 0.02 | bias — they share the misses; pair is worthless |
| 0.00 | — | 0.00 | unidentified — no errors to overlap |
| 0.20 | 1.00 | 0.00 | bias or one draw counted twice |

Do not estimate `q` from disagreement with an unlabeled production stream. Unlabeled `d` mixes “both wrong the same way” (invisible) with “they split.” That is the A→1 hole.

## What this must not do

- Treat two greedy completions as a pair.
- Close work because two instances agreed.
- Majority-vote two labels and call it bias reduction.
- Skip the labeled miss and publish “low `d` ⇒ consistent.”
- Use a second instance to replace rubric authoring or a digest MEASURE.

**Rule:** Same-`M` decorrelates variance only. The test is `q = P(B wrong \| A wrong)` on an issuer-labeled bag that includes a seeded miss: `q ≈ p` is variance (disagreement gate only); `q ≫ p` or both accept the miss is bias (pair is worthless). Bias needs a different estimator, not a second draw.
