# LEGACY-19

An analyzer may be handed a **peer analysis** (another seat’s unmatched list, receipt bundle, LKG snapshot, “healthy” report). **Read** means ingest the conclusions into this decision. **DISCARD** means: do not merge those conclusions; treat the blob as non-evidence; run a **completed** scan of declared state yourself (`LEGACY-18.md`, `TASK-05-DOCTRINE-DRAFT.md`). Generous parse is fail-open (`LEGACY-07.md`). The gate must not become true because a peer said so (`LEGACY-17.md`).

Three **concrete** signals. Any one is enough. Do not “read anyway and discount.”

## 1. Bytes are not the declared artifact

The peer named a path and digest **before** this look (`TASK-24-ARTIFACT-DECLARATION.md`). DISCARD if any of:

- path was never declared, is absolute / `..` / reserved (`TASK-32-PATH-SAFETY.md`), or sits outside the root (`TASK-06-ARTIFACT-CHECK.md`)
- file missing, empty, or unreadable (existence is not a report, `LEGACY-05.md`)
- `sha256(canonical_bytes) != declared.expected` (hash-before-normalise is the same miss, `TASK-15-IDEMPOTENT-RECEIPT.md`)

You have not read an analysis. You have a path-shaped rumour. Parsing JSON from the wrong digest imports a **deterministic lie** (`LEGACY-10.md`).

## 2. The author is not a live third party for this unit

Identity is `re` / `unit` / `pid+start`, not a filename prefix (`TASK-07-DATE-ROLLOVER.md`). DISCARD if any of:

- `analysis.unit` (or `receipt.re`) ≠ the handoff this analyzer is deciding — including a date-prefix “match”
- published lock missing, empty, unparseable, or **not alive**: dead pid, `OS start(pid) ≠ record.start`, or `start < boot_at` (`LEGACY-08.md`, `LEGACY-14.md`)
- author seat **is the subject** (self-report, `LEGACY-06.md`, `TASK-10-SELF-REPORT.md`) or **is this observer** (`LEGACY-13.md`)

A dead, stolen, or self-authored file is a **record**, not a peer check. Reading it lets a second lead or the job under test close its own unit (`LEGACY-01.md`, `LEGACY-12.md`).

## 3. A failed look is dressed as a result

Empty and “found nothing” are different types (`LEGACY-03.md`). DISCARD if any of:

- payload is `[]` / `unmatched=0` / `healthy` **and** there is no `last_successful_scan` UTC from a completed query
- probe kind is UNKNOWN, timeout, or raise, wrapped as a zero or as LKG (`TASK-08-WATCHDOG-PATTERN.md`, `TASK-34-METRIC-THAT-LIES.md`)
- required keys absent (`v`, `unit`, `pid`, `start`, declared digest) while a wrapper bit says success (`TASK-33-SCHEMA-EVOLUTION.md`)

Reading that blob as “the peer already looked” is the dashboard green on a dead store (`LEGACY-04.md`). The search never ran. This analyzer must look, or stay UNVERIFIED.

**Rule:** DISCARD is the fail-closed verdict on the **instrument**, before the contents can become the estate. If the peer cannot be shown to be the declared bytes, a live other holder, and a completed scan, do not read it.
