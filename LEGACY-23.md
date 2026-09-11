# LEGACY-23

**Invalid key** means the credential is not a live secret for **any** declared endpoint: missing, empty, malformed, revoked, or never issued. The predicate “does a completed authn check accept these bytes as a key?” is false everywhere. Fail closed; do not retry the same token (`TASK-28-ERROR-CLASSIFICATION.md`). Do not mint a replacement from the gate (`LEGACY-17.md`).

**Valid for a different region** means those bytes **are** a key — for another provider name, class, or regional base URL than the one this admit asked. `admit(seat, provider)` failed because **binding** failed, not because the secret is nonsense. The same seat against the declared region would pass. Identity is `re` / unit / `pid+start` / `min(seat, provider)`, not “whatever hop answered” (`TASK-23-POLICY-INHERITANCE.md`, `TASK-04-MATCHER-TESTS.md`).

They are different types, like empty vs failed search (`LEGACY-03.md`) and 429 vs 402 (`TASK-28-ERROR-CLASSIFICATION.md`). One is “no such credential.” The other is “wrong store.”

## Why the second is routinely called the first

Vendors collapse both onto one wire: `401`, `403`, `invalid_api_key`, `unauthorized`. The client asked **this** regional URL. The server answers a boolean about **this** URL. A valid us-east key on a eu-west host is, locally, a failed authn. Operators and failover code then take the only named class they have — **invalid** — and rotate, delete, or escalate billing. The key still works in the region it was issued for. The dashboard zero is a lying “no key” (`TASK-34-METRIC-THAT-LIES.md`).

The probe also **omits region from identity**, the way a date-prefix matcher omits `re` (`TASK-07-DATE-ROLLOVER.md`). Config resolves a key by name and a base URL by another name (`TASK-29-CONFIG-PRECEDENCE.md`). Equality is `status != 200`, not `(key_id, region) == declared`. Failover hops providers to “get a working key” and silently widens class, or treats a ceiling deny as a bad secret (`TASK-13-FAILOVER-CEILING.md`). Same shape as Windows path `==`: one string did not match this filesystem, so the script reports “not a path” (`LEGACY-02.md`).

A third party cannot tell the two apart from the status bit alone. Log `provider`, `provider_class`, regional endpoint, and `policy_sha256` with the deny (`LEGACY-22.md`). Classify `unknown_key` vs `region_mismatch`. Retry or hop only for the second, and only under `min(seat, provider)`. Rotate only for the first.

**Rule:** “Invalid” is a property of the **secret**. “Wrong region” is a property of the **bind**. One HTTP code is not a completed check of both.
