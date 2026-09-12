# positive control

Scanner contract: `FOUND` | `NONE` | `ERROR`. Today many implementations print nothing on “no hits” **and** on crash/timeout. Downstream `if not stdout: NONE`. That is empty-as-negative (`LEGACY-03.md`, `TASK-20-FAIL-LOUD.md`).

**Fail if stdout emptiness is the NONE signal:** SIGPIPE, `grep` no-match, timeout with no flush, and a completed zero-hit scan are one wire. Cron then closes WOs (`RECEIPT-MATCHING.md`, `LEGACY-31.md`).

## Output typing

One **object** on stdout (canonical JSON, one trailing newline), plus exit code. No bare text. Stderr is not the type (`TASK-15-IDEMPOTENT-RECEIPT.md`).

```
{"v":1,"kind":"FOUND"|"NONE"|"ERROR",
 "query_id":"…","target":"…",          # identity asked
 "hits":[…],                           # only when FOUND; may be empty never
 "control":{"hit":bool,"miss":bool},
 "error":null|{"class":"timeout"|"io"|"schema","detail":"…"}}
```

| `kind` | exit | `hits` | Meaning |
|---|---|---|---|
| `FOUND` | 0 | len≥1 | completed look, ≥1 declared hit |
| `NONE` | 0 | `[]` required | completed look, zero hits, **both controls passed** |
| `ERROR` | 2 | omitted | look did not complete |

**Fail if `FOUND` with `hits:[]`:** same bug renamed. **Fail if `NONE` exit 1:** callers treat 1 as grep-miss **or** as script error. **Fail if exit 0 + empty stdout:** parser invents NONE. **Fail if ERROR is `kind=NONE` + note in stderr:** notes are not a type (`LEGACY-25.md`). **Fail if you JSON-wrap after printing nothing on timeout:** torn/missing object → consumer NONE (`LEGACY-26.md`). Timeout must still emit `ERROR` or the wrapper writes it; **fail if wrapper mints NONE to keep cron green.**

Parse: missing `kind`, bad JSON, or no object after deadline → caller `ERROR`. **Fail if parse-fail ⇒ NONE.**

`hits` items cite `re`/`unit` + digest, not filename prefix (`TASK-04-MATCHER-TESTS.md`).

## Positive control (`canary-hit`)

Issuer-declared object that **must** be returned by this same matcher, store, and partition (`LEGACY-33.md`, `LEGACY-39.md`). Two-factor: `verify(path, expected)` and checker receipt (`LEGACY-32.md`). Fixed `re` (e.g. `canary-hit`), not `YYYYMMDD`. Not inserted by this scan (`LEGACY-17.md`).

```
if control look fails to complete: ERROR
if canary-hit not in result or verify fails: ERROR   # not NONE
```

**Fail if missing canary ⇒ NONE:** wrapped table / wrong UTC day looks like “no work” (`TASK-22-ROLLING-WINDOW-AUDIT.md`, `TASK-19-TIMESTAMP-HAZARD.md`). **Fail if the scanner `touch`es the canary so the look succeeds:** self-satisfied gate. **Fail if canary is `exists` only:** stub blesses every NONE. **Fail if canary lives in another index/region:** wrong-store bind (`LEGACY-23.md`). Evicted canary → ERROR, then restore from archive — **fail if you skip the scan and report last NONE** (`TASK-08-WATCHDOG-PATTERN.md` LKG).

`control.hit=true` is required on every `NONE` and `FOUND`. **Fail if you cache `hit=true` across process start:** resume lies (`TASK-05-DOCTRINE-DRAFT.md`).

## Negative control (`canary-miss`)

Declared object that **must not** match this query: different `re`, or digest ≠ `expected`, or class above ceiling. Same store.

```
if canary-miss appears in hits: ERROR   # predicate too wide; not FOUND
```

**Fail if no miss fixture:** prefix/fuzzy matcher returns FOUND on everything (`RECEIPT-MATCHING.md`). **Fail if miss match ⇒ NONE:** you hide a broken gate. **Fail if miss is a random string not in the store:** you never tested the matcher, only “unknown id.” Put a real row that a prefix would steal.

`control.miss=true` means “miss stayed out.” Required on `NONE`/`FOUND`.

## When NONE is legal

```
NONE  iff  object parsed AND kind=NONE
           AND look completed
           AND control.hit AND control.miss
           AND hits==[]
```

Anything else with empty stdout or failed controls is **ERROR**. ERROR blocks admit/dequeue (`LEGACY-38.md`). **Fail if ERROR ⇒ retry with a new `query_id` that looks like a new unit** (`TASK-27-RETRY-SEMANTICS.md`).

**Rule:** Empty bytes are not NONE. NONE is a typed, control-checked zero. No controls, no absence.
