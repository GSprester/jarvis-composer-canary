# Auditable agent self-summary

**Status:** design only.  
**Depends on:** `TASK-10-SELF-REPORT.md`, `ADVERSARIAL-SELF-SUMMARY.md`, `EVIDENCE-TIERING.md`, `ARTIFACT-DERIVED-LEDGER.md`.  
**Does not exist in this tree:** summary linter, cite store, agent runtime.

An agent writing a summary of **its own** work is the author of the work and of the sentence (`TASK-10-SELF-REPORT.md`). The paragraph is `model_guess` (`EVIDENCE-TIERING.md`). It does not close units. It is **auditable** only when every **proper noun** and every **technical claim** traces to a **file** or a **command output** a third party can re-hash or re-run. An uncited name is a self-report.

This is not COMPLETED. COMPLETED is still `verify` / `derive` (`DUAL-SIGNATURE-COMPLETION.md`). The summary may **point**. It may not **settle**.

## Cite-or-omit

```
proper_noun   =  file path, unit/re, commit SHA, function/symbol, host,
                 image digest, PR/issue id, metric name with a number
technical_claim =  any verb about the estate: passed, failed, pushed,
                   matched N, exit 0, ready, latched, COMPLETED, “no errors”
```

```
cite(token)  ∈  { file(path, sha256), cmd(argv, cwd, exit, stdout_sha256) }
```

**Rule:** A publishable summary sentence may contain a proper noun or technical claim **iff** that token has a `cite` and a checker **this process** just reproduced:

```
ok(cite)  iff  look completed
               AND ( file: sha256(bytes at path) == cite.sha256
                     AND path lexical-safe (TASK-32)
                   OR cmd: re-run argv (CROSS-PLATFORM-LAUNCHER) in declared cwd
                           AND exit == cite.exit
                           AND sha256(canonical stdout) == cite.stdout_sha256 )
               AND n_unknown == 0
```

Uncited token → **omit** the sentence or mark `UNCITED`. Do not publish as current (`STALE-DOC-CLAIMS.md`). Do not let the agent invent `cite.sha256` from memory (`LEGACY-17.md`: gate must not mint the evidence). The wrapper records cites from **this** incarnation’s looks, then the agent may only **select** among those cite ids.

Chat, “as I recall,” another model’s summary, and `T` are not cites (`MODEL-ROUTE-IDENTITY.md`). `exists(path)` is not a file cite (`LEGACY-05.md`). A command the agent did not run in this session is not a cmd cite unless `K` re-runs it now.

## Shape

```
summary = {
  incarnation: {pid, start},          # this seat (TASK-16)
  sentences: [
    { text, tokens: [{span, cite_id}] }
  ],
  cites: {
    id: {kind:file, path, sha256} | {kind:cmd, argv, cwd, exit, stdout_sha256}
  }
}
```

`event_id = sha256(canonical(summary minus at_utc))`. The body must not include clocks in cited bytes (`TASK-15`).

| Token in the prose | Required cite |
|---|---|
| `BACKGROUND-LAUNCH-READY.md` | `file` of that path, current digest |
| `8e28cb7` / “I committed” | `cmd` `git rev-parse HEAD` / `git log -1` stdout digest, or `file` of `.git` object |
| “`--self-test` passed” | `cmd` of that argv, `exit==0`, stdout digest (not the word PASS in the summary alone) |
| “622 WOs, 74 with `re`” | `file` or `cmd` whose stdout **is** that bag; count via `TASK-18` reconcile, not a typed integer in prose |
| “tests passed” / “no errors” | `cmd` with controls (`POSITIVE-CONTROL.md`); zero without canaries is UNCITED |
| `COMPLETED` / “closed the unit” | **forbidden in the summary as a claim.** Point at `derive`/`verify` cite; the view still belongs to `K` |

A noun used only as a citation id (the path inside `cite`) does not need a second cite.

## Audit (third party)

`K` (≠ the agent) walks every `span` that the grammar tagged as noun or claim:

1. Token has `cite_id` else **UNCITED**.
2. `ok(cite)` else **BROKEN_CITE** (file moved, command not reproducible, exit drifted).
3. The **span text** is a substring of the cited file or of canonical stdout, or a declared alias (e.g. short SHA is prefix of `git rev-parse` output). Else **UNGROUNDED** (name not in the evidence).
4. Technical claims that are numbers: the number appears in the cited output **or** equals `len` of a reconciled bag. A number only in the sentence is UNGROUNDED (`TASK-18`: do not trust `len` in prose).

```
auditable  iff  zero UNCITED / BROKEN_CITE / UNGROUNDED
                AND incarnation same_process or recorded as this run
                AND canary-hit cite ok
                AND canary-miss: a sentence with a planted uncited noun is rejected
```

`auditable` is not COMPLETED. It is “the guess is grounded.” `list_next` must not skip work because a summary is auditable (`ADVERSARIAL-SELF-SUMMARY.md` §1).

Failed look of a cite → UNKNOWN, not “summary fine.”

## What this must not do

- Close a unit from a well-cited paragraph.
- Accept “see chat above” as `cite`.
- Let the agent hash a file it just wrote **as** the only proof it told the truth about a **command** it never ran.
- Grep the summary for `OK` (`ADVERSARIAL-SELF-SUMMARY.md` §6).

**Rule:** Cite-or-omit. Every proper noun and technical claim in an agent self-summary must name a `file(path, sha256)` or `cmd(argv, exit, stdout_sha256)` that **this** checker reproduces, and the span must appear in that evidence. Uncited is unpublished. The summary still cannot write COMPLETED.
