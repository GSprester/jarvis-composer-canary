# crash-loop brake

A **brake** stops `on_launch` / `released` for a `unit`. **By-design:** wrapper wrote `type=crash_loop` after ≥3 `blocked→interrupted` in 3600s UTC (`STALE-CLAIM-RECOVERY.md`). **By-accident:** a leftover hot receipt, `DONE` filename, `status=completed`, presence gate, or `released` without Sig2 (`DUAL-SIGNATURE-COMPLETION.md`, `PROVENANCE-ARCHIVE.md`) — dispatcher says “prior receipt, no claim” forever. One poison attempt looks like a circuit breaker.

**Fail if any hot file ⇒ brake:** first crash never retries. **Fail if you delete the file to “unbrake”:** `never_started` + silent re-fail (`LEGACY-03.md`).

## Accident vs design

| | Brake-by-design | Brake-by-accident |
|---|---|---|
| Row | `crash_loop` in ledger, `v≥1`, `unit`, `n≥3`, `window_s`, `pid+start` of the **wrapper** | No such row; or only `W`’s receipt / `exists` |
| Count | `blocked→interrupted` transitions, same `unit`, UTC window | mtime, “failed twice,” chat |
| `would_deny` | recomputed: `n>=3` | not computable |
| Sig1∧Sig2 | false (else it would be `done`) | often `exists` true, digest wrong |
| Author | wrapper, not model (`TASK-05-DOCTRINE-DRAFT.md`) | worker / cron / presence consumer |

`classify(unit)` (MEASURE, canaries — `LEGACY-39.md`):

```
if ledger look ERROR: UNKNOWN; do not launch          # fail: [] ⇒ open
if crash_loop row well-formed AND n>=3 AND window ok: DESIGN
elif hot_receipt AND not same(claim) AND not A∧B:     ACCIDENT
elif A∧B: done; not a brake
else: open
```

**Fail if `n` is “errors in the last hour”** (`SILENT-FAILURE-DETECTION.md`). **Fail if DESIGN is inferred from three `status=failed` by `W`:** self-report loop (`EVIDENCE-TIERING.md`). **Fail if ACCIDENT is called DESIGN to skip recovery.** **Fail if DESIGN is called ACCIDENT and you archive+retry now:** tight loop the guard was for. **Fail if classify uses `name[:8]`** (`RECEIPT-MATCHING.md`).

`list_next` shows `DESIGN` vs `ACCIDENT` as columns, same `order_key` (`LEGACY-35.md`). **Fail if both look like “blocked.”**

## Resume without silent re-fail

Silent re-fail = relaunch, same defect, same green or same brake, no new MEASURE. **Fail if resume is `touch` or mint a new `unit`** (`TASK-27-RETRY-SEMANTICS.md`).

**ACCIDENT resume** (`blocked→interrupted→open`):

1. `same` / lock: UNKNOWN ⇒ stop (`LIVENESS-PREDICATE.md`).
2. If A∧B: stop (it is `done`).
3. Archive payload+manifest; CAS unlink hot (`PROVENANCE-ARCHIVE.md`).
4. Backoff `min(30s×2^(n-1), 300s)` from **transition count**, not 0.
5. Same `unit`; new `enqueue_id` only after archive `released.kind=stale_archived` (`ORDERED-RELEASE-QUEUE.md` — **fail if new id prepends the lane**).
6. `on_launch` MEASURE canaries + `G.measure` if `C` (`DEPENDENCY-INVERSION.md`). ERROR ⇒ do not send.
7. Decrement quota only after later Sig1∧Sig2 (`BUDGET-AWARE-ROUTING.md`).

**Fail if step 3 is skip-archive.** **Fail if step 6 is skipped “we know why it died.”** **Fail if `C` still globs archive** (brake returns).

**DESIGN resume** (the real circuit breaker):

Do **not** `open` until a **human exemption** (`LEGACY-22.md`): `grantor`, `would_deny` (`n>=3` still true or window expired), `policy_sha256`, `expires_at_utc`. Wrapper writes it. Then reset the transition count **only after** the next A∧B, not after `W` says OK. **Fail if `P` ACK or model “looks fixed” clears DESIGN** (`BURST-REVIEW.md`). **Fail if exemption is “delete crash_loop row.”** **Fail if window expiry auto-opens without one MEASURE of the last archive digest** — same poison, `n` restarts at 0, silent re-fail.

After exemption: one attempt, same `unit`, same backoff floor (300s). If that attempt `blocked` again ⇒ DESIGN immediately (`n` not required to rebuild 3). **Fail if you require another 3** after a granted resume that dies: operators think the brake is off.

Predicate fix (matcher, empty-hash, hook deadlock) **before** exemption when the cheap decisive check fails (`BLAST-RADIUS-ESTIMATE.md`). **Fail if you exempt the whole predicate’s `R`.**

**Rule:** DESIGN is a wrapper `crash_loop` row. ACCIDENT is an orphan receipt. Archive+MEASURE+backoff for accident. Exemption+one shot for design. Never delete, never new `unit`, never skip the look.
