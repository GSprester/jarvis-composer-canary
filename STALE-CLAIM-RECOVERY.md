# stale-claim recovery

**Failure mode:** worker dies after a hot receipt and before `verify`. Dispatcher refuses relaunch: `prior attempt receipt exists without an active claim`. Same `unit`. New id = duplicate delivery (`TASK-27-RETRY-SEMANTICS.md`).

## Invariant

**A hot receipt is allowed only while a live claim names it, or after two-factor `verify` is COMPLETED.** Otherwise the receipt must sit in archive, not on the launch path.

Break it → either a dead attempt blocks forever, or two workers share one `unit` (`LEGACY-01.md`).

## States

`claim` = published lock `{unit, pid, start}` (`LEGACY-08.md`). `alive` = well-formed ∧ pid live ∧ `OS start(pid)==start` ∧ `start>=boot_at` (`LEGACY-14.md`). `receipt` = hot-path file/row the dispatcher keys on `unit` (not `name[:8]`).

| State | claim | hot receipt | verify |
|---|---|---|---|
| `open` | none / not alive | none | not COMPLETED |
| `claimed` / `running` | alive | optional in-progress | not COMPLETED |
| `blocked` | not alive | present | not COMPLETED |
| `done` | irrelevant | present or archived | COMPLETED |
| `interrupted` | not alive | archived | not COMPLETED |

`blocked` is the named failure. Recovery is `blocked → interrupted → open` (same `unit`), never `blocked → delete → open`.

```
on_launch(unit):
    if lock_read UNKNOWN: stop          # fail if you treat unread as stale: steal live
    if alive: refuse (already running)  # fail if you ignore alive: two leads
    if verify(path, expected)==COMPLETED: stop  # fail if you skip: second shell, two artifacts
    if hot_receipt:
        archive(receipt) then drop hot  # order matters
    publish claim (fsync, rename)
```

Sweep (`TASK-03-REBOOT-SWEEP.md`): `claimed|running` ∧ (`start<boot_at` ∨ missing start) → `interrupted`, then same archive+open. **Fail if sweep resumes in-place:** half-written payload becomes the next attempt’s leftover (`LEGACY-05.md`).

Unparseable lock = not alive, still archive receipt. **Fail if you leave it claimed:** empty-gap deadlock (`LEGACY-01.md`). **Fail if you unlink lock before archive:** crash mid-recovery deletes the only attempt record and looks like `never_started`.

## Crash-loop guard

Count `blocked→interrupted` transitions for this `unit` in UTC window `W=3600s`. If `count>=3`: do not `open`; escalate `crash_loop`; leave `interrupted`.

**Fail if no guard:** archive+relaunch storm fills the hot table, wrap evicts the canary (`TASK-22-ROLLING-WINDOW-AUDIT.md`, `LEGACY-33.md`). **Fail if guard keys mtime/age:** merely-old live drain counts as a crash (`LEGACY-14.md`). **Fail if guard keys a new `unit`:** loop resets, duplicates. **Fail if threshold is 1:** first SIGKILL never retries. Reset count only after COMPLETED verify, not after a green worker status (`TASK-17-SEPARATION-OF-DUTIES.md`).

Backoff `min(30s * 2^(n-1), 300s)` between the three attempts. **Fail if retry-now:** same poison payload, tight loop.

## Archive, do not delete

Archive = copy receipt bytes to `archive/<unit>/<start>-<sha256>` (canonical, `TASK-15-IDEMPOTENT-RECEIPT.md`), `verify` the copy digest, flush, **then** unlink hot. Row: `{unit, dead_pid, dead_start, receipt_sha256, reason=stale_claim}`.

Delete-only **fails as:** next `find` is `[]` = never tried (`LEGACY-03.md`); crash-loop and first-run are indistinguishable; a late writer re-creates the hot receipt after delete and blocks again; incident review has no prior attempt (`TASK-22-ROLLING-WINDOW-AUDIT.md`). In-place rewrite of the receipt **fails as:** hash identity moves, citations orphan (`LEGACY-21.md`). Archive-without-verify-then-unlink **fails as:** silent empty archive, same as delete.

Relaunch writes a **new** hot receipt bound to the **new** `pid+start`. Old archive rows are not a claim. **Fail if dispatcher treats any receipt (including archive) as blocking:** recovery never ends.

## Peek / list

`list_next` uses the same predicate as `on_launch` (`LEGACY-35.md`). **Fail if `ls` or “receipt exists” is the UI:** operator deletes the hot file, races the sweep, two leads.

**Rule:** Archive the stale receipt → same `unit` → new claim. Hot receipt ⇔ live claim or COMPLETED.
