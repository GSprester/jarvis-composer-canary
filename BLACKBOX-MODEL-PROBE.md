# Black-box model probe

**Status:** design only.  
**Depends on:** `MODEL-ROUTE-IDENTITY.md` (`M` vs `T`), `TASK-13-FAILOVER-CEILING.md` (class ranks), `ACCEPTANCE-RATE.md` (near-0 / near-1).  
**Does not exist in this tree:** live hop, probe runner, fixture corpus.

A hop is a black box. The wrapper may send bytes and read status, headers, and body. It does not trust any string the model writes about itself. Identity and capacity are **what the wire does**, not what the completion claims.

## What is discarded

`T` is the completion text answering "what model are you?" or echoing a system pin. Discard it for identity and for capacity. A completion that says "I am opus / 1M context / I cannot use tools" is not evidence.

If `T` is the only signal, the hop is **unidentified**. Do not pin from `T`. Do not raise a ceiling from `T`.

## What is measured

Two independent quantities. Do not collapse them into one label.

| Quantity | Meaning | Not |
|---|---|---|
| **Identity** | Which issued SKU (or which issuer + class) is on the other end of this hop | A marketing name in `T` |
| **Capacity** | Measured maxima this hop actually serves (input tokens to first hard reject, tools, modalities, output cap) | A number the model recites |

Identity may be **unknown** while capacity is still measured. Capacity may be **bounded below** a declared SKU even when identity matches `M`.

## Controls (required)

Every probe session runs three classes of request against the **same hop**, same auth, same `M` pin if any. Order: positive, then negative, then ladder. Abort the session if the hop's `M` (or equivalent transport identity) changes mid-session — that is a different instrument.

### Positive control

A request the **declared** SKU must accept: short, in-distribution, no tools unless the declared SKU advertises tools and the admit allows them.

| Wire | Reading |
|---|---|
| 2xx + completion | Hop is alive. Session may continue. |
| 4xx / 5xx / timeout | Hop is down or not the declared SKU. **Do not** interpret later misses. Identity = unknown. Capacity = unmeasured. |

A positive-control miss is not a capacity number. It is a dead or swapped hop.

### Negative control (the hold)

A request the **declared** SKU must **refuse at the wire** (or return a documented issuer error class), which a **wider** SKU on the same issuer would accept.

Pick **one** axis the declared SKU is specified to lack, and that the next rank up is specified to have:

| Axis | Negative request | Hold (declared SKU) | Miss (not the declared SKU, or always-yes) |
|---|---|---|---|
| Context | Input length `N_declared + δ` (just over the published window) | 4xx / issuer `context_length_exceeded` (or equivalent) | 2xx + completion |
| Tools | A tool the declared SKU does not serve | 4xx / `tool_not_allowed` | 2xx that **invokes** the tool (not a textual refusal) |
| Modality | Image / audio / video part the declared SKU does not serve | 4xx / `unsupported_modality` | 2xx that **consumes** the part |
| Class | A request that only a higher `TASK-13` rank is admitted to serve, if the issuer encodes class in the error | Documented class error | 2xx as if the higher rank |

**Hold** = the hop behaved as the declared SKU's **lack**.  
**Miss** = the hop served what the declared SKU must not serve.

A miss on the negative control means: this hop is **not** the declared SKU, **or** the probe is an always-yes instrument (see `ACCEPTANCE-RATE.md`). Either way, **do not pin** the declared name. Do not treat measured capacity as that SKU's capacity.

Textual refusal inside a 2xx body ("I cannot process this image") is **not** a hold. The model can say no while the endpoint accepted the modality. Only the **wire** refusal counts.

If the issuer has no documented wire refusal for that axis, **do not use that axis** as a negative control. An undocumented 4xx is `UNKNOWN`, not a hold.

### Ladder (capacity)

Only after both controls: binary-search (or fixed rungs) on **one** axis at a time until the first hard reject.

- Record `(accepted_max, first_reject)` . Capacity is the closed interval between them, not a point the model stated.
- Do not start the ladder above the declared window to "see what happens." That is a probe of a wider class (`MODEL-ROUTE-IDENTITY.md`). The negative control already asked the one over-window question.
- Timeout / empty body on a ladder rung = `UNKNOWN` for that rung. Do not infer a window from silence (`TASK-08`).

## Identity without `T`

Prefer, in order:

1. **Transport `M`** — response header / body field the **issuer** sets, compared to the admit pin. If `M` is present and ≠ pin → `route_model_mismatch`. If `M` is absent → do not invent it from `T`.
2. **Negative-control hold + positive-control hold** — consistent with the declared SKU's published lacks and haves. This is **class consistency**, not a unique SKU id. Two SKUs that share the same window and tools are indistinguishable on these probes.
3. **Issuer error taxonomy** — stable error codes / shapes that only that issuer emits. Use as issuer identity, not SKU identity, unless the code names the SKU.

Do not build a "which model" classifier from completion style. Style is `T`.

If (1) is missing and (2) only constrains a class, publish **`identity = class:C`** or **`identity = unknown`**, never a guessed SKU string.

## Session verdict

| Positive | Negative | Ladder | Verdict |
|---|---|---|---|
| miss | — | — | `hop_dead_or_swapped`. No pin. No capacity. |
| hold | miss | — | `not_declared_sku` (or always-yes). No pin to the declared name. |
| hold | hold | measured | `consistent_with_declared` + `capacity = [accepted_max, first_reject)`. Pin only if `M` also matches (or `M` absent and policy allows class pin). |
| hold | `UNKNOWN` (no documented wire refuse) | measured | Capacity may be recorded. Identity stays `unknown` or `M`-only. Do not treat the missing negative as a hold. |

`consistent_with_declared` is **not** "we proved it is that SKU." It is "nothing the wire did contradicted the declaration." Unique identity still requires `M` or an issuer field that names the SKU.

## What this must not do

- Trust `T` when `M` and the negative control disagree with it.
- Treat a polite 2xx refusal as a negative-control hold.
- Probe a wider class than the admit to discover which SKU is behind the hop.
- Publish a SKU name from ladder height alone (a short window does not prove the small SKU if the hop is throttling).
- Run the negative control against a different hop than the positive control.

## Test vector (no hop required)

A recorded pair is enough:

1. Positive: 2xx on a 16-token completion request.
2. Negative: input length = declared window + 1 token → issuer `context_length_exceeded`.
3. Same pair with the negative returning 2xx → session must verdict `not_declared_sku`, must not pin.

If a fixture of (3) pins the declared name, the probe is theater.
