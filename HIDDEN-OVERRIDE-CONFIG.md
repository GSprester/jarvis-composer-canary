# Hidden-override configuration

**Status:** design only.  
**Depends on:** `TASK-29-CONFIG-PRECEDENCE.md`, `STALE-DOC-CLAIMS.md`, `LEGACY-22.md`, `LEGACY-21.md`, `OUTWARD-JOB-AUDIT.md`.  
**Does not exist in this tree:** resolver daemon, overlay store, effective snapshot.

A **base profile** `B` is in the tree (or any surface a PR reader can open). A later **override set** `H` is not: host env, unit `Environment=`, secrets manager, cloud console, uncommitted drop-in, exemption ledger on another host. A reader of `B` **cannot** see `H`. If consumers resolve by walking `B` then guessing `H`, effective is not checkable and the doc-stale flip is the default (`STALE-DOC-CLAIMS.md`).

The estate has **one** read surface: a published **effective** object `E`. Precedence below is the **fold** that writes `E`. After `E` is flushed, nothing reads `B` or `H` for a current value.

## Two surfaces

| Object | Who can see it | Tense | Role |
|---|---|---|---|
| `B` base profile | git / PR reader | `default` | Packaged / declared defaults. Not in force. |
| `H` override set | host / secret / other register | — | Inputs to the fold. Not a reader surface. |
| `E` effective snapshot | declared path in the estate (may be outside git; then it is an outward object, `OUTWARD-JOB-AUDIT.md`) | `current` | **The** value admit, workers, and current-claims use. |

`E` is **not** appended onto `B` (`LEGACY-21.md`). Identity is `sha256(canonical(E))` at a declared path. `B` may cite `E`’s path; it must not copy `E`’s values as if they were `B`.

## Precedence (fold only)

When materializing `E`, one key, one winner. Absent is `ABSENT`. `""` is present and wins (`TASK-29-CONFIG-PRECEDENCE.md`).

```
rung 0   process argv                 # this invocation only; if it should persist, it must land in H then E
rung 1   standing overlay receipts    # grant / exemption / declared drop-in **already in the overlay log**
rung 2   environment / unit Environment=
rung 3   host or cloud drop-in not in git
rung 4   base profile B (file)
rung 5   packaged default
```

`resolve(key)` = first present among 0…5. If all absent → **raise** (do not invent). Unknown `class` / unknown key still raises (`TASK-23-POLICY-INHERITANCE.md`).

**Hidden does not mean unlisted.** Every winning value from rungs 0–3 must leave a **receipt** in the overlay log before it is legal to write `E`:

```
{type:overlay_applied, selector, rung, hidden_id,   # register name, env key, grant_id — not the secret
 before, after,                                      # canonical values (or after_digest for secrets)
 grantor, pid, start, policy_sha256}
```

`hidden_id` names the **invisible source** so a third party can re-ask that register. It does not embed the secret. For secret values `E` stores `secret_ref` + `after_digest` of the bytes admit used, never the bytes.

A value in `H` that has **no** receipt is not a rung. The fold must not apply it. That is how an unseen set stays unseen **and** stays non-effective.

## One-place rule

```
effective(key)  =  E[key]     # after a completed look of E
```

Consumers (admit, queue `N`, matcher choice, current-tense claims, `in_force` probes) **only** read `E`. They do not call `resolve` against live env. They do not open `B` for a current number.

```
publish(E):
    look(H enumerators) completed     # timeout → UNKNOWN, do not publish
    fold B ← receipts ← live H
    write E.tmp; flush; rename        # TASK-11
    E.base_digest    = sha256(B)
    E.overlay_digest = sha256(overlay log [0,hwm))
    E.rung[key]      = winning rung
```

`fresh(E)` iff `E.base_digest` matches live `B` **and** `E.overlay_digest` matches the overlay HWM **and** every `E[key]` equals a replay of `resolve` from those two digests plus the receipts. If live `H` changed and `E` was not republished → `E` is **STALE**, not a license to read `H` directly (`STALE-DOC-CLAIMS.md`). Failed look of `H` or `E` → UNKNOWN; do not fall through to `B` (that presents the default as current).

Discoverable from one place means: a third party with `E` (and the overlay log it cites) can answer **what is in force** and **which rung won**, without opening the host env or the README.

## What the reader of `B` is told

`B` is `tense=default`. A current-claim whose `source` is `B` is **uncheckable** and must be refused (`STALE-DOC-CLAIMS.md`, `MECHANICAL-ACCEPT-SPLIT.md`). The only legal current-claim source for these keys is `E`.

PR review of `B` can say: “defaults are these bytes.” It cannot say the estate uses them. That is the same cut as a repo review of a crontab file vs the live register (`OUTWARD-JOB-AUDIT.md`).

## Controls

| Control | Hold |
|---|---|
| canary-hit | Plant `H` that matches `B` for key `k`; `E[k] == B[k]`, `rung` is 4 or 5 |
| canary-miss | Plant `H` that **flips** `k`; `E[k]` must be the flipped value and `rung ∈ {0,1,2,3}`. A reader that reports `B[k]` as effective **fails** |

Zero-acceptance: one consumer that used `B` or raw `H` as current fails the stratum.

## What this must not do

- Let admit read env “because E might be stale.”
- Merge `H` into `B` in place so git looks like production.
- Treat missing `E` as “use `B`.”
- Apply an env value with no overlay receipt.
- Put secrets in `E` or in `B`.
- Walk two files and pick `max` / last-write / prettier.

**Rule:** Fold argv > overlay receipts > env > host drop-in > `B` > packaged default into one flushed `E`. Hidden overrides win only as receipts. Every current look reads `E` only. The base profile is a default, not the estate.
