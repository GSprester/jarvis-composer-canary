# gate remedy deadlock

A policy hook `H` refuses an argv, then prints a **copy-paste remedy**. The operator (or the next seat) runs that remedy. `H` refuses it too. The estate cannot apply the fix the gate itself named. Same family as `GATE-DEADLOCK.md` (predicate required before the command that publishes it) and `LEGACY-17.md` (the gate must not author what it accepts). Here the published object is the **error string**, not a stub file.

This checkout has no live hook. The design is the contract.

## Shape

```
inbound argv  →  class(argv) == REFUSE
             →  print(remedy)          # prose or a shell line
operator      →  exec(parse(remedy))
             →  class(remedy_argv) == REFUSE
```

Worked refuse:

```
H: basename(argv[0]) ∈ {python, pip, curl}  ⇒  REFUSE
H: "run: python3 -m pip install -r requirements.txt --user"
```

`python3` and the token `pip` hit the same matcher. The message is a second policy that was never classified.

Lock-shaped twin: `H` requires a live claim; the message says `run lock_publish --unit U`; `lock_publish` is also under `H` (`LEGACY-09.md`).

## Detection bug

**Policy is applied to the inbound argv only. The outbound remedy is not an argv under `H`.**

Two functions, two projections:

| | Matcher | Remedy formatter |
|---|---|---|
| Input | `execve` argv (or a joined line) | a format string |
| Projection | basename, substring, regex, deny-list | English + a suggested command |
| Tested as | “bad argv exits non-zero” | `assert "python3 -m pip" in stderr` |

The matcher never sees the remedy tokens. The formatter never calls `class`. A deny-list written for attackers (`python`, `pip`) is wide enough to include the repair tool. A require-lock-before-lock predicate is the same bug with a boolean instead of a string.

That is the detection bug: **loss of identity between the refused object and the prescribed object.** The hook measures a projection. The message names a different string. Nothing asserts they are the same command class.

## Why it survives review

- CI proves the **deny** and proves the **hint text**. It does not `exec` the hint through `H`.
- Security review owns the deny-list. UX review owns “actionable errors.” Neither owns the composition.
- Unblocking by **basename-allow** (`python`, `/usr/bin/*`) looks like the patch. It is the smuggle (`ADVERSARIAL-CMD-HOOK.md` §1–2). Review sees “MEASURE works again.”
- The deadlock presents as **0 violations** and a helpful stderr. Operators read “hook is working.”
- README-only refuse lists drift from the printed remedies (`LEGACY-15.md`). Grep of the hook module stays green.

## Minimal patch

One classifier. Remedies are **argv literals**, not prose. They must pass `class` before they may be printed.

```
VERBS = {verify, find, alive, peek, ledger_read, lock_publish}

class(argv) =
  MEASURE  if verb ∈ {verify, find, alive, peek, ledger_read}
           AND every path is declared or .lock/writer-<unit>
           AND open flags ⊆ {O_RDONLY}
  ACQUIRE  if verb is lock_publish AND CAS stale/absent after a completed probe
  MUTATE   if live claim for this unit AND path declared
  REFUSE   otherwise          # default; unknown verb is ERROR

remedy(kind) -> argv          # list[str], not a sentence
print_refuse(kind):
    cmd = remedy(kind)
    assert class(cmd) in {MEASURE, ACQUIRE}   # load-time + before print
    write(stderr, join(cmd))                  # display only
```

- `H` implements `class` on every `execve`, including the command the operator pastes.
- Self-test: for every `remedy(k)`, `class(remedy(k)) ≠ REFUSE`, and `exec` of that argv is not MUTATE without a claim.
- Do **not** patch by allow-listing binaries or by disabling `H` (`GATE-DEADLOCK.md`).
- `H` only reads and refuses. It does not mint a lock, canary, or stub so the remedy becomes true (`LEGACY-17.md`).

## Re-validate after every normalisation

Normalisation (PATH search, `realpath`, case-fold, `\\` → `/`, argv rewrite, shebang expand) produces a **different command** than the one `class` already judged. Check-then-rewrite is hash-then-normalise (`TASK-15-IDEMPOTENT-RECEIPT.md`, `TASK-32-PATH-SAFETY.md`): the verdict was about a pre-image; `execve` runs the post-image.

A “fix” that allow-lists after basename fold is how a forbidden binary is smuggled: copy `evil` to `./python`, `python3.12`, or a file whose resolved basename is `verify`; symlink `verify` → attacker inode; `PATH` cwd-first; `python -c '…'`; `--` insertion.

**After any normalisation, before `execve`, re-run the whole gate on the bytes that will run:**

1. **`class(argv')` again** on the post-normalise argv **array**, not a joined pretty line. First ALLOW does not inherit.
2. **Verb is still an exact enum member.** Basename / prefix / homoglyph after fold ⇒ REFUSE. `verify_and_write` is not `verify`.
3. **`argv'[0]` identity is the shipped publisher** for that verb: `realpath` inode **and** `sha256(file)` equal the declared digest. Basename match is not identity.
4. **Every path argument:** lexical reject (`..`, absolute escape, reserved names) on the **declared string**, then join, then `relative_to(root)` on the resolved path. Both must pass. Resolve-only is the smuggle.
5. **No new interpreter surface.** If normalisation inserted `-c` / `-m` / stdin / a shebang interpreter, that is a new argv: default REFUSE unless it **is** the named module path for an allowed verb.
6. **Open flags still ⊆ `{O_RDONLY}`** for MEASURE. A rewrite that opens `O_RDWR` is MUTATE.
7. **Env that changes the image** (`PATH`, `LD_PRELOAD`, `PYTHONPATH`) on ACQUIRE/MUTATE ⇒ REFUSE, or `PATH` is a sealed dir whose members are the declared digests.

If any of 1–7 fails, REFUSE. Do not print a remedy that would fail 1–7. Do not exec an image that passed `class` only before the rewrite.

**Rule:** The message may name only an argv `class` already allows. Whatever you exec after rewrite is a new argv; classify and digest **that**. Measure is read-only. Acquire is one named CAS. Mutate needs a live claim. Nothing else runs.
