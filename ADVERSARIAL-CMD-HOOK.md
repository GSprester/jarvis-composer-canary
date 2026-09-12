# adversarial review: hook enforcing policy by string-matching commands

Mechanism: pre-exec hook `H` allow/denies argv by **basename**, **substring**, **regex**, or **allow/deny list of words**. Success is typically **ALLOW**, **0 denials**, **CI green**, or **“policy held.”** Confident wrong = **success, completeness, or safety** that does not hold.

## 1. Interpreter / `/usr/bin/*` allow-list = policy **enforced**

**Failure:** `python`, `git`, `cat`, `sha256sum` match ALLOW. Those binaries implement every bypass (`GATE-DEADLOCK.md`). Hook **exists**; seat mutates without a claim.
**Trigger:** unblock MEASURE after deadlock; whitelist the worker image; `argv[0]` in a folder list.
**Symptom:** CI green; `status=completed`; `cat > out/bundle`; second clone (`LEGACY-16.md`).
**Cheapest guard:** classify by **declared verb + path class + open flags**, not binary name. Any `python` publishing a lock ⇒ REFUSE.

## 2. ALLOW by `argv[0]` basename

**Failure:** `./python`, `python3.12`, `Python.exe`, `busybox` applet name, copy on `PATH`. String **matches**. Exec is not the named module.
**Trigger:** cron `PATH` (`ADVERSARIAL-CRON-10M.md` §8); wrapper in cwd; PATHEXT.
**Symptom:** hook ALLOW; inode ≠ declared publisher; two leads (`LEGACY-01.md`).
**Cheapest guard:** compare `realpath(argv[0])` inode/digest to the shipped publisher. Basename is not identity.

## 3. Substring ALLOW (`verify` / `find` / `lock` in the line)

**Failure:** a token appears. Class is MEASURE/ACQUIRE. Body is MUTATE.
**Trigger:** `python -c 'verify=1; open("out","w")'`; path `/tmp/verify`; `--help verify`; filename `find_receipts.py` that writes.
**Symptom:** ALLOW; stub COMPLETED (`LEGACY-05.md`); canary minted (`LEGACY-17.md`).
**Cheapest guard:** argv is an exact verb enum `{verify, find, alive, peek, ledger_read, lock_publish}` **and** remaining args are declared paths. Substring is not a class.

## 4. Deny-list default-ALLOW: new verb is **safe**

**Failure:** completeness of the refuse set. Unknown argv **proceeds**.
**Trigger:** add-only reader ignores new keys (`TASK-33-SCHEMA-EVOLUTION.md`); README-only refuse (`LEGACY-15.md`); `just run the shell`.
**Symptom:** hook 0 denials; undeclared mutate; next wrapper re-enables the argv.
**Cheapest guard:** default REFUSE. Unknown `class(argv)` ⇒ ERROR, not ALLOW. Tombstone old entrypoints so they raise (`LEGACY-16.md`).

## 5. Hook ERROR / MEASURE refuse ⇒ NONE / skip / healthy

**Failure:** blocked look looks like **empty/safe** (`POSITIVE-CONTROL.md`). Policy “held.”
**Trigger:** `verify` classified as MUTATE (`GATE-DEADLOCK.md`); `H` on `find_*`; ERROR mapped to `[]`.
**Symptom:** `list_next=[]`; cron 0; WOs OPEN (`SILENT-FAILURE-DETECTION.md`).
**Cheapest guard:** MEASURE never refused by `H`. Hook ERROR ⇒ ERROR, not NONE. Canary-hit missing ⇒ ERROR.

## 6. Deadlock: nothing runs, **0 violations**

**Failure:** safety of a quiet hook. Lock cannot be acquired; verify cannot run (`LEGACY-09.md`).
**Trigger:** “require lock to acquire lock”; ACQUIRE needs `verify` which needs the lock.
**Symptom:** last_ok green if start-touched; no `promoted`; operators: “hook is working.”
**Cheapest guard:** MEASURE free; ACQUIRE is one named CAS; MUTATE needs live `{pid,start}`. 0 denials is not a completed MEASURE.

## 7. Hook sees the wrapper, not the exec (`sh -c`, `sudo`, `ssh`, shebang)

**Failure:** outer string ALLOW (`bash`, `env`). Inner argv never matched.
**Trigger:** `bash -c "$CMD"`; `python script.py`; `#!/usr/bin/env python3`; `nsenter`/`sudo`.
**Symptom:** policy held on `bash`; child writes `COMPLETED`; wait 0 (`ADVERSARIAL-CRON-10M.md` §16).
**Cheapest guard:** `H` on every `execve`, including interp and `-c`. Wrapper ALLOW ≠ child ALLOW. Classify the **writer** incarnation.

## 8. `-c` / `-m` / stdin / `eval`: forbidden tokens not in argv

**Failure:** argv is `{python, -c, <code>}` or `{sh, -s}`. Deny regex looks at words like `rm`/`completed` and **misses**. ALLOW.
**Trigger:** base64|sh; `import worker; worker.set_completed()`; heredoc.
**Symptom:** hook 0 hits; payload or status bit written.
**Cheapest guard:** `-c`/`-m`/`stdin` of an interpreter is REFUSE unless the named module path is the declared verb. Do not regex the code string as policy.

## 9. Redirect / pipe / fd not in argv (`cat > out`, `tee`)

**Failure:** argv `{cat, file}` ALLOW as MEASURE. Bytes land on a declared or undeclared path.
**Trigger:** shell `>`; `dup2` before exec; `tee log` last-command 0 (`ADVERSARIAL-CRON-10M.md` §10).
**Symptom:** hook ALLOW; digest moved (`LEGACY-21.md`); `C` presence-gates.
**Cheapest guard:** MUTATE is any write fd to a non-MEASURE path. String match cannot see redirects — refuse shell, or enforce open flags in the hook.

## 10. `PATH` / `LD_PRELOAD` / `PYTHONPATH`: name matches, binary does not

**Failure:** allowed string execs attacker code. Policy **same**.
**Trigger:** cwd first on PATH; `LD_PRELOAD`; venv hijack; busybox install.
**Symptom:** ALLOW; inode ≠ shipped digest; lock published by a stranger.
**Cheapest guard:** exec only if `argv[0]` digest == declared. Env that changes loader/import of MUTATE ⇒ REFUSE. Clear `PATH` to a sealed dir for ACQUIRE/MUTATE.

## 11. Encoding / quoting / IFS / homoglyph / NUL: match misses

**Failure:** reconstructed command ≠ bytes exec’d. Deny pattern **safe**.
**Trigger:** Unicode `ｖerify`; `rm\x00`; newlines; Windows quoting; `IFS=,`.
**Symptom:** ALLOW; audit log shows the pretty string that would have denied.
**Cheapest guard:** match on the **execve argv array**, not a joined line. Non-ASCII / NUL in verb ⇒ REFUSE. Do not parse a shell string as argv.

## 12. `exec` replace after ALLOW of the parent

**Failure:** hook allowed `lock_publish`. Same pid, new image (`ADVERSARIAL-PID-ONLY-LIVENESS.md` §5). Policy already **passed**.
**Trigger:** worker `execve` next job; crash wrapper execs `sh`.
**Symptom:** claim still “ours”; new argv writes another `unit`.
**Cheapest guard:** `H` on every exec, including same-pid. Record `exe` digest + spawn token. ALLOW does not outlive the image.

## 13. `--readonly` / flag strings as MEASURE

**Failure:** a word in argv claims read-only. Process writes. Class **MEASURE**.
**Trigger:** `--dry-run` leftover (`ADVERSARIAL-MAILBOX-QUEUE.md` §25); `--check` that writes LKG; `O_RDONLY` only in the help text.
**Symptom:** ALLOW; files created; gate minted evidence (`LEGACY-17.md`).
**Cheapest guard:** MEASURE iff actual open flags ⊆ `{O_RDONLY}` on declared paths. Flag text is `model_guess`.

## 14. Path argument matches `verify` but is `..` / absolute / undeclared

**Failure:** verb ALLOW. Path escapes (`TASK-32-PATH-SAFETY.md`).
**Trigger:** `verify ../out`; `find /tmp/done`; resolve-before-validate.
**Symptom:** ALLOW; hit outside root; estate tree untouched or escaped.
**Cheapest guard:** lexical validate every path **before** join. Escape ⇒ REFUSE even if the verb is MEASURE.

## 15. Hook disabled / bypassed to unstick; file still **present**

**Failure:** `H` exists on disk. Enforcement **complete**. Execs do not enter `H`.
**Trigger:** `H=0`; `LD_DEBUG`; direct syscall; second wrapper without hook; “disable to unstick” (`GATE-DEADLOCK.md`).
**Symptom:** CI greps the hook module; two leads; “just run the shell” (`LEGACY-12.md`).
**Cheapest guard:** MUTATE without a live claim is `queue_full` / refuse at `offer`, not only at `H`. Missing hook on exec ⇒ ERROR, not ALLOW. Do not disable `H` to fix deadlock.

## 16. Two predicates: string-hook wins over `admit` / A∧B

**Failure:** hook ALLOW is **the** gate. Ceiling, digest, `re` ignored (`LEGACY-15.md`).
**Trigger:** README says `min(seat, provider)`; run path is `grep argv`; `C` local hook (`DEPENDENCY-INVERSION.md`).
**Symptom:** restricted WO ALLOW on `public` argv; verify UNVERIFIED; change-detector skips the real gate (`LEGACY-09.md`).
**Cheapest guard:** tombstone string-ALLOW as admit. One published predicate: `class(argv)` ∧ `admit` ∧ live claim for MUTATE. Dual-run until the string branch is dead.

## 17. Self-satisfy: hook writes the predicate its next match requires

**Failure:** `H` authors claim/canary/`exists` so the following string ALLOW. Gate arbitrates the command (`LEGACY-17.md`).
**Trigger:** ACQUIRE calls `verify` and stubs on miss; hook `touch` canary; insert dummy receipt.
**Symptom:** every argv after the mint ALLOW; controls are stubs (`POSITIVE-CONTROL.md`).
**Cheapest guard:** `H` only reads and refuses. ACQUIRE does not `verify` or write payload. Canary ACQUIRE only issuer, once.

## 18. Fuzzy / prefix command match (same shape as receipts)

**Failure:** `ver` / `python3` / date-prefixed script name **is** the verb.
**Trigger:** `name[:8]`; `cmd.startswith("verify")`; allow `*lock*`.
**Symptom:** `verify_and_write` ALLOW; 74 quoted names vs 494 (`RECEIPT-MATCHING.md`).
**Cheapest guard:** exact enum. Prefix/fuzzy ⇒ REFUSE. Canary-miss command that prefix would steal must stay out.

## 19. Audit “0 denials” / hook log exists = scan **complete**

**Failure:** completeness of enforcement. Missed execs never logged.
**Trigger:** inotify overflow; hook not in the pidns; `os.system` not wrapped; marker `hook.ok` (`ADVERSARIAL-EXISTS-SCANNER.md` §16).
**Symptom:** dashboard 0 violations; MUTATE happened; `last_ok` fresh.
**Cheapest guard:** 0 denials is not a MEASURE. Canary-deny (must refuse) missing ⇒ ERROR. Count execs vs hook entries; gap ⇒ ERROR.

## 20. Policy file string-match: empty / reset / substring “class”

**Failure:** hook reads a line that **looks** like deny/allow. Effective class wrong (`TASK-23-POLICY-INHERITANCE.md`).
**Trigger:** silent reset to packaged defaults (`LEGACY-27.md`); `class` substring in a comment; empty string wins (`TASK-29-CONFIG-PRECEDENCE.md`).
**Symptom:** ALLOW at `public` after reset; or deny all MEASURE (looks locked-down, bag empty).
**Cheapest guard:** `effective = min(seat, provider)` from typed fields, not line grep. Empty/unknown class ⇒ raise. Hash ceiling+allow-list; packaged empty ⇒ ERROR.

## 21. `sudo`/`doas`/`ssh user cmd` ALLOW on the outer token

**Failure:** privilege or host hop. Inner command not classified. **Safe** remote exec.
**Trigger:** allow `ssh`; allow `sudo` for “ops”; `ssh host lock_publish`.
**Symptom:** remote tree mutated; local hook 0 denials; two hosts (`ADVERSARIAL-CRON-10M.md` §11).
**Cheapest guard:** outer hop is REFUSE unless the **remote** hook applies the same `class` and returns it. Local ALLOW of `ssh` is not admit.

## 22. Windows / POSIX argv disagreement

**Failure:** hook matches a reconstructed command line. `CreateProcess` parsing differs. ALLOW/DENY both confident.
**Trigger:** quoted paths with spaces; `^`; `.bat` wrap; `/c`.
**Symptom:** deny pattern on the pretty line; real argv writes. Or deny of MEASURE (ERROR→NONE).
**Cheapest guard:** use the API argv array on that OS. Unparseable line ⇒ REFUSE. No shell.

## 23. Hook allow-lists MEASURE verbs but `open` flags include write

**Failure:** `ledger_read` string, `O_RDWR` or append `released` (`IDEMPOTENT-EVENT-LEDGER.md`).
**Trigger:** “read” module that flushes HWM; peek that pop-deletes (`TASK-27-RETRY-SEMANTICS.md`).
**Symptom:** ALLOW MEASURE; `released` without Sig2; queue “complete.”
**Cheapest guard:** MEASURE may not write. Append/`O_RDWR` ⇒ MUTATE ⇒ need live claim. String “read” is not flags.

## 24. Ceiling / 402 / quota decided by matching error text in argv or stdout

**Failure:** hook “sees” `402` / `restricted` in a command or banner. Hop **allowed** or retry **denied** as policy.
**Trigger:** `echo 402`; argv includes a WO title; substring classifier (`ADVERSARIAL-QUOTA-RETRY.md` §18).
**Symptom:** overage hop; or MEASURE blocked; both look like policy.
**Cheapest guard:** admit is `rank(provider)≤rank(seat.ceiling)` (`TASK-13-FAILOVER-CEILING.md`). Quota is a completed `I` MEASURE. Stdout/argv substrings are not policy.

## 25. Dual hook: old string matcher still imported

**Failure:** successor `class(argv)` REFUSE; old hook ALLOW (or reverse). One path **safe**.
**Trigger:** `pip` old module (`LEGACY-15.md`); cron names the tombstone stub that returns `True` (`LEGACY-16.md`).
**Symptom:** split-brain; CI tests the new file; run path is the old symbol.
**Cheapest guard:** old hook raises. Grep CI for the retired symbol. SUPERSEDED without `successor_id` does not merge.

## 26. Exemption / operator “allow this argv” as a string in chat

**Failure:** `P` ACK or a grant line matching the command **overrides** REFUSE (`EVIDENCE-TIERING.md`).
**Trigger:** `LEGACY-22` row with `predicate` grep; mailbox ACK (`ADVERSARIAL-MAILBOX-QUEUE.md` §1); model “looks safe.”
**Symptom:** MUTATE without claim; grant exists as a filename (`LEGACY-05.md`).
**Cheapest guard:** exemption is a flushed row with `would_deny` recomputed on `admit`/`class`, not on a command string. Chat is not a hook. Expired ⇒ REFUSE.
