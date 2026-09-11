# LEGACY-32

**Done** is a conjunction of two factors that do **not** share an author or a file. A stub (`touch`, empty create, `DONE:\n`, date-prefixed leftover) can satisfy any **one** path-shaped check (`LEGACY-05.md`). It cannot satisfy both factors below. The worker cannot write either factor’s success bit (`TASK-17-SEPARATION-OF-DUTIES.md`). The gate does not mint the digest or the receipt (`LEGACY-17.md`).

Empty digest is not a declared obligation. `handoff.expected_sha256` must be present **before** the seat runs, must not be the sha256 of `b""`, and must not be copied from a worker record.

## Factor A — payload (issuer digest)

```
A  iff  path is the declared lexical path
        AND sha256(canonical file bytes) == handoff.expected_sha256
        AND expected ≠ sha256(b"")
```

This is `verify` (`TASK-24-ARTIFACT-DECLARATION.md`, `TASK-06-ARTIFACT-CHECK.md`). A stub fails unless it **is** the issuer’s bytes. Existence, size > 0, mtime, and a self-hash inside the file are not A (`TASK-15-IDEMPOTENT-RECEIPT.md`, `LEGACY-21.md`).

## Factor B — binding (checker receipt)

```
B  iff  a receipt row exists in the append-only log (flush then HWM)
        AND receipt.re == handoff.name          # not name[:8]
        AND receipt.sha256 == handoff.expected_sha256   # citation, not a new hash
        AND the writer is the checker incarnation (pid+start ≠ worker)
        AND the receipt is not appended onto the payload
```

B is a **different object** (`TASK-11-IDEMPOTENCE.md`, `TASK-04-MATCHER-TESTS.md`). The checker writes it **after** A. The worker API has no path to this log (`LEGACY-16.md`). A stub file next to a stub `DONE` sidecar is not B: wrong `re`, or worker `pid+start`, or digest taken from the stub.

## Conjunction

```
done  iff  A AND B
```

| What the stub tries | A | B |
|---|---|---|
| `touch` / empty file | fail (empty ≠ expected) | no checker row |
| `DONE:` / preview bytes | fail (wrong digest) | no row or `re` prefix-only |
| Worker writes receipt + stub | fail or ignored worker sha256 | fail (author is worker) |
| Declare `expected = sha256(stub)` after write | fail (expected not on handoff before run) | checker will not cite it |

Unverified read as settled is exactly A or B alone (`LEGACY-31.md`). One factor is a **record**. Two factors are a **check**. Missing a look at either is UNKNOWN, not done (`LEGACY-03.md`).

**Rule:** Payload bytes match the issuer. A checker who is not the writer cites that digest on `re`. A stub can fake a file. It cannot be both the issuer’s bytes and a foreigner’s receipt.
