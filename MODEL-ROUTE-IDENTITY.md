# model self-report vs misroute

A completion that **names a model** (“I am helix-7-large”) is a worker-written status (`TASK-10-SELF-REPORT.md`, `LEGACY-06.md`). It is also produced when the **route** answered from a different model than `admit` selected (`TASK-13-FAILOVER-CEILING.md`, `LEGACY-23.md`). This checkout has no provider client. The design is the **control** that separates **confabulation** from **misrouting**, and the **observable that is actually diagnostic**.

```
declared  =  (provider, model_id) bound at admit
T         =  model identity asserted in the completion text
M         =  model identity stamped by the transport / provider
```

`T` shares an author with the weights. `M` is written by the hop that served the bytes (HTTP body field, trailer, invoice line, wrapper-parsed vendor JSON — **not** a sentence the model invented). If `M` is absent, the diagnosis is **UNKNOWN**, not “it said the right name.”

## Why `T` cannot tell the two apart

| Estate | Typical `T` | What happened |
|---|---|---|
| Confabulation | Wrong or unstable name | Same declared hop; model does not know its SKU |
| Misroute | Name of the other SKU, or the name you wanted | Load balancer / failover / default model served `P'` |
| Prompted echo | Exactly `declared` | System prompt told it the name; hop may still be `P'` |
| Both | Anything | Text is not a bind |

Two families agreeing on `T` is rubric echo (`RUBRIC-DECORRELATION.md`). Asking “what model are you?” twice is not a control. `T == declared` is the same strength as `status=completed`.

## Diagnostic observable

**`M` versus `declared`**, on a **completed** look, same as `(key_id, endpoint)` vs `status != 200` (`LEGACY-24.md`).

```
confabulation  iff  look completed AND M == declared AND T ≠ declared
misroute       iff  look completed AND M ≠ declared
ambiguous      iff  M missing OR look UNKNOWN (timeout, unreadable JSON)
echo           iff  T == declared AND M ≠ declared     # text is a lie; hop is P'
bound          iff  M == declared                      # T is ignored for identity
```

`M` must come from a field the **model cannot write**: provider `model` in the non-completion envelope, signed usage row, or wrapper-recorded `provider` from the allow-listed URL that accepted the TLS session. Do not parse `T` into `M`. Do not log secrets (`LEGACY-24.md`).

`misroute` is a **bind** failure (wrong store), not “the model is confused.” Failover that lands on a wider class is still a ceiling event (`TASK-12-POLICY-PARITY.md`). `confabulation` does not authorize a hop.

If the vendor omits `M`, you may not invent it from style, latency, or token fingerprint unless that fingerprint was **declared** and collected on a pin (below). Undeclared “it feels like the small model” is `model_guess` (`EVIDENCE-TIERING.md`).

## Control experiment

Pin first. Then cross. Same prompt body, same `policy_sha256`, wrapper `pid+start` (`LEGACY-08.md`). Candidates are **declared** endpoints only — do not probe a wider class to “find the real model.”

**Prompt C** (identity bait): a fixed string that asks for the serving model id. Used only to produce `T`. Never used as `M`.

**Pin A** — declared hop `(P, model_id)`:
1. `admit(seat, P)` must pass.
2. Complete one call. Record `M_A`, `T_A`.
3. Expect `M_A == declared`. If not → **misroute on the pin** (estate already broken); stop. If look fails → UNKNOWN; do not classify `T_A`.

**Cross B** — lab-only, second declared hop `(P', model_id')` with `rank(P'.class) ≤ seat.ceiling`:
1. Admit separately. Complete the same Prompt C.
2. Record `M_B`, `T_B`.
3. Expect `M_B == model_id'`. If `M_B == declared` you did not actually cross.

**Read the table** (only after both looks completed):

| `M_A` | `T_A` | `M_B` | Diagnosis of `T` |
|---|---|---|---|
| declared | ≠ declared | P' | **Confabulation**: pin is bound; text still lies |
| declared | == declared | P' | Text matches pin; still ignore `T` (prompt echo possible) |
| P' | anything | — | **Misroute** on the route under test (pin failed) |
| declared | ≠ declared | P' and `T_B` == declared | Text tracks the **prompt**, not the hop — confabulation |
| declared | ≠ declared | P' and `T_B` == P' | Text **can** track the hop; then `T` is correlated, still not identity — `M` remains diagnostic |

The **diagnostic** cell is `M == declared` vs `M ≠ declared`. `T` is a decoy you collect so you can **show** it moves independently of the hop. If `T` were diagnostic, Cross B would be unnecessary and Pin A would suffice — it does not.

**Behavioral canary** (optional, when `M` is always present it is redundant): a prompt whose **declared** expected digest or exact token is known **only** for `model_id` on Pin A (issuer fixture, not model-authored). `verify` that output on Pin A. On the route under test, mismatch is **not** automatically misroute — the same model confabulates answers too. Use the canary only as a **pin health** check (`canary-hit` for the declared hop). Do not treat a wrong answer as `P'`.

Canary-miss: a second fixture that the declared model must **not** satisfy and `P'` would. If the route accepts the miss, that is evidence of **P'** only when `M` also says `P'` or when the miss is a **provider** behavior (HTTP 404 vs 200), not a generated string.

## What this does not do

- It does not trust “I am the frontier model.”
- It does not classify from latency or cost alone (those mix queue and SKU).
- It does not let the model write `M` into the ledger (`LEDGER-CLASSIFICATION-RECONCILE.md`).
- It does not rotate keys because `T` was wrong (`LEGACY-23.md`).

**Rule:** `T` is a self-report. `M` on a completed pin is the bind. Confabulation is `M` right and `T` wrong. Misroute is `M` ≠ `declared`. No `M`, no diagnosis.
