# receipt matching

Estate now: **622** work orders (WO), **568** receipts, **74** with `re`/filename quote exact, **494** receipts with no quote. **54** WOs have no receipt file at all. Join is **not** `len` (`TASK-18-COUNT-RECONCILIATION.md` — **fail:** 568≠622 looks “almost done”).

Identity is `receipt.re == wo.unit` (`TASK-04-MATCHER-TESTS.md`). Filename `YYYYMMDD-…` is display (`TASK-07-DATE-ROLLOVER.md`).

## Safe declared fields

Both sides must carry these **before** the seat runs (`TASK-24-ARTIFACT-DECLARATION.md`). Extra keys ignored (`TASK-33-SCHEMA-EVOLUTION.md`).

| Field | WO | Receipt | Unsafe substitute |
|---|---|---|---|
| `unit` / `re` | `unit` | `re` | `name`, `name[:8]`, topic, stem |
| `expected_sha256` | issuer digest of **payload** | `sha256` **citation** of that digest, or hash of file bytes | worker-supplied hash (`TASK-17-SEPARATION-OF-DUTIES.md`) |
| `path` | declared lexical path | same path after validate | any file that `exists` (`LEGACY-05.md`) |
| `enqueue_id` | ledger id | same if present | new UUID per retry (`TASK-27-RETRY-SEMANTICS.md`) |

**Fail if `expected_sha256` is empty-hash:** stub closes the WO (`LEGACY-32.md`). **Fail if path is absolute/`..`:** jail escape (`TASK-32-PATH-SAFETY.md`). **Fail if you declare fields by copying the receipt after write:** gate mints the join (`LEGACY-17.md`).

## Fallback ladder

One completed look. Canary-hit must be visible or the whole join is UNKNOWN (`LEGACY-33.md`, `LEGACY-39.md` — **fail if canary skip:** timeout `[]` marks 622 unmatched or 0). Each rung requires **cardinality 1** on both sides.

```
R1  receipt.re == wo.unit                         # the 74
R2  sha256(receipt_bytes) == wo.expected_sha256
    AND paths_equal(declared)                     # 494 may land here
R3  enqueue_id equal AND both non-empty
```

Stop at the first rung that yields exactly one WO and one receipt. **Fail if first-wins among many:** naive prefix matcher (`TASK-04-MATCHER-TESTS.md`) — two WOs one day, wrong pair. **Fail if you skip R1 when `re` is set but wrong:** a mis-quoted receipt binds by digest to a different WO. If `re` is set and ≠ `unit`, **do not** fall through — DISCARD that receipt (`LEGACY-19.md`).

R2 **fail if two WOs share a digest** (empty or template): ambiguous → neither matches. R3 **fail if `enqueue_id` missing treated as `""==""`:** 494×622 cartesian.

Do not use mtime, size, priority, or `scheduled_utc` as a join. **Fail:** two units same second, or clock skew (`TASK-19-TIMESTAMP-HAZARD.md`).

## UNMATCHED

After the ladder, bags (not lengths):

- `unmatched_wo`: 622 minus matched WOs (includes the 54 with no file, plus R2/R3 misses)
- `unmatched_receipt`: 568 minus matched receipts

Write each id into `unmatched/` as a row `{unit|receipt_id, reason, last_successful_scan}`. Status stays UNVERIFIED. **Fail if unmatched ⇒ COMPLETED / “nothing to verify”:** unverified as settled (`LEGACY-31.md`). **Fail if you delete unmatched receipts:** next scan looks like never issued (`STALE-CLAIM-RECOVERY.md`). **Fail if UNMATCHED is omitted when look fails:** dashboard `unmatched=0` (`TASK-34-METRIC-THAT-LIES.md`).

Re-run is the same `unit` / same receipt bytes. **Fail if you mint a new WO id to “help” the 494.**

## Fuzzy is banned

No Levenshtein, cosine, stem containment, or prefix. Those are **wrong completed looks** (`LEGACY-10.md`, `LEGACY-26.md`): deterministic pairs a third party cannot re-derive from declared fields. **Fail:** `20260910-alpha` binds `20260911-alpha-r0` and `20260910-alphabet`. **Fail:** 494 receipts collapse onto 74 popular stems. Two-factor done cannot cite a similarity score (`LEGACY-32.md`).

## Evidence per match

One object, append-only, not written onto the payload (`LEGACY-21.md` — **fail:** digest moves):

```
{wo.unit, receipt_id, receipt.re, rung: R1|R2|R3,
 wo.expected_sha256, receipt_sha256, path,
 enqueue_id or null, n_wo:1, n_receipt:1}
```

`n_*≠1` → do not emit a match. Checker must recompute both hashes. **Fail if evidence is “filename looked close.”**

**Rule:** Exact declared keys only. 74 by `re`, others by digest+path. Rest stay UNMATCHED. No guess.
