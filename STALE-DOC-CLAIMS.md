# Stale documentation claims

**Status:** design only.  
**Depends on:** `TASK-29-CONFIG-PRECEDENCE.md`, `LEGACY-22.md`, `LEGACY-15.md`, `SCHEDULER-POLICY-ENFORCE.md`, `LEGACY-27.md`.  
**Does not exist in this tree:** claim extractor, live resolver, doc probe.

A document that **describes the system** (README, in-tree policy prose, a design that states a live number) is stale when a claim it presents as **current** no longer equals the **effective** value. Stale is not “the markdown is old.” It is a failed MEASURE on a typed claim.

## Claim shape

Every sentence that asserts a live fact is a claim or it is not a fact. Uncheckable prose may exist; it must not use current tense as if it were bound.

```
claim = {
  id,
  selector,          # key the resolver knows (seat_ceiling, queue.N, matcher, E, …)
  expected,          # canonical bytes / enum / number
  source,            # where a third party looks (file path, register, ledger type)
  tense,             # current | default | historical
  successor_id?,     # required when tense = historical
  grant_id?          # if expected is an override, cite the row
}
```

`expected` is hashed after canonicalise (`TASK-15-IDEMPOTENT-RECEIPT.md`). Wording drift is not a miss. Predicate drift is.

## Checkable property (per claim)

```
effective(selector)  =  resolve(arg, env, file, default)     # TASK-29
                        then apply unexpired grant rows      # LEGACY-22
                        then apply standing exemption / freeze rows

fresh(c)  iff  look(c.source) completed
               AND n_unknown == 0
               AND (
                    c.tense = current
                    AND effective(c.selector) == c.expected
                    AND (no live override on c.selector
                         OR c.grant_id names that row
                         OR c.expected already is the overridden value)
                 OR c.tense = default
                    AND file_or_packaged(c.selector) == c.expected
                    AND the claim text does not say current / in force / we use
                 OR c.tense = historical
                    AND c.successor_id is set
                    AND c.expected is not used as a launch predicate
               )
```

`STALE` iff `c.tense = current` and the look completed and `effective ≠ expected`.  
`UNKNOWN` iff the look failed — not “doc is fine” (`TASK-20-FAIL-LOUD.md`).  
`uncheckable` iff `tense = current` and `form(selector, source)` fails (no mechanical look). Uncheckable current is **refuse**, same timing as an unverifiable unit (`MECHANICAL-ACCEPT-SPLIT.md`): do not publish the sentence as current.

Zero-acceptance (`CLASSIFIER-ZERO-ACCEPT.md`): one `STALE` current-claim fails the doc stratum. Do not average “most claims match.”

Controls on the probe (`ACCEPTANCE-RATE.md`):

| Control | Hold |
|---|---|
| canary-hit | A claim whose `expected` is planted equal to live effective → `fresh` |
| canary-miss | A claim whose `expected` is planted **unequal** to live effective → `STALE` |

If miss is `fresh`, the probe is always-yes (compares the doc to itself, or reads the file rung only).

## Failure mode: silent override flip

Precedence is argument > env > file > default. Grants and release overrides sit **above** the file the README was copied from. The doc almost always cites the **file or default** and marks it **current**.

```
t0  README claim C: seat_ceiling = restricted   tense=current
    file_map[seat_ceiling] = restricted
    effective = restricted
    fresh(C)

t1  later override (any one):
    argv / env SEAT_CEILING=internal
    OR exemption_grant would_deny=true, unexpired
    OR release_override / standing TTL-null token
    OR silent reset that drops the key so default wins (LEGACY-27)

t2  effective = internal (or public, or the grant’s hop)
    C still in the tree, still current, still “restricted”
    no claim row updated, no successor_id
```

At t2 the document **presents as current** a value the estate no longer uses. Reviewers, seats, and change-detectors that follow the README apply `restricted`. The run path applies `internal`. Both look deterministic. That is LEGACY-15 split-brain: two predicates, one of them a comment.

It is **silent** because:

- The override is a rung or a ledger row, not a PR on the README.
- `exists(README)` and mtime of the doc do not move (`LEGACY-05.md`, `LEGACY-14.md`).
- A probe that hashes **file** vs **claim.expected** still holds — it never asked `effective`.
- Operators call the env “just this once.” Empty env is set (`TASK-29`); a later empty string flips to `""`, not back to the file. The doc still shows the file.

Worked cousins: freeze in the README while a second crontab fires (`SCHEDULER-POLICY-ENFORCE.md`); `matcher = re` in the doc, prefix join still on the cron path; `N = 32` in TASK-25 while `QUEUE_N=128` is exported.

## Remediation

**Detect.** Every tick that cares about “what the docs say,” run `fresh` on all `tense=current` claims against **effective**, not against the file rung. Append `doc_probed` { `claim_id`, `fresh` | `STALE` | `UNKNOWN`, `effective_digest`, `expected_digest`, `rung` } to the same ledger. README is not that row.

**On writing an override** (wrapper, not the seat, `LEGACY-22.md`):

1. Recompute `would_deny` / new `effective`.
2. For each current claim on that `selector`, **one** of:
   - rewrite `expected` to the new effective and set `grant_id` / `rung`, flush, then hop; or
   - retone to `default` or `historical` with `successor_id` pointing at the grant / at a new current claim; or
   - refuse the override until the claim file is updated (gate does not mint the new sentence — issuer or grantor edits the claim, `LEGACY-17.md`).
3. Do not hop first and “update the README later.” That is t1→t2.

**On STALE already observed:** the live override stays (do not delete evidence). Relabel the claim: not current. Either bind `expected` to measured `effective` (cite the row) or tombstone with `successor_id`. Do not “fix” by writing the file so it matches the override **without** a grant receipt — that hides the rung (`LEGACY-27.md` intentional vs silent). Do not fail-open by treating STALE as wording.

**Retire the false current.** A current claim that cannot pass `fresh` must not remain on any path that admits, skips, or pages (`LEGACY-15.md`). Leave a tombstone. A second seat that still greps the old number is the same footgun as a retired matcher left `pip`-able.

## What this must not do

- Treat markdown mtime or “last reviewed” as freshness.
- Compare claim.expected only to the file default.
- Let the probe rewrite `expected` from `effective` in the same function that declares `fresh` (gate mints the doc).
- Publish uncheckable sentences as `tense=current`.
- Clear STALE by deleting the override row so the README looks right.

**Rule:** A current claim is fresh iff `effective(selector) == expected` after precedence and live grants. An override that flips effective while the doc still says current is STALE. Remediate by retone or rebind the claim **before or with** the override, never by hoping the README is still the estate.
