# TASK-15-IDEMPOTENT-RECEIPT

This checkout has no receipt writer. The contract: given the same **handoff id** and **outcome**, two writes must produce **byte-identical** receipt files. The matcher (`TASK-04-MATCHER-TESTS.md`) keys on `receipt.re == handoff.name`; the artifact checker (`TASK-06-ARTIFACT-CHECK.md`) keys on the file digest. Both require stable bytes, not “same meaning.”

## Inputs that must not leak into the bytes

| Allowed in the receipt | Forbidden (breaks byte identity) |
|---|---|
| `re`: exact handoff id | Wall-clock `written_at`, local date in the receipt **name** |
| `outcome`: declared enum (`ok`, `reboot_interrupted`, …) | Host, pid, seat session id |
| Canonical JSON body | Pretty-print, key insertion order, trailing newline variance |

A UTC date on the **filename** is display (`TASK-07-DATE-ROLLOVER.md`). It is not part of the hashed body. The body uses `re`, not `name[:8]`.

## Canonicalisation step (hash **after** this)

```python
def canonical_body(handoff_id: str, outcome: str) -> bytes:
    # 1. Restrict to the identity pair. No clocks.
    obj = {"outcome": outcome, "re": handoff_id}
    # 2. Normalise: sorted keys, tight separators, UTF-8, one trailing newline.
    return (
        json.dumps(obj, ensure_ascii=True, separators=(",", ":"), sort_keys=True)
        + "\n"
    ).encode("utf-8")


def write_receipt(path: Path, handoff_id: str, outcome: str) -> bytes:
    body = canonical_body(handoff_id, outcome)
    digest = hashlib.sha256(body).hexdigest()
    # 3. Hash the normalised bytes. Embed digest in a second canonical envelope
    #    whose field order is also sorted, so the file is determined by (id, outcome).
    envelope = {"outcome": outcome, "re": handoff_id, "sha256": digest}
    blob = (
        json.dumps(envelope, ensure_ascii=True, separators=(",", ":"), sort_keys=True)
        + "\n"
    ).encode("utf-8")
    path.write_bytes(blob)
    return blob
```

`write_receipt(p, "20260910-alpha", "ok")` called twice yields the same `blob`. A third-party checker hashes the file and compares to `envelope["sha256"]` only after recomputing sha256 of `canonical_body` (the envelope without `sha256`), or treats the whole file as the artifact and stores the digest **outside** the file (`TASK-06-ARTIFACT-CHECK.md`).

Self-check: `write_receipt` then `write_receipt` again; `first == second` as bytes.

## Why hashing **before** normalisation breaks the guarantee

If you hash the object **as first seen** (Python dict order, pretty JSON, local receipt name `20260911-alpha-r0`) and then serialise a different, “cleaned” form:

1. **The digest is not a digest of the file.** Write 1 hashes `{"re":...,"outcome":...}` (insertion order). Write 2 hashes `{"outcome":...,"re":...}`. JSON default dumps differ, so `sha256` differs, so the written envelopes differ — same handoff, same outcome, not byte-identical.
2. **Embedded hash poisons the second write.** Even if you later `sort_keys=True`, the **hash field** was computed on the unnormalised bytes. Two writers that normalise at different times embed different `sha256` values into otherwise-sorted JSON. The files still differ.
3. **Date-prefix names leak.** Hashing `receipt.name` (`20260911-*`) before replacing it with `re` (`20260910-alpha`) makes a next-UTC-day retry a new hash for the same handoff and outcome. Idempotence is gone; the matcher still wants `re`.

Order that holds: **normalise → hash those bytes → write**. Hash-then-normalise writes a lie: the digest does not match the artefact a third party will read, and a retry is not byte-identical.

## Minimal proof

```python
a = write_receipt(Path("r1.json"), "20260910-alpha", "ok")
b = write_receipt(Path("r2.json"), "20260910-alpha", "ok")
assert a == b
assert hashlib.sha256(canonical_body("20260910-alpha", "ok")).hexdigest() in a.decode()
```

A writer that hashes `json.dumps(obj)` without `sort_keys=True` fails this assert as soon as key order differs.
