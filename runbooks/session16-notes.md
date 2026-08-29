# Session 16 — Bash Basics: Scripts, find, and the stdout/stderr Split

Date: 2026-08-23

A short teaching-focused session, not a lab-build session — the start of a
deliberate bash-scripting practice track. No infrastructure state changed
except a cosmetic `.bashrc` edit on pve-node1.

---

## 1. FIRST SCRIPT — SHEBANG + EXECUTE PERMISSION

A script is just a text file with two things that make it runnable:
- **A shebang line** at the top — `#!/bin/bash` — tells the OS which
  interpreter to run the file with.
- **Execute permission** — `chmod +x scriptname.sh`. Without it, the file
  can be read but not run.

```bash
#!/bin/bash
echo "Hello, $USER. Today is $(date)"
```
```bash
chmod +x hello.sh
./hello.sh
```

Concepts covered: `$USER` is an environment variable already set by the
shell; `$(date)` is command substitution — it runs `date` and drops its
output inline; `./` is required to run a script from the current directory,
since the current directory usually isn't in `$PATH`.

## 2. THE `find` SYNTAX GOTCHA

First attempt: `find / type -f -name readiness.py` — missing the `-` before
`type`. **Every `find` option needs its own leading dash.** Corrected:
```bash
find / -type f -name "readiness.py"
```

## 3. stdout vs. stderr — THE REAL LESSON OF THE SESSION

Piping `find` output to `grep` didn't hide the flood of "Permission denied"
lines. Why: every command has two separate output streams —
- **stdout** (stream 1) — normal results
- **stderr** (stream 2) — error messages

A pipe (`|`) only carries stdout to the next command. stderr prints straight
to the terminal regardless, bypassing the pipe entirely — that's why `grep`
couldn't filter it out.

**Fix:**
```bash
find / -type f -name "readiness.py" 2>/dev/null
```
`2>` redirects stream 2 (stderr) to `/dev/null` — a discard target; anything
written there is simply gone.

Related patterns for future reference:
- `1>/dev/null` — hides stdout instead (rarely useful)
- `> /dev/null 2>&1` — hides both streams; useful when only success/failure
  matters, not the output itself. **Order matters** — `2>&1` must come
  *after* the stdout redirect, since it means "send stderr wherever stdout
  is currently going," and that target has to already be set.

## 4. RECURSIVE CONTENT SEARCH (started, in progress)

Needed to search file *contents* (not filenames) for a term:
```bash
grep -rn "SEARCH_TERM" /path/to/search 2>/dev/null
```
- `-r` — recursive through subdirectories
- `-n` — show line numbers
- `2>/dev/null` — the same permission-denied noise fix from Part 3, now
  applied to a different command, reinforcing that it's a general shell
  concept, not something specific to `find`.

This search was started but its result wasn't confirmed by the end of the
session — open item for next time.

## LESSONS

- **Every `find` option needs its own leading dash** — a missing one breaks
  the whole invocation silently confusingly rather than with an obvious
  error.
- **A pipe only carries stdout, never stderr.** Anything that "shows up
  even though I piped it away" is almost always stderr leaking around the
  pipe, not a bug in the receiving command.
- **`2>/dev/null` is a general shell pattern, not a `find`-specific trick** —
  it applies to any command that can emit errors you want to suppress.

---
END. Open item carried forward: confirm the result of the recursive grep
search for the live/blocked string.
