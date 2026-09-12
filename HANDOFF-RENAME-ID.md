# Handoff identity across rename

**Status:** design only.  
**Depends on:** `TASK-04-MATCHER-TESTS.md`, `RECEIPT-MATCHING.md`, `DUAL-SIGNATURE-COMPLETION.md`, `TASK-26-DEDUPE-BY-CONTENT.md`, `LEGACY-21.md`.  
**Does not exist in this tree:** handoff store, rename event, locator index.

A handoff **describes** a file (declared path + `expected_sha256`). The file’s **name** is a locator. If the handoff’s identity **is** that name, a rename orphans receipts, locks, and Sig2 (`DUAL-SIGNATURE-COMPLETION.md`: MAC on `path` forges or misses). The record must survive rename of the file it describes.

## Canonical identifier

```
unit  =  handoff.name     # opaque to the filesystem
```

`unit` is the only join key: `receipt.re == unit`, lock `writer-<unit>`, ledger `unit`, Sig2 MAC over `unit|expected|d`. It is **not** `path`, `basename`, `name[:8]`, inode, or mtime (`TASK-07-DATE-ROLLOVER.md`, `LEGACY-14.md`).

`path` is a **locator**, last-flushed, lexical-safe (`TASK-32-PATH-SAFETY.md`). Rename updates the locator. It does not mint a new `unit`.

```
handoff = {
  unit,                 # canonical id
  expected_sha256,      # payload obligation; ≠ sha256(b"")
  path,                 # current locator only
}
```

`exists(path)` is not identity (`LEGACY-05.md`). After rename, verify is `sha256(bytes at current locator) == expected`. Same `unit`, same `expected`. A leftover at the old path is not this handoff.

## Derivation

`unit` is assigned **once**, at admit, **before** the file exists (`TASK-24-ARTIFACT-DECLARATION.md`). It is derived from the **obligation**, not from the locator.

```
body = canonical({
  "job_id": job_id,
  "expected": expected_sha256,   # 64 hex, lower
  "seq": seq                     # uint, unique within job_id
})
# path, filename, clock, seat, pid  EXCLUDED
unit = sha256(body)              # 64 hex
```

`canonical` is JSON `sort_keys`, tight separators, UTF-8, one trailing newline (`TASK-15-IDEMPOTENT-RECEIPT.md`). Hash **after** that step.

| Included | Why |
|---|---|
| `expected` | The file the handoff describes is the payload digest, not the name |
| `job_id` | Same digest in two jobs is two units |
| `seq` | Two copies in one job (even byte-identical templates) stay distinct admits |

| Excluded | Why |
|---|---|
| `path` / basename | Rename would change `unit` |
| `YYYYMMDD` / clock | UTC rollover (`TASK-07`) |
| seat / pid | Resume is a new incarnation, same unit (`TASK-05`) |

Issuer may instead supply an opaque `unit` that is **not** a path stem. If supplied, it must not be recomputed from `path` later. Derived and supplied ids share the collision rules below.

Rename is an event, not a new derivation:

```
{type:handoff_renamed, unit, from_path, to_path, pid, start}
```

`unit` and `expected` unchanged. `to_path` lexical-validate **before** join. Two leads renaming at once: first flushed wins the locator; the other is a conflict, not a second `unit`.

## Collision behaviour

A **collision** is two admits that want the same `unit`, or one `unit` with two live locators / two `expected` values.

| Event | Behaviour |
|---|---|
| Second admit, **same** `unit`, **same** `expected`, new path | **Not** a new handoff. Treat as `handoff_renamed` or refuse if old locator still holds different bytes (conflict). Do not mint `unit'`. |
| Second admit, **same** `unit`, **different** `expected` | **Refuse.** Same id, different obligation (`TASK-26`: same id + different body do not collapse). Do not first-wins. Do not rewrite `expected` to the new file (`LEGACY-17.md`). |
| Two derives, same `(job_id, expected, seq)` | Same `unit` by construction — one admit. Replay is idempotent (`TASK-15`). |
| Two derives, same `expected`, **different** `seq` | Two units. Byte-identical templates collapse for **classification** (`BACKLOG-TEMPLATE-CLASS.md`), not for locks. |
| Accidental hash collision (distinct bodies, same `unit`) | **Refuse** the later admit. Do not alias. Operator assigns opaque `unit` or bumps `seq`. Birthday is treated as a store fault, not a merge. |
| Two live paths for one `unit` | **Conflict.** `job` not complete (`TWO-LANE-SPLIT.md`). Do not pick later mtime / prettier name. |
| Receipt `re == unit` after rename | **Match** (R1). Receipt filename may still carry the old date prefix. |
| Receipt `re` = old **path** or `name[:8]` | **Not** a match. DISCARD (`RECEIPT-MATCHING.md`). Do not fall through to R2 if `re` is set and wrong. |
| Sig2 / MAC covers `path` | **Invalid.** Rename would forge or orphan closure. MAC is `unit\|expected\|d` only. |

Ambiguous join (two handoffs, one receipt) → neither matches, ERROR, not first-wins (`TASK-04-MATCHER-TESTS.md`).

Canary: a handoff whose file is renamed after admit must still pair `receipt.re == unit` and `verify(new locator, expected)`. If the join follows the old basename, the identifier was the path.

## What this must not do

- Key the handoff on `path`, inode, or filename date.
- Re-derive `unit` after rename.
- Close on `exists(old_path)` or `exists(new_path)` without `expected`.
- Merge two `expected` values onto one `unit`.
- Treat template-same digest as the same `unit` without `seq`.

**Rule:** `unit = sha256(canonical(job_id, expected, seq))`. Path is a locator. Rename appends `handoff_renamed`; `unit` and `expected` stay. Same `unit` + different `expected` refuses. Two locators for one `unit` is a conflict, not a pick.
