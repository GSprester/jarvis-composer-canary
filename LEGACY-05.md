# LEGACY-05

**Against.** “The output file exists” is not proof that work completed. Existence is a **necessary** check for a path-shaped artifact; it is not **sufficient**. Completion is a third-party check that the **declared** path contains the **declared** bytes (`TASK-05-DOCTRINE-DRAFT.md`, `TASK-24-ARTIFACT-DECLARATION.md`).

## Why existence lies

A file can be on disk for reasons that are not this run of this unit:

- **Empty leftover.** Create succeeded; write did not. Size 0. The path exists. The digest is the empty hash, not the handoff digest (`TASK-06-ARTIFACT-CHECK.md`).
- **Stale previous run.** Same path, last week’s payload. Existence is exactly what a retry is supposed to **replace**, not celebrate.
- **Wrong bytes.** Worker wrote a preview, a `DONE:` line, or a date-prefixed receipt that does not match `re` (`TASK-07-DATE-ROLLOVER.md`, `TASK-17-SEPARATION-OF-DUTIES.md`). The file is real. The obligation is not closed.
- **Worker-authored “success.”** Touching the path and setting `status=completed` is one author (`TASK-10-SELF-REPORT.md`). A third party who only asks `exists?` has been captured by that author.

If existence were enough, declare-then-verify would stop after `path.is_file()`. The tests in TASK-24 exist because that stop is wrong: empty and stale both exist and both stay UNVERIFIED.

## What would have to be true

Existence would suffice only if the filesystem guaranteed that a name appears **if and only if** a single, atomic, content-addressed publish of the declared digest occurred — no empty create, no leftover, no second writer (`LEGACY-01.md`). POSIX create-then-write does not give that. Rename-of-complete-bytes plus an independent `sha256` check does.

## The side, in one rule

**Treat existence as “there is a candidate.” Treat `sha256(file) == declared` (after lexical path safety, `TASK-32-PATH-SAFETY.md`) as completion.** Anything less is a completion **record**, not a completion **check**. Schedulers that green on `os.path.exists` are LEGACY-04 case 4: success while the work is UNVERIFIED.

That is the same invariant as the gold-set COMPLETED label: every obligation has **exact cited evidence**, not a path that merely appeared.
