# LEGACY-13

An **observer** (sweep, watchdog, `ps` of writers) must **not count itself** as a member of the set it is measuring. The observer’s pid, its lock-read, its heartbeat row, and its log line are **instrumentation**, not estate work. Including them makes the scan a measurement of the scan (`LEGACY-10.md`: deterministic, not accurate).

## Why exclude

Alive-lead is `pid` + `start` (`LEGACY-08.md`). The sweeper has a pid too. If it opens the lock file, or creates a heartbeat “claimed” row so it can run, that incarnation is live — and it is **the observer**. A filter `pid != self.pid or start != self.start` is part of the predicate, same as `start >= boot_at`.

## Failure mode when it does not

**Self-as-holder: stale lead never released.** Sweep lists live writers. It includes itself. There is always ≥1 live pid. A dead seat’s lock is judged “someone is alive in this repo” or, worse, the sweeper’s pid is **compared to the lock’s pid** after a sloppy “any live process means do not steal.” Recovery never runs (`LEGACY-01.md`, `LEGACY-12.md` §2–3). The empty or dead lock stays. One-writer is a **dead** writer plus a **scanner that thinks the set is non-empty**. `open_units` / `live_writers` stay ≥1 (`TASK-34-METRIC-THAT-LIES.md`). The scheduled reclaim job is skipped (`LEGACY-09.md`: detector watches a set that always contains the detector).

**Self-as-target: the observer kills itself.** A “kill stray workers matching `writer`” pass matches the sweep command line (classic `ps | grep writer` hitting `grep`). The observer SIGTERMs its own pid, drain aborts, lock publish is the empty-gap (`TASK-31-GRACEFUL-SHUTDOWN.md`). Next boot: another sweep, same pattern.

**Self-as-sample: the scan writes what it counts.** Observer appends a tick row, then `COUNT(*)`. The count includes the tick. Zero can never mean idle; wrap evicts real work first (`TASK-22-ROLLING-WINDOW-AUDIT.md`).

**Rule:** Subtract `{self.pid, self.start}` (and the observer’s own unit ids / heartbeat types) **before** decide, alert, or kill. If the remainder is empty, that is a completed scan of **others** — not UNKNOWN, not “I am the lead.”
