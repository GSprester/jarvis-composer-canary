# outward job audit

Jobs that **act on** this estate but **live outside** the repository under review: crontabs, timers, daemons, registered services. A PR review of `HEAD` cannot see them. This checkout has no host inventory. The design is why the tree is blind, the **enumeration** that is the look, and a **verdict per item** (`ADVERSARIAL-CRON-10M.md`, `LEGACY-04.md`, `ARGV0-PERMISSION-HOOK.md`).

```
repo review   ⊂   files in the git object
outward jobs  ∈   OS / supervisor / cloud registries
```

Those sets meet only if someone **declared** a job in-tree **and** the live register still points at that digest. The meet is not the default.

## Why a repo review cannot see them

| Place the job lives | What `git grep` / PR diff sees |
|---|---|
| User crontab, `/etc/cron.*`, anacron | Nothing unless a copy was committed (often stale) |
| systemd unit / timer (`systemctl`) | Nothing; units live under `/etc/systemd`, `~/.config/systemd` |
| launchd, `schtasks`, Windows services | Nothing |
| `supervisord`, `pm2`, Docker `--restart`, k8s CronJob in another cluster | Nothing unless that manifest is **this** repo and **this** apply |
| `at`, `incron`, `mdadm` hooks, `if-up.d` | Nothing |
| Cloud scheduler (cron-as-a-service) on another account | Nothing; different trust boundary |

Even an in-repo `deploy/foo.timer` is not the live job. The register may still exec `/opt/old/checkout/scripts/handoff_lifecycle.py` or a `PATH` binary whose basename matches (`ADVERSARIAL-CMD-HOOK.md`). Reviewing the unit **file in git** is not reviewing **what the host will exec next tick**.

`ls` of the repo cannot list `*/10`. Failed host look ≠ “no outward jobs” (`TASK-20-FAIL-LOUD.md`). `open_units=0` in the ledger does not mean cron is gone (`TASK-34-METRIC-THAT-LIES.md`).

## Enumeration procedure

One completed MEASURE per **register**. Canary: a declared `canary-timer` / `canary-cron` **must** appear in that register’s listing (`LEGACY-39.md`). Missing canary → ERROR, stop; do not publish “0 outward jobs.”

```
enumerators = [
  crontab -u * and /etc/cron.d / cron.daily / anacrontab,
  systemd list-timers --all, list-units --type=service --all,
  launchctl / schtasks / Get-Service,   # OS-appropriate
  supervisorctl status, pm2 list,
  docker inspect RestartPolicy,         # local runtime
  atq,
  declared cloud schedulers (typed API; fail ⇒ UNKNOWN)
]

for R in enumerators:
    bag = list(R)                       # raise on timeout; no []
    if canary(R) not in bag: ERROR
    for job in bag:
        record(identity(job), argv, uid, schedule, cwd, unit_path)
        resolve argv[0] → (realpath, inode, sha256)   # after strip of timeout/nohup/env
        verdict(job)
```

Identity is **register + unit name + argv digest**, not a UTC date filename (`TASK-07-DATE-ROLLOVER.md`). `cd &&` jobs: argv is the shell; inner command is UNKNOWN unless parsed as an exact verb (`ARGV0-PERMISSION-HOOK.md`). Privilege hops (`sudo` in cron) stay in the record; do not peel.

Do not enumerate by grepping the repo for `cron` strings. That is the blind review again.

## Verdict per item

Human or wrapper records one of:

| Verdict | Meaning |
|---|---|
| **BOUND** | Live argv[0] digest == declared in-repo publisher; paths inside the reviewed tree (or declared install prefix + digest); schedule declared next to it |
| **STALE** | Register points at a path that **used** to be this repo; digest ≠ `HEAD` (or ≠ last released digest). Job is outward **and** drifted |
| **OUTSIDE** | Runs; no declared in-repo object names this register identity. Repo review never saw it |
| **ORPHAN** | Register exists; binary or unit file missing. Tick will fail or skip-as-success (`LEGACY-04.md`) |
| **DISABLED** | Registered, not enabled. Still list — disable is not delete |
| **UNKNOWN** | Enumerator or `realpath`/hash failed. Not “OUTSIDE,” not “none” |

BOUND is not COMPLETED. It means the job is **in the audit bag** and matches a declaration. COMPLETED remains A∧B on that job’s declared artifact, if any (`TASK-06-ARTIFACT-CHECK.md`). Cron exit 0 is not BOUND and not COMPLETED.

`OUTSIDE ∪ STALE ∪ ORPHAN` is the set a PR cannot fix by editing markdown. Action is: declare (append a ledger `outward_declared` / unit file in-tree **and** re-point the register), disable, or tombstone the register entry. **Fail if you “fix” by committing a copy of crontab and leaving the host unchanged.**

Reconcile with the append-only ledger (`LEDGER-CLASSIFICATION-RECONCILE.md`): each verdict is an event `outward_audited` on the **same** log (`register_id`, `argv_sha256`, `verdict`, `policy_sha256`). Side inventory files get tombstoned after append. `list_next` for operators is the fold, not `crontab -l` pasted into chat.

Zero-acceptance (`CLASSIFIER-ZERO-ACCEPT.md`): one BOUND job whose live digest ≠ declaration fails the **BOUND** stratum for that host. One accepted `canary-miss` timer (a job that must not be installed) fails the enumerator.

## What this does not do

- It does not treat “no cron file in git” as no schedule.
- It does not allow-list `systemd` / `cron` by `argv[0]` basename.
- It does not write `last_ok` at enumeration start.
- It does not invent `enqueued` rows for OUTSIDE jobs (ghost work). It records the audit event only.

**Rule:** Repo review sees the tree. Outward jobs live in registers. Enumerate every register with a canary, then one verdict per item: BOUND, STALE, OUTSIDE, ORPHAN, DISABLED, or UNKNOWN. Empty listing without a canary-hit is ERROR, not “nothing scheduled.”
