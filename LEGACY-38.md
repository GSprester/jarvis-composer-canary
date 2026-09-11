# LEGACY-38

An **environment probe** answers: may this harness admit work on this host, policy, and store? Two-state (`ok` / `fail`) cannot say “I did not obtain a reading.” The missing third state is always implemented anyway — as **pass** (`timeout → []`, silence → live, `exists` → ready). That is fail-open (`LEGACY-07.md`, `LEGACY-03.md`, `LEGACY-37.md`).

## Enum

```
ProbeResult = healthy | unhealthy | UNKNOWN
```

Same three wires as the watchdog (`TASK-21-WATCHDOG-SILENCE.md`, `TASK-08-WATCHDOG-PATTERN.md`). No fourth “degraded” that callers treat as ok. Extra keys on the row are ignored (`TASK-33-SCHEMA-EVOLUTION.md`).

| Value | Meaning | Probe completed? | Admit / dequeue / LKG |
|---|---|---|---|
| `healthy` | Predicate ran; environment matches declaration | Yes | Allow |
| `unhealthy` | Predicate ran; environment is wrong | Yes | **Block** |
| `UNKNOWN` | No reading: timeout, raise, missing callable, torn body, canary absent | No | **Block** |

```
healthy    iff  completed look
                AND canary two-factor visible          # LEGACY-33, LEGACY-32
                AND live policy digest == intentional or receipted edit  # LEGACY-27
                AND admit(seat, provider) is decidable # classes known, TASK-23
unhealthy  iff  completed look AND a declared clause is false
UNKNOWN    iff  not a completed look
```

`unhealthy` examples (completed): packaged reset without intent, `seat.ceiling` missing, route `pre_send_refuse` when a hop was required, canary digest mismatch. `UNKNOWN` examples: store timeout, lock unreadable, truncated probe body (`LEGACY-26.md`), one of three endpoints not asked (`LEGACY-24.md`), `route_intent` without result (`LEGACY-30.md`).

Do not encode UNKNOWN as `None`, `[]`, `0`, or empty string. Those are negative findings or pass (`TASK-20-FAIL-LOUD.md`).

## Why UNKNOWN must block, not pass

**Pass** means the next seat may mutate the tree. A probe that did not finish has not measured the environment. Treating UNKNOWN as `healthy` is an unverified claim read as settled (`LEGACY-31.md`): cron exits 0, the detector skips, failover hops, keys rotate, `open_units` stays 0 (`LEGACY-04.md`, `TASK-34-METRIC-THAT-LIES.md`). The instrument produced no measurement; the gate would be minting “ok” so work can start (`LEGACY-17.md`).

**Block** is fail closed: no admit, no dequeue, no lock steal, no LKG update, no silent watchdog path (`TASK-08-WATCHDOG-PATTERN.md`). Disposition is `probe_unknown`, wrapper-authored (`TASK-05-DOCTRINE-DRAFT.md`). Retry the **probe** (idempotent read), not a new `unit` (`TASK-27-RETRY-SEMANTICS.md`). Humans escalate; models do not rewrite `seat.ceiling` to make the probe pass.

`unhealthy` also blocks, but it is a **finding**. UNKNOWN is not “probably fine” and not “probably broken.” It is the third type. Collapsing it into `healthy` hides a dead store. Collapsing it into `unhealthy` pages on every timeout and invites a “fix” that inserts a stub canary so the look succeeds (`LEGACY-17.md`). Block, stay UNVERIFIED, look again.

**Rule:** Two states describe the world after a reading. The third describes the instrument. If the instrument did not finish, nothing that needs an environment may proceed.
