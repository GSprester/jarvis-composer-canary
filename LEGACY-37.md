# LEGACY-37

**“Silence is positive evidence of live work”** means: no log line, no heartbeat text, no event, therefore the seat is still doing the unit. The inference is tempting because this estate **designs** quiet paths. A silent watchdog prints nothing when a **completed** probe is healthy (`TASK-08-WATCHDOG-PATTERN.md`). A drain holds the lock and may not append (`TASK-31-GRACEFUL-SHUTDOWN.md`, `LEGACY-14.md`). Long jobs do not emit; events are hints (`LEGACY-18.md`). A noisy worker looks like a second lead. Operators then read **absence of speech** as **presence of a writer**.

That leap treats an empty channel as a negative finding about failure — without a positive control (`LEGACY-33.md`). Silence is one wire type for “healthy and quiet,” “never tried,” “dead,” “wrong region,” and “I did not look” (`LEGACY-03.md`, `LEGACY-30.md`). It does not name a predicate (`LEGACY-25.md`).

## When the inference is safe

Only as a **remainder** after a completed live check, not instead of one:

```
quiet_and_live  iff  published lock well-formed
                    AND OS pid live AND start(pid) == record.start
                    AND start >= boot_at
                    AND (optional) last completed probe was healthy
```

The seat may be silent. **Liveness is the OS**, not the missing line (`LEGACY-08.md`). Watchdog silence is health only if this tick’s probe **ran** and was healthy — not if the probe timed out and took the silent path, and not if the episode fingerprint already spoke (`LEGACY-04.md`). Listing “no alerts” in filename order is not this check (`LEGACY-35.md`).

## When it is unsafe

- **No incarnation.** Never-started or empty lock. Quiet because nothing published (`LEGACY-14.md`).
- **Stale incarnation.** Pid dead, `start` mismatch, `start < boot_at`. Quiet because the writer is gone. mtime can still look fresh.
- **Failed look mapped to quiet.** Timeout → `[]` → no due work → cron silent success (`LEGACY-03.md`, `LEGACY-04.md`). Canary not visible (`LEGACY-33.md`).
- **Unverified read as settled.** `status=completed` before fsync; checkers skip; the estate goes quiet (`LEGACY-31.md`).
- **Route never tried / wrong region.** No `route_intent`, or `denied_here` called `invalid_api_key` then silence (`LEGACY-23.md`, `LEGACY-30.md`).
- **Truncated then dropped.** Intent flushed, no result; consumer treats missing result as “still running” (`LEGACY-26.md`).
- **Self-as-sample.** Observer’s own quiet tick is counted as the worker (`LEGACY-13.md`).
- **Harness gone.** Successor not admit-ing; old binary tombstoned; no one to speak (`LEGACY-15.md`, `LEGACY-29.md`).
- **Starved low priority.** Queue lists a `p0` first; `p3` is silent because it has not been popped, not because it is computing (`LEGACY-34.md`).

Silence has **unbounded** meaning until you poll declared state. A heartbeat sentence the holder writes is not a substitute (`LEGACY-17.md`).

**Rule:** Silence is not a finding. Live work is `pid+start` (and a completed healthy probe, if you have one). Missing speech after a missing look is UNKNOWN, not progress.
