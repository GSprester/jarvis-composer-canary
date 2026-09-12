# liveness predicate

Record `{pid, start}` at acquire (`LEGACY-08.md`). `start` = OS creation time, UTC seconds. Same process = that **incarnation**, not “pid exists.”

```
same  iff  pid_exists AND start_os(pid)==record.start AND record.start>=boot_at
```

ε ≤ `1/CLK_TCK` (Linux) or one FILETIME tick after convert. **Fail if you round to whole seconds:** two spawns in 1s collapse. Steal only when `same` is **false** after a **completed** query (`LEGACY-14.md`).

## Rank (strong → weak)

| Signal | Why | Failure if used alone or OR’d |
|---|---|---|
| **start_time** | Survives pid recycle; changes on exec-replace | Skip `boot_at`: pre-boot clock looks like the new boot’s process |
| **pid** | Slot id | Recycle: you wait on or trust a new process |
| **env token** | Seat-minted; not kernel identity | EACCES skipped as match; exec drops env while writer lives |
| **cmdline** | Display | Same `python worker.py` reused; Win case/`\` (`LEGACY-02.md`); truncate |

**Fail if cmdline/token OR start:** recycled pid + same image looks live. **Fail if pid+cmdline without start:** service restart never reclaims. Lock `seat` is not liveness. **Fail if mtime:** drain looks stale; `touch` looks live (`TASK-16-ONE-LEAD-LOCK.md`).

## Query error

| Error | `same` | Steal | Treat live |
|---|---|---|---|
| pid gone | **false** | yes (CAS those bytes) | no |
| timeout, EACCES / ERROR_ACCESS_DENIED (5), unparseable `/proc`, no `btime` | **UNKNOWN** | **no** | **no** |

**Fail if EACCES ⇒ same:** unseen recycle. **Fail if EACCES ⇒ stale:** steal a live holder. **Fail if timeout ⇒ false:** load spike looks like a mass death (`LEGACY-03.md`, `LEGACY-38.md`). Exclude self (`LEGACY-13.md`). Do not scrape others’ `environ`; start is enough.

## Python

```python
import os, sys, time
from pathlib import Path

class ProbeUnknown(Exception):
    pass

def boot_at_utc() -> float:
    if sys.platform == "win32":
        import ctypes
        g = ctypes.windll.kernel32.GetTickCount64
        g.restype = ctypes.c_uint64
        return time.time() - g() / 1000.0
    for line in Path("/proc/stat").read_text().splitlines():
        if line.startswith("btime "):
            return float(line.split()[1])
    raise ProbeUnknown("btime")

def linux_start_utc(pid: int) -> float:
    try:
        raw = Path(f"/proc/{pid}/stat").read_text()
    except FileNotFoundError:
        raise
    except OSError as e:
        raise ProbeUnknown(str(e)) from e
    r = raw.rfind(")")
    if r < 0:
        raise ProbeUnknown("stat")
    ticks = int(raw[r + 2 :].split()[19])  # field 22
    return boot_at_utc() + ticks / os.sysconf("SC_CLK_TCK")

def windows_start_utc(pid: int) -> float:
    import ctypes
    from ctypes import wintypes
    class FILETIME(ctypes.Structure):
        _fields_ = [("lo", wintypes.DWORD), ("hi", wintypes.DWORD)]
    k = ctypes.windll.kernel32
    h = k.OpenProcess(0x1000, False, pid)  # QUERY_LIMITED_INFORMATION
    if not h:
        err = ctypes.GetLastError()
        if err == 5:
            raise ProbeUnknown("access")
        if err == 87:  # ERROR_INVALID_PARAMETER: typically no such pid
            raise FileNotFoundError(pid)
        raise ProbeUnknown(err)
    c, e, kr, u = FILETIME(), FILETIME(), FILETIME(), FILETIME()
    try:
        if not k.GetProcessTimes(h, ctypes.byref(c), ctypes.byref(e),
                                 ctypes.byref(kr), ctypes.byref(u)):
            raise ProbeUnknown("times")
        return ((c.hi << 32) | c.lo) / 10_000_000 - 11_644_473_600
    finally:
        k.CloseHandle(h)

def start_utc(pid: int) -> float:
    return windows_start_utc(pid) if sys.platform == "win32" else linux_start_utc(pid)

def same_process(pid: int, start: float) -> bool | None:
    """True same, False not, None UNKNOWN (do not steal)."""
    try:
        if start < boot_at_utc():
            return False
        os_start = start_utc(pid)
    except FileNotFoundError:
        return False
    except (ProbeUnknown, OSError, ValueError, IndexError):
        return None
    clk = 1e7 if sys.platform == "win32" else float(os.sysconf("SC_CLK_TCK") or 100)
    return abs(os_start - start) < (1.0 / clk)
```

Windows: error **5 ⇒ UNKNOWN**, not gone. **Fail if OpenProcess 0 ⇒ dead:** ACCESS_DENIED looks stealable. Optional cmdline/token are **AND** after `True`, never substitutes.

**Rule:** `pid` picks the slot. `start` defeats recycle. Error is UNKNOWN. Cmdline and env do not vote.
