# LEGACY-02

Review checklist for a script that decides **two paths are equal on Windows**. Ten items. Fail the review if any is unchecked. Path equality is not `a == b` (`TASK-32-PATH-SAFETY.md`).

1. **Case-sensitivity trap.** Default NTFS is case-**in**sensitive (`C:\Repo\App.log` and `c:\repo\app.LOG` are one file). Python/Git/`==` are case-**sensitive**. A script that uses string equality will treat those as different and open a second writer, rotate the “other” name, or miss a lock (`LEGACY-01.md`). A directory with the Windows case-sensitive bit set is the opposite trap: folding case merges two real files. Record which rule the script uses and test both.

2. **Separators.** `/` and `\` must compare equal after a single lexical split, not after `os.path.normpath` of only one side.

3. **Drive letter and UNC.** `C:\a` vs `c:\a` vs `\\localhost\C$\a` vs `\\?\C:\a`. Equality must state the form it accepts; do not resolve then compare (`TASK-32-PATH-SAFETY.md`: resolve-first is the vulnerability).

4. **`.` / `..` / extra slashes.** Reject traversal on the **declared** strings first. Do not claim equality because `abspath` erased `..`.

5. **Trailing slash, trailing space, trailing dot.** Win32 often strips `foo.` and `foo ` ; the other path may still be a different string. Test `dir\` vs `dir`.

6. **Reserved device names.** `CON`, `NUL`, `COM1`, `LPT1` (and `CON.txt`) are not files. Equality with a reserved name is a reject, not `True`.

7. **8.3 short names.** `PROGRA~1` vs `Program Files` can be the same inode. String compare misses that; inode compare after an unsafe resolve can follow a junction **out**.

8. **Junctions / symlinks / subst.** Two strings may be one target or a jail escape. Compare **declared** paths for policy; use a post-join `relative_to(root)` only as a second check.

9. **Long-path prefix and ADS.** `\\?\` and `file:stream` change identity. The script must not drop the prefix in one argument only, and must not treat `file` and `file:zone.identifier` as equal without saying so.

10. **Tests on both separator styles.** Same pairs as item 1–6 with `/` and `\`. If the script only tests POSIX strings, the review fails.
