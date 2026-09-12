# silent failure detection

Monitor `M` prints **“no errors in the last hour.”** That sentence can be **true of the counter** and **false of the estate**. `errors` is often `COUNT(*)` of rows `kind=ERROR` in a hot window. Zero is a length shortcut (`TASK-18-COUNT-RECONCILIATION.md`, `TASK-34-METRIC-THAT-LIES.md`).

**Fail if `errors==0` ⇒ healthy.** **Fail if `M` uses local “last hour”** (`TASK-19-TIMESTAMP-HAZARD.md`). Window is `[now_utc-3600, now_utc]`.

## How it is true while broken

1. **No look.** Probe timeout / raise / never scheduled → no ERROR row. Empty stdout stored as NONE (`POSITIVE-CONTROL.md`, `LEGACY-30.md` never-tried). Clock still advances; counter stays 0.
2. **ERROR mapped to success.** `[]`, `exists`, prefix match, `status=completed`, `released` without delivery (`LEGACY-31.md`, `EVIDENCE-TIERING.md`). The defect emits **confident success**, not ERROR. `M` is correct: zero ERROR rows.
3. **Fingerprint already spoke.** Silent watchdog: one alert per episode; repeats are quiet (`TASK-08-WATCHDOG-PATTERN.md`). Hour contains the silence, not the first line.
4. **Wrap / rotate.** Hot table or `app.log` dropped the ERROR (`TASK-22-ROLLING-WINDOW-AUDIT.md`, `TASK-30-LOG-ROTATION.md`). Archive has it; `M` reads hot only.
5. **Filter.** `M` counts `level=error` or HTTP 5xx. UNKNOWN, 402, `denied_here`, canary-miss-as-FOUND, `unhealthy` env probe are excluded (`LEGACY-23.md`, `LEGACY-38.md`).
6. **Wrong partition.** Errors keyed `YYYYMMDD` local; hour query is UTC (or the reverse). Bag is empty, work is not.
7. **Self-as-sample.** `M` inserts a heartbeat and `COUNT` includes only that type, or excludes “our” ERROR (`LEGACY-13.md`).
8. **Dead writer, quiet lock.** Holder stale; no new ERROR because nothing runs (`LIVENESS-PREDICATE.md`). “No errors” while units stall.
9. **`M` itself failed.** Query ERROR swallowed as 0 (`LEGACY-03.md`). **Fail if `M` exit 0 on its own timeout.**
10. **Consumer drifted.** `C`’s local gate says success; `G` would ERROR (`DEPENDENCY-INVERSION.md`). `M` watches `C`.

**Fail if you “fix” by alerting on any log line:** healthy MEASURE is quiet (`LEGACY-37.md`) and you page forever. **Fail if you widen the hour to 24h local and still only count ERROR.**

## Guard: healthy vs silent

Healthy is a **completed MEASURE** with controls, not a zero. Silent is zero **without** that MEASURE.

```
healthy  iff  probe.kind == FOUND|NONE     # typed, POSITIVE-CONTROL
              AND control.hit AND control.miss
              AND last_successful_scan_utc ≥ now-interval
              AND last_probe_outcome ≠ UNKNOWN
              AND (if locking: same_process live OR no claim required)
              AND errors counted in hot+archive, same UTC window
              AND M's own look completed   # ERROR if not

silent   iff  errors==0 AND NOT healthy
```

`NONE` here is “canary-hit seen, canary-miss out, target bag empty” — a **finding**. **Fail if `NONE` without controls.**

Emit **three** gauges, not one (`LEGACY-38.md`): `errors_in_window`, `last_successful_scan_utc`, `probe_kind`. Alert if `silent` or `probe_kind==ERROR` or `last_successful_scan` stale. **Fail if only `errors>0` pages:** that is how (1)(2)(4) stay dark.

Cheap decisive check (`BLAST-RADIUS-ESTIMATE.md`): run `G.measure(canary-hit)` and `canary-miss` now. Hit missing or miss FOUND ⇒ broken **and** `errors` may still be 0. **Fail if that check is cron-silent on success and omitted on timeout.**

`M` is MEASURE (`GATE-DEADLOCK.md`): read declared store + archive index; do not insert an ERROR so the counter looks live (`LEGACY-17.md`). **Fail if `M` writes a synthetic error to “prove” the pipe.**

List the window with `order_key`, not `ls` (`LEGACY-35.md`). Include archive (`PROVENANCE-ARCHIVE.md`). **Fail if `M` globs only `*.log` current generation.**

Tier: `errors==0` without the guard is `inferred` at best, usually a **guess** about health. Deterministic health is the probe (`EVIDENCE-TIERING.md`). **Fail if the dashboard prints “no errors” without `tier` and `last_successful_scan`.**

**Rule:** Zero ERROR rows are not health. Health is a completed, control-checked look in UTC. Zero plus a dead, wrapped, filtered, or unrun instrument is silent failure.
