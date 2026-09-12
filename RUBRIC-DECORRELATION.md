# rubric decorrelation

A two-model verification layer (cheap classify, expensive verify; different families) whose **dominant error is the rubric**, not the weights. This checkout has no models. The design is why family diversity does not fix a definitional error, and the procedure that does: **independent rubric authoring plus a diff** (`TASK-12-POLICY-PARITY.md`, `LEGACY-20.md`).

```
label(item)  =  apply(rubric, item)
```

If `rubric` is wrong, `label` is wrong for every family that applies it. Overturn rate measures **disagreement under one definition**, not whether that definition matches the obligation.

## Why two families do not decorrelate a definitional error

Different families decorrelate **sampling noise** and some **stylistic bias** (verbosity, refusal shape). They do not decorrelate a **shared predicate**. The four-bucket backlog rubric (LIVE / SUPERSEDED / DUPLICATE / INFO) is that predicate. Both seats receive the same clause text, the same examples, the same “wrong SUPERSEDED is cheap” story.

| Shared object | What family diversity does | What it does not |
|---|---|---|
| Token sampling | Uncorrelated draws | Change what LIVE *means* |
| Architecture / training | Different blind spots on **wording** | Invent a missing survivor-id rule |
| Same rubric bytes | Echo | Detect that SUPERSEDED without `re` is a kill |

Worked definitional miss: SUPERSEDED = “a newer note exists.” No requirement that the note **name** the survivor (`receipt.re == handoff.name`, `TASK-04-MATCHER-TESTS.md`). Both families, given that sentence, will mark date-prefixed copies SUPERSEDED. Agreement is 100% on the bug. Model-B “final on disagreement” never sees a disagreement. The overturn metric stays green (`TASK-34-METRIC-THAT-LIES.md`).

Same-family echo was the stated reason for two vendors. That reason is true for **noise**. It is false for **definitions**. A second model on the same rubric is LEGACY-20 pre-process: the expensive seat is accurate about a self-authored world. The world was authored in the rubric file, not in the cheap model.

Byte-identical templates (`BACKLOG-TEMPLATE-CLASS.md`) make it worse: both families read the same bytes through the same clauses. Guaranteed agreement, zero information.

## Procedure: independent authoring plus a diff

The checkable object is the **rubric**, not the model vote. Two humans (or two wrapper-recorded grantors — not the models, `TASK-17-SEPARATION-OF-DUTIES.md`) write a rubric **without seeing each other**. Then a deterministic diff. Reconcile **before** either model runs.

### Author

Each author produces a canonical document. Identity is clause ids and predicates, not prose order (`TASK-15-IDEMPOTENT-RECEIPT.md`: hash after canonicalise).

```
rubric = {
  version,
  buckets: [LIVE, SUPERSEDED, DUPLICATE, INFO, UNKNOWN],
  clauses: [
    { id, bucket, predicate,   # machine-checkable when possible
      requires,                # fields that must be present (re, path, digest)
      forbids }                # e.g. filename[:8], exists(), model confidence
  ]
}
```

**Forbidden in an author’s copy:** the other author’s file, model outputs, a shared “draft we all liked.” Resume-as-cold-start (`TASK-05-DOCTRINE-DRAFT.md`): a second sitting does not inherit in-seat memory of the first rubric.

UNKNOWN is required. A four-class forced choice is itself a definitional error.

### Diff

Same shape as provider parity: symmetric difference of **clause ids and predicates**, not of wording.

```
diff(A, B) =
  only_A      = clauses in A not in B   (id + canonical predicate)
  only_B      = clauses in B not in A
  wording_only = same id+predicate, different prose   # not a conflict
  equal       = only_A = only_B = ∅
```

Missing `requires` (no survivor `re` on SUPERSEDED; no digest on TEMPLATE) is a **clause miss**, not wording. Two empty stubs must not report equal (`TASK-12`: missing class raises).

Worked miss the diff must catch:

| Author A | Author B | `only_*` |
|---|---|---|
| SUPERSEDED iff newer filename date | SUPERSEDED iff `re` names a live survivor | both clauses; date-prefix vs identity |
| INFO iff “no action words” | INFO iff no `re` and no declared path | predicate mismatch |
| no UNKNOWN | UNKNOWN if look fails | A missing UNKNOWN |

`wording_only` does not block. Predicate mismatch **blocks apply()**. Do not merge by “take the union of buckets and ship.” Union of contradictory SUPERSEDED rules is a third, unauthored rubric (the gate minting the predicate, `LEGACY-17.md`).

### Reconcile, then apply

1. Authors sit together **on the diff only**. Each conflict becomes a rewritten clause or a dropped clause. New canonical `rubric*` is flushed (write → flush → HWM, `TASK-11-IDEMPOTENCE.md`).
2. `policy_sha256 = sha256(canonical(rubric*))`. Both model prompts **cite that digest**. A prompt that omits it is a different rubric.
3. Models **apply** `rubric*`. They do not add buckets, do not weaken `requires`, do not write `tier=deterministic` (`EVIDENCE-TIERING.md`).
4. Overturn between families is interpreted **only after** `diff(A,B)` was empty (or reconciled). If you skipped the diff, overturn is not a check.

Holdout: keep a bag of items the **pre-reconcile** rubrics would have labelled differently. After `rubric*` ships, those items must be reviewed (`CLASSIFIER-ZERO-ACCEPT.md` on that bag). If both models agree with `rubric*` and a human MEASURE disagrees, the error is still the rubric — a second family will not save you. Fix the clause, do not add a third model.

## What this does not do

- It does not treat Model-B as final on a shared unread rubric.
- It does not let either model author clauses.
- It does not replace digest collapse of templates with a rubric vote.
- It does not use chat ACK as `rubric*` (`APPROVAL-TOKEN-TTL.md`).

**Rule:** Family diversity decorrelates noise. Rubric diversity (two independent texts, then a clause diff) is what decorrelates definition. Apply one reconciled rubric; never count agreement on an undiffed definition as verification.
