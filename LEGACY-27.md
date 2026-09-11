# LEGACY-27

A **silent reset** restores packaged defaults (or deletes keys so resolution falls through, `TASK-29-CONFIG-PRECEDENCE.md`) **without** a declared intent. An **intentional edit** changes the declared bytes and leaves a third-party-checkable receipt. `exists(path)` and mtime cannot tell them apart (`LEGACY-05.md`, `LEGACY-14.md`). The detector must not be the process that writes the file (`LEGACY-09.md`, `LEGACY-17.md`).

Hash **after** canonicalise (`TASK-15-IDEMPOTENT-RECEIPT.md`). Store the last intentional digest **outside** the config (`LEGACY-21.md`). Failed look → UNKNOWN, not “unchanged” (`LEGACY-03.md`).

## 1. Triple digest

Keep three sha256s: `live` (file now), `intentional` (last flushed operator/wrapper write), `packaged` (shipped template, declared at image build).

```
silent_reset  iff  live == packaged  AND  intentional != packaged
edit          iff  live ≠ intentional  AND  live ≠ packaged
unchanged     iff  live == intentional
```

`live == packaged == intentional` is a first boot or an intentional revert — only the receipt in §2 distinguishes revert from “never configured.” A third party re-hashes the three declared paths.

## 2. Intent receipt missing for the new digest

Every intentional write appends a grant-shaped row first (`LEGACY-22.md`): `grantor`, wrapper `pid+start`, `before_sha256`, `after_sha256`, `unit`, UTC. Flush, then replace the file. Detect reset when `live ≠ intentional` **and** no completed row has `after_sha256 == live`. Image bake, package reinstall, and “helpful” defaults writers do not mint that row. Do not let the resetter write the receipt.

## 3. Add-only keys vanished

Shipped config is add-only (`TASK-33-SCHEMA-EVOLUTION.md`). Operator keys (`seat_ceiling`, regional endpoints, `key_id`) once present stay present or are **tombstoned**. A well-formed file that **omits** those keys and parses as the v1 template is a reset, not a careful delete. A delete without `successor_id` is not an edit (`LEGACY-15.md`). Absent key → default is the silent path (`TASK-29-CONFIG-PRECEDENCE.md`); `""` is set and is an edit if receipted.

## 4. Generation sidecar, not mtime

A sidecar `config.gen` (integer) increments **only** on an intentional write, fsync-then-rename (`TASK-30-LOG-ROTATION.md` generation rule). Reset copies the packaged file and leaves `gen` behind, or wipes both and `gen` is missing. `mtime` after an image pull is new and lies (`LEGACY-14.md`).

```
silent_reset  iff  live == packaged  OR  (live ≠ last_bytes_for_this_gen)
                  AND  gen did not advance with live
```

Missing sidecar → UNKNOWN, not “gen 0 = defaults are fine.”

## 5. Operator bindings collapsed to factory

Hash only fields **defaults never set**: `seat.ceiling`, provider allow-list, three probe endpoints (`LEGACY-24.md`). If that projection equals the packaged empty/public set while the file still parses and `exists`, the interesting policy was stripped. Wholesale reset and “set `seat.ceiling = provider.class` then reboot into defaults” both show up here (`TASK-23-POLICY-INHERITANCE.md`). Do not hash LKG or worker status (`LEGACY-09.md`).

**Rule:** Intentional = new digest **and** a flushed receipt **and** gen advance. Live bytes equal to packaged, or keys gone, without that triple → silent reset. Escalate; do not admit on factory policy as if someone chose it.
