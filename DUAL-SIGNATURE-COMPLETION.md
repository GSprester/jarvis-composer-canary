# dual-signature completion

Worker `W` may write **payload bytes** at the declared path. It cannot write `COMPLETED` (`TASK-17-SEPARATION-OF-DUTIES.md`). Closure is two signatures over the **same content digest**, different authors, different files (`LEGACY-32.md`).

`expected = handoff.expected_sha256` is declared **before** `W` runs (`TASK-24-ARTIFACT-DECLARATION.md`). `expected ≠ sha256(b"")`. **Fail if `W` supplies `expected` after write.** **Fail if `COMPLETED` is `exists(path)`** (`LEGACY-05.md`).

## Second-signature design

**Sig1 (payload / issuer).** Bytes at the lexical declared path. `d = sha256(canonical(payload))`. Sig1 holds iff `d == expected`. Author of the bytes may be `W`. The **obligation** is the issuer’s digest, not `W`’s self-hash inside the file (`TASK-15-IDEMPOTENT-RECEIPT.md` — **fail:** hash-then-normalise). Do not append Sig2 onto the payload (`LEGACY-21.md` — **fail:** `d` moves).

**Sig2 (checker / MAC).** After MEASURE proves Sig1, checker `K` (wrapper key, `key_id` only in the row — **fail if the key is in the receipt**) appends a ledger object (`IDEMPOTENT-EVENT-LEDGER.md`):

```
{v:1, type:completion_sig2, unit, d, expected,
 key_id, pid, start,         # K incarnation ≠ W
 mac}                        # HMAC-SHA256(K, canonical(unit|expected|d))
```

Flush, then HWM, then `released` with `delivery.evidence_sha256=d`. **Fail if Sig2 is a filename `DONE` / `unit.ok`.** **Fail if Sig2 is `status=completed` written by `W`.** **Fail if `K` is `W`’s seat** (`LEGACY-17.md`). **Fail if MAC covers `path` or `name` only:** rename forges closure. **Fail if MAC covers `d` that `K` did not just hash** (copied from `W`’s record).

`P` burst-review (`BURST-REVIEW.md`) is the same Sig2 path: checker, not `P`’s ACK. **Fail if substitute `review_closed` is Sig2** (`SUBSTITUTE-REVIEW-QUEUE.md`).

`K` is MEASURE then one MUTATE (ledger append) (`GATE-DEADLOCK.md`). **Fail if `W`’s API can append `type=completion_sig2`.** Tombstone that symbol (`LEGACY-16.md`).

## What the reviewer checks

Reviewer `R` (third party, or `G.measure` — `DEPENDENCY-INVERSION.md`) recomputes, does not trust row prose (`EVIDENCE-TIERING.md` — only `deterministic` closes):

1. Handoff has `unit`, `path`, `expected` from **before** the run. **Fail if missing ⇒ treat as `W`’s digest.**
2. Path lexical-safe, under root (`TASK-32-PATH-SAFETY.md`).
3. `d' = sha256(file bytes)`; `d' == expected == row.d`. **Fail if you check `exists` or size.**
4. `receipt.re == unit` (R1). **Fail if `name[:8]`** (`RECEIPT-MATCHING.md`).
5. Recompute `mac` with `K`; `pid+start` is `K` and `same_process` or recorded as checker. **Fail if `W.pid` signs.** **Fail if MAC verify UNKNOWN ⇒ FOUND.**
6. Canary-hit/miss on this MEASURE (`LEGACY-39.md`). **Fail if skip controls to “close the burst.”**

Any miss ⇒ UNVERIFIED, not COMPLETED. ERROR on failed look (`POSITIVE-CONTROL.md`). **Fail if `R` writes Sig1** (gate authors content). `R` may only write Sig2 after (3) holds.

## Anti-forgery

**Property:** a false COMPLETED requires a **preimage** of `expected` (the issuer’s content) **and** a MAC under `K`. Creating a filename, touching `out/<unit>`, or writing `DONE:\n` yields neither.

| Forgery | Breaks |
|---|---|
| Empty / stub / date-prefix leftover | Sig1 (`d ≠ expected`) |
| `W` writes `d` into a sidecar and `status=completed` | Sig2 author / MAC |
| Rename `other` → declared path | Sig1 unless bytes are the preimage |
| Append note to payload after Sig1 | `d` changes; Sig2 MAC no longer matches |
| Copy `W`’s sha256 field into Sig2 without hashing | Reviewer step 3 fails |
| Fuzzy / model “looks done” | `model_guess`; cannot override (`EVIDENCE-TIERING.md`) |

**Fail if `expected` is guessable empty or a public template shared by 494 WOs:** one preimage closes many (`BLAST-RADIUS-ESTIMATE.md`). **Fail if `K` is a repo-committed secret.** **Fail if filename glob is treated as Sig2** (`PROVENANCE-ARCHIVE.md` presence gate).

`released` only after both sigs (`IDEMPOTENT-EVENT-LEDGER.md`). **Fail if promote ⇒ complete.**

**Rule:** Sig1 is the issuer’s bytes. Sig2 is the checker’s MAC over `(unit, expected, d)`. Filenames are not signatures.
