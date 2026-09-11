# LEGACY-21

A **hash-addressed** artifact is named by `sha256(canonical_bytes)` (declared before write, `TASK-24-ARTIFACT-DECLARATION.md`). **Immutable** means those bytes do not change; a later fact is a **new** object (or a new row in an append-only log, `TASK-33-SCHEMA-EVOLUTION.md`). **Appending context** (a note, seat, clock, summary, “healthy,” extra receipt line) to **that** blob is an in-place mutate of an identity. Failure modes:

## 1. The declaration is orphaned

Append changes bytes. `verify(path, declared.expected)` becomes UNVERIFIED even though the original work was good (`LEGACY-05.md`). Operators then “fix” it by rewriting `expected` to the new digest. That is the gate minting the evidence it accepts (`LEGACY-17.md`). The handoff no longer names what a third party was told to hash (`TASK-06-ARTIFACT-CHECK.md`).

## 2. Path and content split

The store is path-shaped (one file, one unit). After append, `exists(path)` is still true; the digest is not. Stale leftover and “updated context” are the same filesystem fact (`LEGACY-09.md`). Caches that kept the old hash 404 or fetch a different object. Gold-set citations and the live path disagree (`LEGACY-10.md`: deterministic path, inaccurate bytes).

## 3. Retries are no longer byte-identical

Context that TASK-15 forbids — wall-clock, host, pid, seat session, prose — leaks into the hashed body. Two writes of the same `(re, outcome)` differ. Idempotent receipt and content-hash dedupe both break (`TASK-15-IDEMPOTENT-RECEIPT.md`, `TASK-26-DEDUPE-BY-CONTENT.md`). Count of units inflates (`TASK-18-COUNT-RECONCILIATION.md`).

## 4. Self-hash after mutate

The file embeds `sha256` of an earlier body, then grows. Third-party hash of the file ≠ the embedded field (hash-before-normalise, hash-before-append). Or the writer re-hashes the whole file **including** the digest field and chases a moving envelope. The instrument is no longer a digest of the artifact.

## 5. Two objects, one citation

A true content store keeps the old hash (still fetchable) and mints a new hash for “bytes + context.” Readers that follow the **name** see the append; readers that follow the **declared hash** see the original. Two leads on one unit (`LEGACY-01.md`, `LEGACY-12.md`). SUPERSEDED without `successor_id` is this split (`LEGACY-15.md`).

## 6. Context is a judgment, not a byte

Appended summary, LKG, or “already analysed” is a weaker-model self-report (`LEGACY-20.md`, `LEGACY-06.md`). Peer analyzers that read it are captured (`LEGACY-19.md`). A failed look dressed as a trailing `[]` / healthy note is now **inside** the immutable object (`LEGACY-03.md`). DISCARD cannot un-mix payload from commentary.

## 7. Concurrent append loses a writer

Two seats append. Last rename wins. One context disappears; neither digest matches the declaration. Flush-then-HWM (`TASK-11-IDEMPOTENCE.md`) points at a moving blob. Drain that wrote “done” before the losing append looks COMPLETED and is not (`TASK-31-GRACEFUL-SHUTDOWN.md`).

**Rule:** Context is a **new** hash-addressed object (or an add-only log row that does not rewrite history). The original digest stays the completion check. If you had to append to make the artifact usable, you were editing identity.
