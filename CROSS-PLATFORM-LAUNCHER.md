# Cross-platform launcher

**Status:** design only.  
**Depends on:** `ARGV0-PERMISSION-HOOK.md`, `TASK-32-PATH-SAFETY.md`, `LEGACY-02.md`, `PROCESS-BOUNDARY-EXIT.md`.  
**Does not exist in this tree:** launcher binary, Windows/POSIX spawn.

A launcher `L` starts a declared command so the **same argv list** becomes the **same child** on Windows and POSIX. “Identically” means: same resolved image digest, same argument vector the child sees, same exit-class mapping (`PROCESS-BOUNDARY-EXIT.md`). It does **not** mean the same command-line **string**. `CreateProcess` is not `execve`. Reconstructing a string and hoping both shells parse it is the hazard (`ADVERSARIAL-CMD-HOOK.md` §22).

`L` must not invoke a shell. `system()`, `os.system`, `subprocess(..., shell=True)`, `cmd.exe /c`, `powershell -Command`, `sh -c` are refuse.

## Binary resolution

Input is an **argv vector** `A = [a0, a1, …]` plus a declared **image** (path or allow-listed name). Lexical-validate **before** resolve (`TASK-32-PATH-SAFETY.md`).

```
resolve(a0) → (image_path, image_digest)
```

| Rule | POSIX | Windows |
|---|---|---|
| Image identity | `execve(image_path, A)` | `CreateProcessW`: **`lpApplicationName` = image_path**, command line built from `A` (below). Do not let Windows parse the binary out of the string |
| Search | No implicit `PATH` unless `a0` is an allow-listed **name** resolved by a **declared** PATH list (same order both OS) | Same list. **Not** cwd-first. **Not** `PATHEXT` (`.exe` / `.bat` / `.cmd` / `.com` surprise) |
| Extension | as declared | If `a0` has no suffix, do **not** append `.exe`/`.bat` unless the handoff said so |
| Equality | `realpath` after lexical OK; must stay under root | Case/separator/`\\?\` per `LEGACY-02.md`; reserved `CON`/`NUL` refuse; junctions: `relative_to(root)` after join |
| Digest | `sha256` of the regular file (not symlink target outside root) | Same; refuse if image is `.bat`/`.cmd` (those **are** `cmd.exe`) |
| `argv[0]` | May be the declared display name; hook classifies **image_digest**, not basename (`ARGV0-PERMISSION-HOOK.md`) | Same. `timeout`-shaped prefixes are not the image |

Failed resolve / UNKNOWN look → do not spawn (`TASK-20`). A second image that matches by basename only is refuse (PATH hijack).

Canary: declared `image_path` with a space; `resolve` must still hash **that** file, not `C:\Program` + leftover.

## Shell-mangling hazards

These change **which** image runs or **how** argv splits. Any of them means `L` is not identical across OS.

| Hazard | What happens |
|---|---|
| `shell=True` / `system()` | POSIX `sh -c` word-split and glob; Windows `cmd.exe` `& \| <>^%!` |
| `cmd /c` + concatenated string | `foo.bat` vs `foo.exe`; `%PATH%`; caret-escape; newline injection |
| `sh -c` / `bash -c` | `cd &&` — verb not in argv (`ARGV0`); `$`, backticks, IFS |
| Building one string then split | Windows CRT rules ≠ POSIX `execve` argv. Spaces and `"` disagree (`ADVERSARIAL-CMD-HOOK.md` §22) |
| Implicit `PATH` / cwd | Different first hit per OS; Windows cwd is a search slot |
| `PATHEXT` | `verify` launches `verify.bat` |
| Unquoted `Program Files` | POSIX: two argv; Windows: `C:\Program` as image |
| `%VAR%` / `!VAR!` in args | `cmd` expands even inside some quotes |
| `^`, `&`, `|` in a `cmd` string | extra commands |
| `*` `?` `[` under `sh -c` | glob |
| NUL / CR / LF in an arg | POSIX truncates at NUL; Windows line split. **Refuse** these bytes |
| Wrapper peel of `env`/`timeout` without digest | image is no longer `a0` |

`L` never parses a user string as a shell. Operators who type a shell line must go through a tokenizer that **fails closed** on metacharacters it cannot put in argv.

## Quoting strategy (spaces and metacharacters)

**Source of truth is argv, not a string.** POSIX: pass `A` to `execve` / `posix_spawn`. No quotes are applied (quotes would be **bytes in the argument**).

Windows: `CreateProcessW` requires a command-line string. Encode `A` with **CommandLineToArgvW-compatible** rules so the child CRT sees the same `A`:

```
encode_win(A):
    for each arg:
        if arg is empty or contains space, tab, or ":
            wrap in "
            each \ immediately preceding a " or the closing " is doubled
            each " inside is backslash-escaped
        else:
            emit raw
        join with a single space
    lpApplicationName is set; first token must still match the image
```

Do **not** use `cmd` escaping (`^`, `%`). Do **not** wrap the whole line in `cmd /c "..."`.

**Refuse** (do not encode) if any arg contains `NUL`, `CR`, `LF`. Those cannot be identical on both OS.

Round-trip test (same `A` on both hosts, or a Windows encode/decode without spawn):

```
A includes:
  C:\Program Files\app\verify.exe
  /tmp/dir with spaces/verify
  --path
  out\file (name).txt
  arg_with_"quotes"
  trailing\
  & | < > ^ % ! $ ` * ?
```

POSIX child `argv == A`. Windows child after `CommandLineToArgvW(encode_win(A)) == A`. Image digest equals `resolve`. If encode/decode disagrees, `L` must not spawn.

Hook `class(A)` runs on the **vector**, after resolve, same as POSIX (`ARGV0` peel only for declared peelable wrappers with shipped digest). A reconstructed display string is not what the hook sees.

Parent reap uses the same exit map on both OS (`PROCESS-BOUNDARY-EXIT.md`). Windows error 5 on OpenProcess of the child is UNKNOWN, not “ok.”

## What this must not do

- `shell=True` “because Windows needs a string.”
- PATH/PATHEXT/cwd search that differs by OS.
- Let `CreateProcess` pick the image from an unquoted first token.
- Treat a successful `cmd /c` as a portable launch.

**Rule:** Resolve a declared image (lexical then digest), spawn with argv (`execve` / `lpApplicationName` + CRT-encoded line). No shell. Args with spaces and metacharacters stay one argv slot; NUL/CR/LF are refuse. Identity is the child’s argv and image digest, not the command-line string.
