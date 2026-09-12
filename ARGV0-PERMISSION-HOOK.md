# argv[0] permission hook

A pre-exec hook `H` that allow/denies by **`argv[0]` only** (basename or realpath of the first token). This checkout has no live hook. The design is the contract: why that projection is wrong, how wrappers shift the token, how to strip, and the denial that proves stripping did not widen authority (`ADVERSARIAL-CMD-HOOK.md`, `GATE-DEADLOCK.md`).

```
H0(argv) = class(basename(argv[0]))     # the bug
```

`timeout 10 python evil.py` has `argv[0] == timeout`. `H0` never sees `python`.

## Wrappers that shift the token

A **shift** means the program that mutates the estate is **not** `argv[0]`. Two families.

### Prefix wrappers (payload starts at some `argv[k]`, k≥1)

Peelable **only** if the wrapper binary’s digest is the shipped one and it does not change uid, mount ns, or loader env.

| Wrapper | How the token shifts | Consume (then recurse) |
|---|---|---|
| `timeout` | `timeout [opts] DURATION CMD …` | options (`-s`, `--signal`, `-k`, `--kill-after`, `--foreground`, `--preserve-status`, `-v`); next positional is duration; rest is CMD |
| `nohup` | `nohup CMD …` | no operands; rest is CMD |
| `env` | `env [opts] [NAME=VALUE]… CMD …` | `-i` / `-u NAME` / `-C DIR` / `-S` / `-` / assignments; first non-`=` token that is not an option is CMD |
| `nice`, `ionice`, `chrt`, `taskset`, `numactl`, `prlimit` | scheduler/affinity prefix | their options; rest is CMD |
| `stdbuf`, `eatmydata`, `chronic`, `setsid`, `flock`, `time`, `/usr/bin/time` | stdio / lock / timing prefix | their options; rest is CMD |
| `watch` | `watch [opts] CMD` | options; rest is CMD (often one shell string — treat as `sh -c`) |
| `strace`, `ltrace`, `perf`, `gdb --args`, `valgrind`, `rlwrap` | tracer prefix | options up to `--` / `--args`; rest is CMD |
| `command`, `exec` (shell) | builtin prefix | rest is CMD |
| `busybox <applet>` | applet name is `argv[1]` | if `argv[0]` digest is busybox, classify `argv[1]` as the wrapper or verb |

**Not peelable** (privilege, namespace, or host hop). Stripping these **widens** or hides authority. Default REFUSE unless a remote hook returns the same `class`:

`sudo`, `doas`, `su`, `runuser`, `ssh`, `chroot`, `unshare`, `nsenter`, `bwrap`, `firejail`, `docker`, `podman`, `systemd-run`, `systemd-nspawn`, `lxc-attach`, `nsenter`.

### Shell compounds (`cd &&` and kin) — CMD is not in argv

| Form | What `execve` sees | Shift |
|---|---|---|
| `cd DIR && CMD` | `argv[0]` is `sh`/`bash`; string is `-c` | CMD is **inside** `argv[2]`. No prefix strip reaches it. |
| `(cd DIR && CMD)` | same | same |
| `pushd DIR && CMD` | same | same |
| `CMD1 && CMD2`, pipes, `` `cmd` `` | same | inner argv never built |
| `env -C DIR CMD` | prefix form | this **is** `cd` without a shell; `-C` must be consumed as a cwd change, not ignored |
| `sh -c`, `bash -c`, `dash -c`, `zsh -c`, `eval`, `.` / `source` | interpreter + string | not a peelable wrapper |

`H0` on `sh` allow-lists the shell and never sees `python` (`ADVERSARIAL-CMD-HOOK.md` §7–8).

## Stripping algorithm

Operate on the **argv array**, not a joined line. After every rewrite, re-classify (`GATE-REMEDY-DEADLOCK.md`).

```
strip(argv) -> (cwd', env_delta, inner_argv) | REFUSE

1. If argv is empty → REFUSE
2. Identify argv[0] by realpath inode + sha256 == a shipped digest.
   Basename-only match → REFUSE (copy named timeout is not timeout).
3. If argv[0] is a privilege hop → REFUSE (do not peel).
4. If argv[0] is sh/bash/dash/zsh or the payload is -c/-m/stdin
   → REFUSE unless inner_argv is already a parsed exact VERB enum
     with declared paths (default: REFUSE; do not regex the string).
5. If argv[0] is a peelable wrapper (table above):
   consume that wrapper’s options/operands only (unknown flag → REFUSE)
   if env:
     any NAME in {LD_PRELOAD, LD_LIBRARY_PATH, PYTHONPATH, PATH, IFS}
       or -i / -u of those → REFUSE          # drop would widen
     -C DIR → cwd' = DIR (lexical + resolve; must stay in repo root)
   recurse on the remaining tokens
6. Else inner_argv = argv. Stop.
7. class(inner_argv) with cwd' and env_delta applied to path checks
   and open flags (GATE-DEADLOCK class: MEASURE / ACQUIRE / MUTATE / REFUSE).
```

`env -C /tmp &&` / `cd / && cmd` that resolves outside the repo → REFUSE (`TASK-32-PATH-SAFETY.md`: lexical first, then `relative_to(root)`).

## Denial test: stripping did not widen authority

**Widen** means some argv that `class(inner)` would REFUSE becomes ALLOW after peel, or a hop that adds privilege/loader control becomes ALLOW because the inner verb is MEASURE.

Let `D` be a denied payload (e.g. `python -c 'open("out","w")'`, or undeclared `cat >`). Let `A` be a MEASURE payload (`verify` + declared path, `O_RDONLY`). Let `W` range over peelable wrappers from the table, each instantiated with legal options (including `timeout 1s`, `nohup`, `env FOO=1`, `env -C <in-repo-dir>`).

```
# 1. Peel does not un-deny
for W in peelable:
    assert class(W ++ D) == REFUSE
    assert class(D) == REFUSE

# 2. Peel does not change a MEASURE decision (same authority, not more)
for W in peelable:
    assert class(W ++ A) == class(A) == MEASURE

# 3. Privilege / loader hops are not peelable — would widen if stripped
assert class(["sudo", *A]) == REFUSE
assert class(["env", "LD_PRELOAD=x.so", *A]) == REFUSE
assert class(["env", "PATH=/tmp", *A]) == REFUSE
assert class(["env", "-i", *A]) == REFUSE

# 4. cd && is not a prefix; shell string stays denied
assert class(["sh", "-c", "cd /tmp && python evil.py"]) == REFUSE
assert class(["sh", "-c", "cd repo && verify declared"]) == REFUSE
assert class(["env", "-C", "/etc", *A]) == REFUSE

# 5. Counterfactual H0 (argv[0] only) — these must not be the implementation
assert H0(["timeout", "1s", *D]) == ALLOW   # documents the bug
assert class(["timeout", "1s", *D]) == REFUSE
```

If test 1 fails, strip looked at `timeout`/`nohup`/`env` and dropped `D`. If test 3 or 4 fails, strip treated a hop or `cd` as transparent and **widened** who may run MEASURE. `H0` failing test 5 is expected; `class` must not.

**Rule:** `argv[0]` is not the command. Peel only shipped, non-privilege prefixes. `cd &&` and `-c` are not prefixes. After strip, `class` of the inner argv must be a subset of what `class` would have decided on that inner argv alone — never allow what the inner would refuse.
