# Sparse-`re` receipt join

**Status:** design only.  
**Depends on:** `RECEIPT-MATCHING.md`, `TASK-04-MATCHER-TESTS.md`, `TASK-18-COUNT-RECONCILIATION.md`, `BACKLOG-TEMPLATE-CLASS.md`.  
**Does not exist in this tree:** matcher job, unmatched store.

Stated bag: **622** work orders, **568** receipts, **74** with `receipt.re` naming a WO, **494** with no `re`, **54** WOs with no receipt file. Only **~13%** of receipts reference the item they answer. A join that requires `re` and then **drops** the rest reports “74 matched, 0 leftover” and hides 494+54. A join that **guesses** the 494 (prefix, stem, first-wins) mints false pairs.

## Matching strategy

Partition **receipts** first. Do not run one fuzzy matcher on the whole bag.

```
R_quoted   = { r | r.re is set }
R_bare     = { r | r.re is absent }
W          = work orders keyed by unit
```

**R1 — quoted only.** Pair iff `r.re == w.unit` and cardinality 1. Wrong `re` → **DISCARD** that receipt (`LEGACY-19.md`). Do **not** fall through to digest so a mis-quote binds another WO.

**R2 — bare only, exact obligation.** Pair iff `sha256(canonical(receipt bytes)) == w.expected_sha256` **and** locators equal (`HANDOFF-RENAME-ID.md`: path is locator, `unit` unchanged). Cardinality must be 1 on **both** sides. Template-shared `expected` (`BACKLOG-TEMPLATE-CLASS.md` 43%) → `n_wo ≠ 1` → **no pair**.

**R3 — bare leftover, ledger id.** Pair iff `enqueue_id` equal and **both non-empty**. Missing `enqueue_id` is not `""==""`.

Stop at the first rung with `n_wo==1` and `n_receipt==1`. File order / mtime / `name[:8]` / Levenshtein are not rungs (`TASK-04`, `TASK-07`).

One completed look of both stores. Canary-hit (quoted) and canary-miss (prefix cousin that must **not** pair) on the same path (`POSITIVE-CONTROL.md`). Failed look → whole join **UNKNOWN**, not “74 is the answer.”

## False-positive risk

The 494 are the danger. They have no `re`. Every completed wrong pair is a **deterministic** lie (`LEGACY-10.md`): Sig1∧Sig2 can close the wrong WO.

| Strategy | How a false pair forms | Scale |
|---|---|---|
| Date-prefix / stem | Two WOs one UTC day; first-wins (`TASK-04` naive) | 494 collapse onto popular `YYYYMMDD` |
| Digest-only, no path | Byte-identical templates share `expected` | ~0.43 of the bag binds many↔many |
| Empty `expected` | Stub receipt closes every WO (`LEGACY-32.md`) | 622 false closes |
| `enqueue_id` both empty | Cartesian 494×622 | almost-all paired |
| Skip R1 on bad `re` | Mis-quote then R2 steals another digest | quoted receipt, wrong unit |
| `len(568)≈len(622)` | “almost done” | hides 54 missing + swaps (`TASK-18`) |

R2 without `n==1` is the template hole. R3 without non-empty ids is the cartesian hole. Prefix is banned even as “just for the 494.”

Canary-miss: a receipt whose `name[:8]` matches a WO and whose `re` is absent or wrong **must not** appear in `matched`. If it does, the join is always-pair (`ACCEPTANCE-RATE.md`).

False **negative** (a true pair left unmatched) is cheaper than a false **positive**: UNVERIFIED stays open; a false pair can `released` the wrong `unit` (`LEGACY-09.md` skip). Prefer no pair when `n≠1`.

## Fallback that does not drop rows

After R1–R3, **every** input id is in exactly one bag:

```
matched            = pairs with evidence row {unit, receipt_id, rung, n_wo:1, n_receipt:1}
unmatched_wo       = W minus matched WOs          # includes 54 with no file
unmatched_receipt  = (R_quoted ∪ R_bare) minus matched receipts
                      plus DISCARD (wrong re)
unknown            = whole join if any look failed
```

`reconcile(|W|+|R|, keys of the four bags)` must be all unchanged (`TASK-18`). Length of `matched` is not completeness.

| Forbidden drop | What it looks like | Required instead |
|---|---|---|
| Omit `R_bare` from output | “74 matched, unmatched=0” | 494 in `unmatched_receipt` reason=`no_re` / `r2_ambiguous` / `r3_absent` |
| Delete unmatched files | Next scan = never issued | Write `unmatched/` rows; keep bytes (`STALE-CLAIM-RECOVERY.md`) |
| Unmatched ⇒ COMPLETED / “nothing to verify” | Unverified as settled (`LEGACY-31.md`) | Status UNVERIFIED |
| Failed look ⇒ unmatched=0 | Dashboard green (`TASK-34`) | `kind=ERROR`, do not publish bags |
| Mint a new `unit` so a bare receipt can R1 | Second lead | Same `unit`; stay unmatched |
| Hide DISCARD | Bad `re` vanishes | `unmatched_receipt` reason=`re_mismatch` |

Re-run uses the same `unit` and the same receipt bytes. Do not rename receipts to inject `re` from a guess (`LEGACY-17.md`: gate mints the join).

`list_next` shows unmatched counts as first-class gauges, same UTC window as the join. Zero unmatched is publishable only when both canaries passed **and** `unknown==0` **and** bag reconcile holds.

## What this must not do

- Require `re` and treat the rest as noise.
- Prefix-join the 494 to “recover coverage.”
- First-wins on digest or day.
- Publish `matched` without the unmatched bags.

**Rule:** Quote `re` when set (wrong `re` discards, no fall-through). Bare receipts pair only on unique digest+path or unique non-empty `enqueue_id`. Everything else stays in `unmatched_*`. A silent drop is a completed look that omitted those ids.
