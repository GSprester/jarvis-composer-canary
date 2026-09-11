# LEGACY-25

An error message is a **completed check of one predicate**, written by the process that failed that check. It is not a map of the estate. Reading it means listing what the string **does not say**, then refusing to fill those holes with the same noun (`LEGACY-06.md`, `LEGACY-10.md`). If the instrument did not finish, even the message you have is UNKNOWN (`LEGACY-07.md`).

## How to read (one pass)

1. **Name the ask.** Which URL, path, query, or lock did the caller hit? Identity is `(unit, endpoint, pid+start)`, not the error token (`TASK-04-MATCHER-TESTS.md`).
2. **Name the predicate that ran.** Authn on *this* host. `exists(path)`. `query` until timeout. That is all the text can claim.
3. **Write the predicates that did not run.** Other regions. Digest. Alive. “Search completed.” `min(seat, provider)`.
4. **Do not promote.** Do not rotate a key, close a unit, or steal a lock from a word the vendor reused (`LEGACY-17.md`). Missing a look is not a finding (`LEGACY-03.md`).

Silence, `[]`, and `exit 0` are error messages with the words stripped off (`LEGACY-04.md`). Read them the same way.

## Three real examples

### 1. `{"error":"invalid_api_key","status":401}`

**Said:** this regional URL rejected the Authorization header (`LEGACY-23.md`).

**Did not say:** the secret is revoked, malformed, or never issued. Did not say the other two declared endpoints were asked (`LEGACY-24.md`). Did not say `seat.ceiling` vs `provider.class`. Did not name `key_id` or `policy_sha256`.

A valid eu-west key on us-east produces this exact body. Treating it as `unknown_key` is a lying zero (`TASK-34-METRIC-THAT-LIES.md`). Classify `denied_here` until three completed rows exist.

### 2. `TimeoutError: store.query timed out` → caller returns `[]`

**Said:** this look did not finish (`TASK-20-FAIL-LOUD.md`).

**Did not say:** handoff `20260910-alpha` has no receipt. Did not say `re` was checked. Did not say unmatched, COMPLETED, or healthy. Did not say `last_successful_scan`.

`[]` after a timeout is the same wire as a completed empty scan (`LEGACY-03.md`). Downstream “no receipts” is a sentence the error never wrote. Raise `SearchFailed`; disposition UNKNOWN.

### 3. `FileNotFoundError: [Errno 2] No such file or directory: 'out/20260911-alpha'`

**Said:** that **path string** is not a file on this root.

**Did not say:** the unit never started. Did not say abandoned vs stale (`LEGACY-14.md`). Did not say the declared path+digest failed `verify` (`LEGACY-05.md`) — a leftover empty file would *not* raise and still be UNVERIFIED. Did not say the work is under yesterday’s UTC date (`TASK-19-TIMESTAMP-HAZARD.md`, `TASK-07-DATE-ROLLOVER.md`). Did not say the lock holder is dead.

“Missing file” is a candidate check, not a completion or a reclaim. Read the published lock and `verify(declared_path, declared_digest)` before you write `never_started` or rotate anything.

**Rule:** Believe the predicate that ran. Inventory the ones that did not. Those are not implied. They are UNVERIFIED.
