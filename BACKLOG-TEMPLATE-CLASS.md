# backlog template classification

A backlog of work-item files in which **43%** of items are **byte-identical templates**. This checkout has no classifier. The design is the contract: do not send that mass to an LLM; collapse it with a digest; sample only the bespoke tail (`TASK-26-DEDUPE-BY-CONTENT.md`, `TASK-18-COUNT-RECONCILIATION.md`).

Let **N** = item count. **T = 0.43 N** share one or more exact template bodies. **U = 0.57 N** is the tail (each remaining digest, including whitespace-only near-misses — those are **not** templates).

Worked bags (stated estate, not a measurement here):

| N | Template items T | Tail items U |
|---:|---:|---:|
| 1,000 | 430 | 570 |
| 622 (WO bag, `RECEIPT-MATCHING.md`) | 267 | 355 |

## Why an LLM pass is the wrong tool on the 43%

The template mass is already a **deterministic fact**: `sha256(canonical(bytes))` is equal. A third party can re-run that. An LLM cannot (`TASK-10-SELF-REPORT.md`, `EVIDENCE-TIERING.md`: `model_guess`).

| What the LLM does | Why it is wrong |
|---|---|
| Re-reads 0.43 N copies | Token spend on a hash that was free |
| Splits one digest into LIVE vs INFO | Same bytes, different seats — echo, not a check |
| Collapses near-misses (“looks like the template”) | Whitespace / one-word drift is a **different** item (`TASK-18`: `"ok"` ≠ `"ok "`) |
| Trusts filename dates | `20260910-*` vs `20260911-*` look distinct; body is the same (`TASK-07-DATE-ROLLOVER.md`) |
| Two-model verify on templates | Same-family or not, the input is identical; agreement is guaranteed |

If the shared digest is an empty or public stub, an LLM “LIVE” on every copy is the template-closes-many failure (`LEGACY-32.md`, `RECEIPT-MATCHING.md` R2). If it is a real issuer template, one MEASURE of that digest classifies the cluster. A second model on copy 200 is theatre.

LLM (or a human) is for **unique** bodies, after collapse.

## Deterministic collapse

Hash **after** canonicalisation. Identity is content, not `id` / filename / mtime (`TASK-15-IDEMPOTENT-RECEIPT.md`, `TASK-26-DEDUPE-BY-CONTENT.md`).

```
canonical(item) =
  file bytes if the item is a raw note
  else JSON(body minus {id, name, path, mtime, granted_at_utc},
            sort_keys, tight separators, UTF-8)

h(item) = sha256(canonical(item))

clusters = group items by h
template_clusters = { h : |cluster(h)| ≥ 2 } ∪ { h : h in declared_template_digests }
survivor(h) = first item in cluster(h) by (unit or enqueue_id), never by name[:8]
```

Rules:

- Same `h` ⇒ one survivor + `count`. Label the cluster **TEMPLATE** (or DUPLICATE of `survivor.unit`). Do not LLM-vote the copies.
- Same `id`, different `h` ⇒ **do not** collapse (stale rewrite vs new body).
- `h == sha256(b"")` or a declared stub digest ⇒ cluster is not LIVE. UNVERIFIED / INFO. One checker look (`TASK-06-ARTIFACT-CHECK.md`).
- Two work orders that **declare** the same template `expected_sha256` stay **unmatched** to each other (`RECEIPT-MATCHING.md`: n≠1). Collapse is of **backlog notes**, not a join key for completion.
- Failed read of a file is UNKNOWN, not a unique tail item (`TASK-20-FAIL-LOUD.md`).

After the walk: **C** = |template_clusters| (usually ≪ T). Example: one repeated body ⇒ C = 1, 430 items become one row `count=430`. Several stock notes ⇒ C = number of distinct template hashes.

Census cost on the template mass is **C MEASURES**, not 0.43 N model calls.

## Sampling plan for the bespoke tail

The tail is the set of items whose `h` is unique (and not a declared template). Size **U = 0.57 N**. Each row needs a judgment. The expensive tool (human or frontier pass) runs **only here**, and not necessarily on all of U.

**Spend C first** (template representatives). Remaining budget **B** looks go to the tail.

### If the ask is a rate (fraction LIVE / SUPERSEDED / …)

Simple random sample of the tail, without replacement. For a 95% Wald interval of half-width **e** on a proportion, worst case p = 0.5:

```
n = min(U, ceil(1.96² × 0.25 / e²))
```

| e | n (infinite pop.) | n at U = 570 | n at U = 355 |
|---:|---:|---:|---:|
| 0.05 | 385 | 385 | **355 (census)** |
| 0.08 | 151 | 151 | 151 |
| 0.10 | 97 | 97 | 97 |

At N = 622, U = 355 < 385: **census the tail** if you need ±5 pp. Do not sample. At N = 1,000, U = 570: sample **385** for ±5 pp, or 151 for ±8 pp if B is tight.

Report the interval. A point estimate from n = 40 is UNKNOWN width, not a dashboard 43% (`TASK-34-METRIC-THAT-LIES.md`).

### If the ask is to action LIVE work

A rate sample is not a close-out. After the rate sample:

1. If `p̂_LIVE × U` is small and the CI is tight, **census the remainder** only if B allows; else queue by a **cheap deterministic stratum**, not by model confidence.
2. Strata (no LLM): has exact `re` / `unit`; body length; contains a declared path; digest equals a known stub (already removed). Oversample “has `re` + declared path” — that is where LIVE hides. Undersample unique INFO-shaped stubs only after a MEASURE that they have no `re` and no path.
3. Second model / human verify (`TASK-10`) on the **tail sample only**: all sampled LIVE, plus a holdout of sampled kills. Do not verify template copies. Overturn rate is conditioned on the tail, not on the 43% (`classify-then-verify` trap).

### What the plan forbids

- LLM on a template cluster “to be sure.”
- Fuzzy collapse of the tail into the template (prefix, stem, embedding).
- Treating `|T|/N = 0.43` as “43% done” after collapse. Collapse is not COMPLETED.
- A new `unit` per template copy (`TASK-27-RETRY-SEMANTICS.md`).

**Rule:** Same bytes, one digest, one look. The 43% is a hash join. Spend models on the 57% unique bodies, at a sample size derived from U and the error you will actually quote — or census when U is already smaller than that n.
