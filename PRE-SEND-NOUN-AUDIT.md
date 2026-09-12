# Pre-send noun audit

**Status:** design only.  
**Depends on:** `AUDITABLE-AGENT-SUMMARY.md`, `STALE-DOC-CLAIMS.md`, `MECHANICAL-RULE.md`, `TASK-27-RETRY-SEMANTICS.md`.  
**Does not exist in this tree:** outbound mailer, report builder, noun extractor.

Any **report** that will leave the seat (page, PR body, mail, dashboard publish, customer status) and that **names a system, a count, or a capability** must pass a noun audit **before** send. The author may be an agent, a cron banner, or a human draft. After send, the sentence becomes someone else’s **current** claim (`STALE-DOC-CLAIMS.md`). Cite-or-omit on a file that never ships does not bind the wire (`LESSONS-WRITE-ONLY.md`).

Send is the chokepoint (`MECHANICAL-RULE.md`). The effect this gate forbids is an outbound body that asserts a system / count / capability the estate cannot re-derive **this look**.

## Exact failure it prevents

**Shipping a grounded-looking report whose nouns are not in the evidence** — so the receiver treats them as the estate.

| Outbound sentence | What actually happened | Damage |
|---|---|---|
| “Provider B is a second account” | Shared 429 bucket (`SHARED-RATE-CEILING.md`) | Failover storm |
| “0 unmatched / 0 errors / N passed” | Failed look or partial bag (`ADVERSARIAL-SELF-SUMMARY.md`) | COMPLETED / skip |
| “1M context / tools / we are opus” | `T` (`BLACKBOX-MODEL-PROBE.md`) | Ceiling / pin lie |
| “freeze is on” / “N=32” | Hidden overlay (`HIDDEN-OVERRIDE-CONFIG.md`) | Stale current |
| “the job started” | `L` exit 0, `ready` false (`BACKGROUND-LAUNCH-READY.md`) | Cron green, nothing running |

That is one failure mode: **outward false current**. Not a wording nit. The receiver cannot see `D` vs `E` (`DECLARED-DERIVED-AUDIT.md`). They will hop, page, or close on the noun.

A report that names none of the three classes is out of scope (pure process chat). If a system, count, or capability appears, the whole body is in scope.

## Noun-audit procedure

Wrapper `K` (≠ the author) runs **before** `send`. Timeout → do not send (`TASK-20`). `K` does not mint cites (`LEGACY-17.md`).

### 1. Extract

Tag every span in the body (and subject/title) as:

| Class | What |
|---|---|
| `system` | hop, SKU, `key_id`, host, register, repo, `unit`, filename used as identity |
| `count` | integer or ratio about the estate (`N`, `A`, unmatched, errors, “74 of 568”) |
| `capability` | window, tools, modality, class, “can failover”, “in force”, “ready”, COMPLETED |

Extractor is a **declared** grammar / allow-list of patterns (numbers; `class:`; known provider names; `COMPLETED` / `ready` / `in force`). An LLM pass may **propose** spans; `K` must re-tag with the grammar. Proposed-only spans are still sent-risk: if the grammar misses a name, add it to the grammar (canary-miss: planted “helix-7” must tag `system`).

### 2. Bind

Each span needs a cite from **this** incarnation’s MEASURE bag (`AUDITABLE-AGENT-SUMMARY.md`):

```
cite  ∈  file(path, sha256) | cmd(argv, exit, stdout_sha256)
         | effective(E[key]) | derive(unit) | ready(unit) | in_force(P)
```

| Class | Extra bind |
|---|---|
| `system` | Transport `M` or register identity — not `T`, not README current (`MODEL-ROUTE-IDENTITY.md`, `OUTWARD-JOB-AUDIT.md`) |
| `count` | Bag reconcile (`TASK-18`): the number **equals** `len` of a cited bag or appears in cited stdout. `len(D)==len(E)` is not a count of matched |
| `capability` | Wire hold (`BLACKBOX-MODEL-PROBE.md` / `E` / `ready` / `in_force`) — not a model sentence, not `L` exit 0 |

`exists` is not a file cite. Chat is not a cite.

### 3. Verify (`ok` this look)

```
pass_span  iff  look(cite) completed
                AND ok(cite)                         # re-hash / re-run / effective
                AND span is in the cited bytes
                    or declared alias (short SHA)
                AND count: number == bag or stdout
                AND capability: MEASURE hold, not T
```

Any `UNCITED` / `BROKEN_CITE` / `UNGROUNDED` / UNKNOWN → **refuse send**.

### 4. Controls

- **canary-hit:** a report whose only system/count/capability spans are in the MEASURE bag → send allowed (still not COMPLETED of work).
- **canary-miss:** planted body “unmatched=0” with no bag cite, or “1M context” with only `T` → **must refuse**. If it sends, the gate is always-yes (`ACCEPTANCE-RATE.md`).

Zero-acceptance: one sent miss fails the outbound stratum.

### 5. Send

Only after flush of an audit row `{body_sha256, span_ids, cite_ids, pid, start}`. Same body bytes that were audited. Retry uses the same send token (`TASK-27`). Edit after audit → new `body_sha256` → audit again. `K` exit follows `PROCESS-BOUNDARY-EXIT.md`: refuse is parent 2, not 0 with “skipped send” in stderr only.

## What this must not do

- Audit a sidecar draft and send a different body.
- Let the author attach cites after `K` hashed the body.
- Treat a well-cited report as COMPLETED of the underlying units.
- Skip the audit because the author is human or “just Slack.”

**Rule:** Before send, every system, count, and capability span must `ok(cite)` on this look (`M`/`E`/bag/`ready`/`in_force` — not `T` or README). The failure this prevents is an outward false current: a receiver acting on a name the estate cannot re-derive.
