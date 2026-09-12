# Two-lane split

**Status:** design only.  
**Depends on:** `TASK-16-ONE-LEAD-LOCK.md`, `TASK-18-COUNT-RECONCILIATION.md`, `TASK-26-DEDUPE-BY-CONTENT.md`, `TASK-27-RETRY-SEMANTICS.md`, `STALE-CLAIM-RECOVERY.md`, `SAME-MODEL-PAIR.md`.  
**Not this:** `ORDERED-RELEASE-QUEUE.md` lanes (priority frozen at promote).  
**Does not exist in this tree:** dispatcher, two worker pools, merge job.

One **job** is a declared bag of units. Two **lanes** of **identical** workers (`M` equal, same admit class) exist to raise throughput, not to verify each other. Identity of a unit is `re` (`handoff.name`), never a UTC prefix (`TASK-04-MATCHER-TESTS.md`).

## Partition rule

Each unit has exactly one **home** lane. Home is a function of stable ids, computed once at admit, recorded on the handoff. Workers do not choose.

```
home(u)  =  sha256( canonical(job_id) || 0x00 || canonical(u.re) )  mod  2
```

`canonical` is the same byte string the receipt hasher uses (`TASK-15-IDEMPOTENT-RECEIPT.md`). Do not hash `name[:8]`, seat id, enqueue clock, or filename `lane-{n}`.

| Property | Hold |
|---|---|
| Same `re` | same home, every replay |
| Balance | last bit of a content hash; not “whichever queue is shorter” as the assignment |
| Exclusivity | only home may **first** acquire the writer lock (`TASK-16`) |
| Bound | each lane’s queue is still cap `N` (`TASK-25-QUEUE-BACKPRESSURE.md`). Split does not double the estate cap by enqueueing every unit twice |

**Not a partition:** send every unit to both lanes at admit. That is a same-SKU pair (`SAME-MODEL-PAIR.md`). Identical workers do not decorrelate bias. It also burns two slots per unit.

**Steal is not a re-partition.** The other lane may run `u` only after home is **not alive** and the unit is not COMPLETED, via stale-claim recovery (`blocked → interrupted → open`). Steal publishes a new `pid+start` on the **same** `unit` / same declared path. Do not mint a second `re`. At most **one** steal. A steal while home is alive is two leads (`LEGACY-01.md`).

Lane membership of a running claim is `home` or `stolen`. It is not inferred from which host happened to dequeue.

## Merge rule

Merge is a **bag reconcile** of declared `re`s against accepted artifacts, not a concatenation of lane files and not `len(A)+len(B)==len(job)`.

After both lanes are terminal (or the job look is UNKNOWN):

```
declared  =  bag of re in the job
accepted  =  bag of re whose Sig1 ∧ Sig2 hold (one digest each)
merge     =  reconcile(declared, accepted)     # TASK-18
```

`job_complete` iff `added = removed = ∅` and every accepted `re` has **exactly one** digest and `n_unknown = 0`.

Per-`re` table:

| Home | Other | Merge |
|---|---|---|
| Sig1∧Sig2, no other write | — | take home digest |
| not COMPLETED, steal Sig1∧Sig2 | one digest, `stolen_from=home` | take steal |
| both Sig1∧Sig2, **same** digest | collapse (`TASK-26`) | one COMPLETED |
| both Sig1∧Sig2, **different** digest | **conflict** | job not complete; do not pick later / larger / prettier |
| one COMPLETED, one fail, **same path different bytes** | **conflict** | fail wrote a payload; do not hide it |
| neither COMPLETED | see dual-fail / UNKNOWN | no COMPLETED |

Do not sort merge by `mtime`, `ls`, or “whichever finished” (`ORDERED-RELEASE-QUEUE.md`). Do not emit a job-level COMPLETED because both lanes drained their queues — a lane can drain by dropping (`TASK-34-METRIC-THAT-LIES.md`).

A conflict is not resolved by a third identical worker. It is two leads’ bytes. Escalate or issuer-abort. Wrapper records the disposition (`TASK-05-DOCTRINE-DRAFT.md`).

## Dual-fail (unit failed on both sides)

**Both sides** means: home completed a **work** attempt that is not COMPLETED, **and** the one legal steal completed a work attempt that is not COMPLETED. Both looks finished.

| Pair | Not dual-fail |
|---|---|
| Either look timeout / unreadable / lock-read UNKNOWN | `UNKNOWN` for that `re`. Do not merge as fail (`TASK-20-FAIL-LOUD.md`) |
| Home fail, steal never admitted | single-lane fail; steal may still be legal |
| Both `queue_full` / hop dead | estate, not the unit. Re-admit **home** when the hop is healthy. Do not call it dual-fail |
| Greedy second draw on the same live claim | one attempt counted twice |

**Disposition:** `failed_both`. The unit is **not** COMPLETED. Declared path is not accepted. Partial bytes from home and from steal are **not** concatenated into a survivor.

**No third identical attempt.** Two lanes already are the same estimator (`SAME-MODEL-PAIR.md`: `q → 1` on shared misses). A third draw is variance-only and is the crash-loop the stale-claim guard already caps. After `failed_both`:

1. Archive both hot receipts, drop hot, leave `interrupted` (`STALE-CLAIM-RECOVERY.md`).
2. Do not `open` for the same `M` / same admit class.
3. Escalate: different estimator (other family after a rubric diff, human MEASURE, or deterministic checker) **or** issuer abort of the job.
4. Job bag stays incomplete: `removed` contains this `re`. Successful siblings stay accepted. Do not fail-close the whole bag to hide the hole, and do not mark the hole COMPLETED to make `len` match.

Dual-fail is **not** evidence that the unit is impossible. Identical workers agreeing on fail is the bias event. Do not publish “confirmed unworkable” from `failed_both`.

## What this must not do

- Assign home by date prefix, seat load, or worker self-report (`T`).
- Enqueue the full bag on both lanes as a check.
- Steal under a live `pid+start`.
- Merge by count, mtime, or file-order concat.
- Resolve two digests for one `re` by picking one.
- Treat `failed_both` as COMPLETED, as UNKNOWN, or as a license for a third same-SKU launch.

**Rule:** Home is `sha256(job_id, re) mod 2`. Merge is bag-reconcile of `re` → one digest. A unit that fails home and fails the one steal is `failed_both`: archive, do not COMPLETED, do not retry the same estimator.
