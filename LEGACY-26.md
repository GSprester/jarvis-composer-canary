# LEGACY-26

A **truncated** response is an unfinished look. Bytes stopped before a completed, parseable object: torn JSON, cut stream, timeout mid-body, crash before flush, tail past the high-water mark (`TASK-11-IDEMPOTENCE.md`, `TASK-31-GRACEFUL-SHUTDOWN.md`). The instrument **did not finish**. What you hold is not a finding (`LEGACY-03.md`).

A **wrong** response is a finished look at the **wrong predicate**. The body is well-formed and repeatable. It answers a different question than the obligation: date prefix instead of `re`, `exists(path)` instead of digest, `401 invalid_api_key` on this URL instead of three-endpoint bind (`LEGACY-10.md`, `LEGACY-05.md`, `LEGACY-23.md`). Accuracy failed. Completeness did not.

Truncated is “I cannot tell.” Wrong is “I can tell, and it is not this unit.” They share a surface — `[]`, short file, 401, silent watchdog — and they are different types (`LEGACY-25.md`).

| | Truncated | Wrong |
|---|---|---|
| Did the check complete? | No | Yes |
| Can a third party re-parse the same bytes as the author’s object? | No (torn, unmarked, mid-hash) | Yes |
| Same ask again | May finish | Same lie (`LEGACY-10.md`) |
| Estate label | UNKNOWN | DISCARD / UNVERIFIED, not “empty,” not COMPLETED |

## Consumer: truncated

Fail closed (`LEGACY-07.md`). Do not parse a prefix as a document. Do not `json.loads` a cut object and keep the keys that happened to arrive. Do not hash a torn file and compare to `declared.expected` — that is a new digest of junk (`LEGACY-21.md`). Do not map timeout to `[]` or healthy (`TASK-08-WATCHDOG-PATTERN.md`). Do not advance HWM, LKG, or `last_successful_scan`.

Retry only what is safe (`TASK-27-RETRY-SEMANTICS.md`): the **read**, or an idempotent put of the same identity. Same `unit` / send token. Do not mint a second id because the first answer was short. If flush never happened, the writer’s “done” event is a hint (`LEGACY-18.md`); poll declared state.

## Consumer: wrong

DISCARD the conclusions (`LEGACY-19.md`). Do not retry the **same** matcher hoping for a different prefix. The instrument will return the same wrong bit. Do not “fix” it by appending context to the artifact or rewriting `expected` (`LEGACY-17.md`, `LEGACY-21.md`). Name the predicate that actually ran, then run the one the handoff declared: `receipt.re == handoff.name`, `verify(path, digest)`, `(key_id, endpoint)` on all declared regions (`LEGACY-24.md`).

A completed wrong body may still be archived as a **record** (wrapper disposition). It must not close the unit. COMPLETED remains a view over declared bytes (`TASK-17-SEPARATION-OF-DUTIES.md`).

**Rule:** Truncated → UNKNOWN, hide the tail, retry the look. Wrong → DISCARD, change the predicate, never promote the finished lie.
