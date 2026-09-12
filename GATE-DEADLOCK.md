# gate deadlock

Policy hook `H` runs on every subprocess. It requires a **satisfied** predicate (live claim, `admit`, canary-hit, `verify` path exists) **before** the argv proceeds. The commands that **publish** that predicate (lock rename, canary restore, `verify` itself, ledger append) are also argv, so `H` blocks them. Read-only probes (`find_receipts`, `same_process`, `list_next`) share the hook and die the same way. Result: no measure, no mutate, no escape (`LEGACY-09.md` detector-cancels-the-job).

**Fail if “require lock to acquire lock”:** empty-gap forever (`LEGACY-01.md`). **Fail if `verify` is classified as mutate:** first completion can never be observed. **Fail if ERROR from the hook is mapped to NONE:** store looks empty (`POSITIVE-CONTROL.md`).

## Minimal non-deadlocking fix

Split **MEASURE** from **MUTATE**. `H` applies only to MUTATE.

MEASURE (no hook refuse): `verify`, `find_*`, `same_process`, `list_next`/`peek`, canary read, `boot_at`, ledger **read** `[0,hwm)`. Open **O_RDONLY** on declared paths only (`TASK-32-PATH-SAFETY.md`).

MUTATE (hook on): lock tmp+rename, receipt write, archive copy, ledger **append**, shell that is not MEASURE, unlink hot receipt.

**Fail if MEASURE may write:** gate authors the evidence (`LEGACY-17.md`). **Fail if lock publish is MEASURE:** anyone “probes” by becoming lead. **Fail if you disable `H` entirely to unstick:** one-writer is gone (`LEGACY-12.md`).

Lock acquire is a **one-shot MUTATE** that `H` allows iff `same(record)` is false after a completed probe (`LIVENESS-PREDICATE.md`) and the argv **is** the lock publisher (named module, not `python -c`). **Fail if any `python` can publish:** blanket hole.

## Objection to blanket-whitelisting

Allow-listing binaries (`python`, `git`, `cat`, `sha256sum`, `/usr/bin/*`) unblocks MEASURE **and** every bypass those binaries implement: `python` sets `status=completed`, `cat > out/bundle`, `git` second clone (`LEGACY-16.md`). **Fail:** the hook still “exists” and CI is green; the seat mutates without a claim. **Fail if you whitelist by argv[0] basename:** `./python` or `python3.12` slips. **Fail if you whitelist the worker image:** the image **is** the mutator.

## Fix that avoids both

Classify by **declared verb + path class**, not by binary name.

```
class(argv) =
  MEASURE  if argv is one of {verify, find, alive, peek, ledger_read}
           AND every path is declared or .lock/writer-<unit>
           AND open flags ⊆ {O_RDONLY}
  ACQUIRE  if argv is lock_publish AND CAS stale/absent
  MUTATE   if live claim for this unit AND path declared
  REFUSE   otherwise
```

`H` implements `class`. MEASURE never satisfies MUTATE’s predicate (it cannot create the file `verify` wants). ACQUIRE is the only way a claim appears; it does not run `verify` to green. **Fail if ACQUIRE calls `verify` and writes a stub on miss.**

Canary-hit/miss are **data** MEASURE reads. Restore from archive is MUTATE under a live claim, or a **bootstrap** ACQUIRE of unit `canary-hit` only, once, issuer digest. **Fail if any seat can ACQUIRE `canary-hit`:** they mint the control (`LEGACY-39.md`).

Ledger: `enqueued`/`promoted`/`released` stay MUTATE; `released` still after delivery (`IDEMPOTENT-EVENT-LEDGER.md`). **Fail if MEASURE may append `released`.**

## Still refuse

- Worker `set_completed` / `exists` as done (`TASK-17-SEPARATION-OF-DUTIES.md`)
- Rewrite `expected_sha256` or append onto the payload (`LEGACY-21.md`)
- `seat.ceiling = provider.class` (`TASK-13-FAILOVER-CEILING.md`)
- New `unit` / `enqueue_id` on retry (`TASK-27-RETRY-SEMANTICS.md`)
- Fuzzy receipt join (`RECEIPT-MATCHING.md`)
- Delete hot receipt instead of archive (`STALE-CLAIM-RECOVERY.md`)
- Steal on UNKNOWN liveness (`LIVENESS-PREDICATE.md`)
- `just run the shell` without claim (`TASK-25-QUEUE-BACKPRESSURE.md`)
- Absolute / `..` / undeclared paths
- Hook ERROR treated as NONE or healthy (`LEGACY-38.md`)

**Fail if refuse-list lives only in a README:** next wrapper re-enables the argv (`LEGACY-15.md`). Tombstone the old hook entrypoint so it raises.

**Rule:** Measure is free and read-only. Acquire is one named CAS. Mutate needs a live claim. Nothing else runs.
