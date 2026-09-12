# TASK-02-POLICY-AUDIT

No standalone provider-policy loader, live `POLICY.json`, or configuration-loading module exists in this checkout. These paths are absent: `cloud_dispatch.py`, `external_provider_policy.py`, `scripts/handoff_lifecycle.py` (`scripts/` is absent). `TASK-13-FAILOVER-CEILING.md` states `admit` as proposal only; no `admit()` implementation is present.

Three markdown files contain extractable stdlib scripts that *are* policy/config contracts (not imported at runtime). Those fences are the only code reviewed below. `artifacts/history-provenance-goldset-20260910/validate.py` is a gold-set schema checker, not a provider-policy or config loader.

## Early return / raise table

Line numbers refer to the current files in this checkout.

| Location | Kind | Line | What happens |
|---|---|---|---|
| `TASK-12-POLICY-PARITY.md` `permissions_of` | early return | 35–38 | `entry.get("permissions") or []` then optional `class`; always returns a set, never raises |
| `TASK-12-POLICY-PARITY.md` `diff_providers` | early bind | 42 | `policy.get("providers") or {}` if `providers` missing/empty |
| `TASK-12-POLICY-PARITY.md` `diff_providers` | raise | 43–45 | `KeyError` if left or right provider name is absent |
| `TASK-12-POLICY-PARITY.md` `diff_providers` | return | 52–61 | report dict; no further validation of `class` values |
| `TASK-12-POLICY-PARITY.md` `main` | return | 71–72 | `--self-test` → `self_test()` |
| `TASK-12-POLICY-PARITY.md` `main` | return | 73–75 | missing `--policy`/`--left`/`--right` → exit 2 |
| `TASK-12-POLICY-PARITY.md` `main` | return | 78–80 | caught `KeyError` → exit 2 |
| `TASK-12-POLICY-PARITY.md` `main` | return | 81–82 | print report; exit 0 if `equal` else 1 |
| `TASK-23-POLICY-INHERITANCE.md` `rank` | raise | 40–41 | `PolicyError` on unknown class (including `""`) |
| `TASK-23-POLICY-INHERITANCE.md` `effective` | return | 46 | `NAME[min(rank(seat), rank(provider))]` after both `rank` calls |
| `TASK-23-POLICY-INHERITANCE.md` `failover` | raise | 50–51 | `PolicyError("missing provider")` if hop names absent |
| `TASK-23-POLICY-INHERITANCE.md` `failover` | return | 53 | `effective(seat_class, providers[to_provider])` |
| `TASK-23-POLICY-INHERITANCE.md` `main` | return | 92–93 | `--self-test` → `self_test()` |
| `TASK-23-POLICY-INHERITANCE.md` `main` | return | 94–95 | otherwise usage, exit 2 |
| `TASK-29-CONFIG-PRECEDENCE.md` `resolve` | return | 47–48 | present `arg` (including `""`) wins |
| `TASK-29-CONFIG-PRECEDENCE.md` `resolve` | return | 49–50 | `key in env` (including `""`) wins |
| `TASK-29-CONFIG-PRECEDENCE.md` `resolve` | return | 51–52 | `key in file_map` (including `""`) wins |
| `TASK-29-CONFIG-PRECEDENCE.md` `resolve` | return | 53–54 | present `default` returned |
| `TASK-29-CONFIG-PRECEDENCE.md` `resolve` | raise | 55 | `KeyError("unresolved: " + key)` if every rung absent |
| `TASK-29-CONFIG-PRECEDENCE.md` `main` | return | 113–114 | `--self-test` → `self_test()` |
| `TASK-29-CONFIG-PRECEDENCE.md` `main` | return | 115–116 | otherwise usage, exit 2 |

## FAIL-OPEN flags

**`TASK-12-POLICY-PARITY.md` `permissions_of` (lines 35–38):** missing `permissions` becomes `[]`; missing `class` is omitted. Neither raises. Two provider entries with both fields missing compare `equal` and `main` exits 0. That is “parity holds,” not an admit-allow, but a missing required field does not deny.

**`TASK-12-POLICY-PARITY.md` `diff_providers` (line 42):** missing `providers` becomes `{}`, then the name check raises. Not fail-open.

**`TASK-23-POLICY-INHERITANCE.md`:** unknown or missing class labels raise. Missing provider names raise. No path returns a wider class or a default allow. Not fail-open.

**`TASK-29-CONFIG-PRECEDENCE.md` `resolve` (lines 53–54):** if the caller passes `default` (self-test uses `"public"` for `seat_ceiling`), a missing arg/env/file key **returns that default** instead of raising. That is a permissive fallback when the key is a required policy field. `resolve(key)` with no default raises (line 55). Empty string on a winning rung does not fall through (not inherit-wider).

**`TASK-13-FAILOVER-CEILING.md`:** no code. Prose says missing/malformed `class` must deny; that predicate is not implemented here.

No other provider-policy or configuration-loading code was observed.
