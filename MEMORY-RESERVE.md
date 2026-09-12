# Memory layer with a permanent reserve

**Status:** design only.  
**Depends on:** `TASK-09-RETENTION-RISK.md`, `TASK-22-ROLLING-WINDOW-AUDIT.md`, `BUDGET-AWARE-ROUTING.md`, `PROVENANCE-ARCHIVE.md`.  
**Does not exist in this tree:** hot table, evictor, archive migrator.

A **memory layer** `H` (rolling event table, in-flight queue, working set) has a hard cap **N**. Oldest-first wrap with no reserve makes `unreconstructable iff S+L > N/R` (`TASK-09`). Canaries, latches, and live locks die in the same wrap as noise. A **permanent reserve** `ρ` is a slice of N that ordinary rows **must not** occupy. Headroom is an invariant, not a hope.

Stated estate: `N = 2048` rows, `ρ = 256` (canary-hit, canary-miss, live `pool_latch`, in-flight `{pid,start}`, one incident span). **Usable** for evictable work: `U = N − ρ = 1792`.

## Reserve

```
class(row) ∈ { reserved, evictable }
reserved   =  canary-* | pool_latch live | lock live | HWM / policy_sha256
              | in-flight promoted | unmatched/ gauge row for the open join
evictable  =  everything else (terminal done, old heartbeats, stale probes)
```

`reserved` is declared on the row at insert (`TASK-24`), not inferred from “it looks important.” A worker cannot stamp `reserved` (`TASK-17`).

```
invariant  iff  look(H) completed
                AND |{ r in H : evictable }| ≤ U
                AND |H| ≤ N
                AND every canary-hit required by live probes is still in H
```

`ρ` is **permanent**: it is not lent to a retry storm and “paid back later.” Lending `ρ` is `reserve=0` on the only high-class seat (`BUDGET-AWARE-ROUTING.md`). If `|evictable|` would exceed `U`, the insert is refused **or** an evictable row is **migrated** first. The insert never lands in `ρ`.

Failed look of occupancy → UNKNOWN. Do not evict. Do not insert (`TASK-20-FAIL-LOUD.md`).

## Eviction policy (guarantees headroom)

Evict **only** `evictable`, and **only** after a completed migrate (below). Order: oldest flushed `event_id` in `[0,hwm)` among evictable — not `at_utc`, not `ls` / mtime (`IDEMPOTENT-EVENT-LEDGER.md`, `LEGACY-35.md`).

```
on_insert(row):
    if lock_read UNKNOWN: stop
    if class(row)==reserved and reserved_count == ρ: refuse  # reserve full; do not steal evictable’s U
    if class(row)==evictable and evictable_count == U:
        victim = oldest evictable
        migrate(victim)                 # must COMPLETED
        then unlink victim from H       # only that inode / that event_id
    if still no slot: refuse            # queue_full / table_full — not wrap-delete
    insert row; flush; HWM
    assert invariant
```

| Forbidden | Why it loses headroom |
|---|---|
| Delete oldest including canaries | `S+L > N/R` on the control itself (`TASK-22`) |
| Evict `reserved` to make room for evictable | Next probe is UNKNOWN mapped to 0 (`POSITIVE-CONTROL.md`) |
| Unlink-first then “archive if we can” | Crash = never_present (`ADVERSARIAL-ARCHIVE-MANIFEST.md`) |
| Grow N on pressure | Unbounded queue (`TASK-25`) |
| Evict by mtime / “quiet” | Live silent worker looks old (`SILENT-WORKER-STOP.md`) |

Headroom **guarantee:** after every successful `on_insert`, `U − evictable_count ≥ 0` and `ρ` still holds its declared occupants (or refuse). A storm of 10,000 evictable writes/day wraps **U**, not `ρ`. Canaries and latches remain until their own class changes (canary restore is archive→hot into a **reserved** slot).

Zero-acceptance: one insert that drops a live canary or a live latch fails the stratum.

## Migration path (preserves provenance)

Eviction from `H` is **not** delete. It is archive-then-drop, same order as quarantine (`CAPABILITY-QUARANTINE.md`, `PROVENANCE-ARCHIVE.md`).

```
migrate(victim):
    d0 = sha256(canonical(victim bytes))          # this process
    copy regular file → archive/<unit>/<gen>-<d0>/payload
    fsync payload
    write manifest {unit, sha256:d0, predecessor, reason:hot_evict,
                    event_id, gen}                # not appended onto payload
    fsync manifest
    if sha256(payload) != d0: abort               # do not unlink H
    CAS unlink victim only if it still hashes to d0
    index month JSONL: unit, d0, relpath          # pointer, not the only copy
```

**Provenance preserved** iff a third party can re-derive the same `unit` and `d0` from the archive payload after `H` no longer holds the row.

| Must hold | Break |
|---|---|
| `unit` is the handoff id, not `path` / `name[:8]` (`HANDOFF-RENAME-ID.md`) | Rename orphans the archive |
| Payload bytes unchanged (no footer, no “evicted at”) (`LEGACY-21.md`) | Citations 404 or `expected` rewritten |
| `predecessor` chain for the same `unit` | Two archives, no order |
| Distinct inode from hot; no symlink | One inode; unlink deletes both |
| `gen` monotonic (`TASK-30`) | Clock-named dir clobbers |
| Index evicted ≠ payload gone | Wrap of the index looks like never migrated |

Restore (audit, not automatic): copy archive → hot tmp → `sha256 == d0` → rename into `H` only if a **reserved or freed evictable** slot exists. Do not steal `ρ` for a restore of ordinary noise. Do not restore by pointing hot at `archive/` (`exists` lies).

If `migrate` is UNKNOWN (disk full, timeout), **do not** evict and **do not** insert. Headroom is kept by **refuse**, not by drop (`DISPATCH-POOL-LATCH.md` shape: stop, don’t fail-delete).

Canary-miss: plant wrap of `U+1` evictable inserts. If a reserved canary leaves `H` without a verifying archive, the policy failed. Canary-hit: reserved row still in `H` after that wrap.

## What this must not do

- Treat empty disk hope as `ρ`.
- Evict reserved rows because they are old.
- Unlink from `H` before archive verify.
- Migrate by rewriting `expected` or appending a note.
- Auto-grow `N` or auto-restore into a full `U`.

**Rule:** Evictable occupancy never exceeds `U = N − ρ`. Overflow migrates the oldest evictable (copy-hash-then-unlink, `unit`+`d0` in the manifest) or refuses the insert. `ρ` is never lent. Provenance is the archive payload digest, not the hot path.
