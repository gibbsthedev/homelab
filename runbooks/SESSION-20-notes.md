# SESSION 20 — Cowrie Shell Realism: Source-Code Modifications

**Date:** 2026-08-30
**Box:** Hetzner Cloud VPS `ubuntu-4gb-hel1-2` (62.238.47.215)
**Cowrie root:** `/home/cowrie/cowrie` (user `cowrie`)
**Admin SSH:** port 2200. **Honeypot:** port 22.
**Scope:** six defects fixed in Cowrie's Python source, one phantom defect disproved,
one duplicate installation removed.

This session did not touch the fake filesystem (`fs.pickle`) or any decoy content.
Everything here is a change to Cowrie's **emulation code** — the shell's behavior
rather than its contents.

---

## 0. Why this session happened

The original goal was migrating to T-Pot for a more realistic shell and wider
attack surface. That was scoped and rejected:

| Option | Requirement | Verdict |
|---|---|---|
| T-Pot Standard | 8–16 GB RAM, 128 GB disk | Box has 3.7 GB / 38 GB — no |
| T-Pot Sensor | ~4 GB, no dashboard, ships to external Hive | Possible, but Wazuh already owns that role in the plan |
| T-Pot Mini | Fits 4 GB, reduced honeypot set | Loses the coverage that motivated the switch |
| Upsize VPS | CX33 (4 vCPU/8 GB) $9.99/mo | **Cost-Optimized unavailable in hel1** |
| Upsize VPS | CPX32 (4 vCPU/8 GB) $41.99/mo | Deferred — 4x cost for a lab box |

Decision: **stay hand-built, deepen the existing shell instead.** The reasoning
that settled it — building it yourself is where the transferable skill lives;
T-Pot is a turnkey Docker stack that hands you a dashboard and hides the internals.

High-interaction mode (real Docker containers as the backend) was scoped and
deliberately deferred: it changes the box's risk model from "attacker is trapped
in an emulator" to "attacker is in a real container that could theoretically be
escaped." Do it last, deliberately, not first.

---

## 1. Finding the real gaps — let the logs pick the targets

Do not guess which commands need work. Cowrie logs every command it could not
handle. That is a priority list ranked by real attacker frequency.

```bash
cd /home/cowrie/cowrie/var/log/cowrie
grep -h '"eventid":"cowrie.command.failed"' cowrie.json* | \
  grep -oP '"input":"\K[^"]+' | sort | uniq -c | sort -rn | head -30
```

- `grep -h` — suppress filename prefixes. Without `-h`, multi-file matches prepend
  `cowrie.json.2026-08-27:` to every line and corrupt the later `sed`/`sort` stages.
- `cowrie.json*` — the glob catches all rotated days. `cowrie.json` alone is today only.
- `"eventid":"cowrie.command.failed"` — the stable JSON key. `cowrie.log`'s human
  wording changes between releases; the JSON `eventid` does not.
- `grep -oP '"input":"\K[^"]+'` — `-o` prints only the match, `-P` enables PCRE,
  `\K` discards everything matched before it. Net effect: print the command string
  without the surrounding JSON. Cheaper than a `sed` with two substitutions.
- `sort | uniq -c | sort -rn` — the standard frequency idiom. First `sort` groups
  identical lines (required — `uniq` only collapses *adjacent* duplicates), `-c`
  counts, `sort -rn` orders by count descending.

### Result

```text
    871 system
    866 enable
    855 shell
    659 linuxshell
    242 lockr -ia .ssh
     74 enablelinuxshell
     20 ll
      9 l
      8 la
      6 /ip cloud print
      5 nano
      5 ip a
      4 cd..
```

---

## 2. The phantom: why the top 3,300 hits needed NO fix

`system`, `enable`, `shell`, `linuxshell`, `enablelinuxshell` dominate the list at
~3,300 combined hits. The instinct is to build them. That instinct is wrong.

These are **Mirai/Gafgyt-family handshake probes.** Before dropping a payload, the
botnet sends this sequence to determine what kind of device it landed on — a
Cisco-style embedded CLI, a router shell, or real Linux. `enable` and `system` are
IOS-style commands. They are **not valid on real Linux either.**

The only question that matters: does Cowrie's rejection *look* like real bash's
rejection?

```bash
grep -rn "command not found" src/cowrie/
```

```python
# src/cowrie/shell/honeypot.py:938
return f"-bash: {cmd}: command not found\n"
```

That is byte-identical to real bash — same `-bash:` prefix, same colon spacing,
same wording. **No fix needed.** A real Ubuntu box behaves exactly this way.

> **LESSON — the loudest signal is not the fault.** Same shape as the e1000e NIC
> bug: the noisy, high-count symptom was not the problem. Verify what "correct"
> actually looks like before building a fix for something that already works.

`cd..` (no space) falls in the same category — real bash treats it as one
unrecognized token. `lockr -ia .ssh` is the Outlaw/Shellbot anti-competition
lockout already documented in the KB, not a shell gap.

Genuine gaps after filtering: **`ip`, `ll`, `la`, `nano`.**

---

## 3. Writing `ip.py` — and why it imports from `ifconfig.py`

Cowrie ships `ifconfig.py` but has no `ip`. On modern Ubuntu, `iproute2` is standard
and `ifconfig` is often *not installed at all* — an admin or bot reaches for `ip a`
first. Its absence is a real fingerprinting tell.

### The consistency trap

`ifconfig.py` generates its MAC and IPv6 at **module import time**, once per Cowrie
process:

```python
HWaddr = f"{randint(0, 255):02x}:{randint(0, 255):02x}:..."
inet6 = f"fe{randint(0, 255):02x}::{randrange(111, 888):02x}:..."
```

If `ip.py` generated its own random MAC, then `ifconfig` and `ip a` would report
**different hardware addresses for the same interface in the same session.** That is
a harder tell than not having `ip` at all. The fix is to import, not regenerate.

```bash
cd ~/cowrie
cat > src/cowrie/commands/ip.py << 'EOF'
from __future__ import annotations

from cowrie.commands.ifconfig import HWaddr, inet6
from cowrie.shell.command import HoneyPotCommand

commands = {}


class Command_ip(HoneyPotCommand):
    def call(self) -> None:
        if not self.args or self.args[0] in ("a", "addr", "address"):
            self.do_addr_show()
        elif self.args[0] in ("l", "link"):
            self.do_link_show()
        else:
            self.errorWrite(f'Object "{self.args[0]}" is unknown, try "ip help".\n')

    def do_addr_show(self) -> None:
        ip = self.protocol.kippoIP
        bcast = ip.rsplit(".", 1)[0] + ".255"
        result = f"""1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether {HWaddr} brd ff:ff:ff:ff:ff:ff
    inet {ip}/24 brd {bcast} scope global dynamic eth0
       valid_lft forever preferred_lft forever
    inet6 {inet6} scope link
       valid_lft forever preferred_lft forever"""
        self.write(f"{result}\n")

    def do_link_show(self) -> None:
        result = f"""1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP mode DEFAULT group default qlen 1000
    link/ether {HWaddr} brd ff:ff:ff:ff:ff:ff"""
        self.write(f"{result}\n")


commands["/sbin/ip"] = Command_ip
commands["ip"] = Command_ip
EOF
```

- `from cowrie.commands.ifconfig import HWaddr, inet6` — **the whole point.** Binds
  to the same module-level objects `ifconfig` uses. Both commands now report
  identical hardware.
- `self.protocol.kippoIP` — Cowrie's configured public IP. Using it keeps `ip a`
  consistent with `ifconfig`'s `inet addr:` line.
- `qdisc fq_codel` — the modern Ubuntu default queueing discipline. `pfifo_fast` is
  the older default and would date the box.
- `commands["/sbin/ip"]` **and** `commands["ip"]` — Cowrie dispatches on the literal
  typed string. `/sbin/ip a` and `ip a` are different dict lookups; register both.

---

## 4. `ll` / `la` — the alias gap

Ubuntu's stock `/etc/skel/.bashrc` ships `alias ll='ls -alF'` **enabled**. Real users
have `ll`. Cowrie does not, and it never will via aliases:

```bash
grep -rn "alias" src/cowrie/commands/base.py
```

```python
# src/cowrie/commands/base.py:1141
commands["alias"] = Command_nop
```

`alias` is registered as a **no-op**. There is no alias resolution layer anywhere in
Cowrie. Building a general alias system is a large lift for little return; the
shortcut is to register the aliases as commands that inject the flags themselves.

```bash
cat >> src/cowrie/commands/ls.py << 'EOF'


class Command_ll(Command_ls):
    def call(self) -> None:
        self.args = ["-la"] + self.args
        super().call()


class Command_la(Command_ls):
    def call(self) -> None:
        self.args = ["-a"] + self.args
        super().call()


commands["ll"] = Command_ll
commands["la"] = Command_la
EOF
```

- `cat >>` — **append**, not `>`. A single `>` truncates `ls.py` and destroys the
  real `ls` implementation.
- `class Command_ll(Command_ls)` — subclass, so all the existing `-l` formatting,
  permission-bit rendering, and uid/gid resolution is inherited untouched.
- `self.args = ["-la"] + self.args` — prepend, don't replace. `ll /etc` must still
  receive `/etc`. `ls.py` parses with `getopt.gnu_getopt`, which handles the combined
  `-la` form correctly.
- `super().call()` — hand off to the real `ls` logic after injecting the flag.

---

## 5. The registration trap — why none of it worked

Restart, test, and everything still failed:

```text
root@app-prod-01:~# ip a
-bash: ip: command not found
root@app-prod-01:~# ll
-bash: ll: command not found
root@app-prod-01:~# ifconfig
eth0      Link encap:Ethernet  HWaddr 9b:de:fc:b7:4d:72   <-- still works
```

`ifconfig` working proves the restart succeeded and the service is healthy. This is
a **discovery** failure, not a crash.

```bash
grep -rn "commands.update\|command_modules" src/cowrie/shell/protocol.py
```

```python
# src/cowrie/shell/protocol.py:40-46
commands: ClassVar[dict] = {}
for c in cowrie.commands.command_modules:
    try:
        module = import_module(f"cowrie.commands.{c}")
        commands.update(module.commands)
    except ImportError:
        _log.failure("Failed to import command {cmd}", cmd=c)
```

**Cowrie does not auto-discover command files.** It iterates an explicit list,
`cowrie.commands.command_modules`, defined in `src/cowrie/commands/__init__.py`.
A new `.py` file in `commands/` that is not in that list is never imported.

Note the `except ImportError` — a broken module fails **silently** into the Twisted
log. It does not crash the service and it does not appear on the console.

### Diagnostic: reproduce the loader by hand

When a module will not load, run Cowrie's own loop with the exception printed:

```bash
source /home/cowrie/cowrie-env/bin/activate
python3 << 'EOF'
from importlib import import_module
import cowrie.commands

commands = {}
for c in cowrie.commands.command_modules:
    try:
        module = import_module(f"cowrie.commands.{c}")
        commands.update(module.commands)
    except ImportError as e:
        print("FAILED:", c, "-", e)

print("ip" in commands, "/sbin/ip" in commands, "ls" in commands)
EOF
```

This is the single most useful debugging tool in this section: it makes the silent
`except ImportError` loud.

### Narrowing further

```bash
python3 -c "from cowrie.commands import ip"          # no output = file is valid
python3 -c "import cowrie.commands as c; print('ip' in c.command_modules)"   # False
python3 -c "from importlib import import_module; print(import_module('cowrie.commands.ip').commands)"
# {'/sbin/ip': <class ...>, 'ip': <class ...>}  = the file's own dict is fine
```

Conclusion: the module is valid, its `commands` dict is correct, but `command_modules`
does not list it. Which contradicted `grep`, which *did* show it. That contradiction
was the real finding.

---

## 6. The duplicate installation — two Cowrie trees

`grep` said `"ip",` was on line 36. Python said it was not in the list. Both were
telling the truth about **different files.**

```bash
python3 -c "import cowrie.commands; print(cowrie.commands.__file__)"
# /home/cowrie/cowrie/src/cowrie/commands/__init__.py
```

The earlier `sed` edit had run at a prompt reading `root@ubuntu-4gb-hel1-2:~/cowrie#`.
As **root**, `~/cowrie` is `/root/cowrie` — a second, entirely separate copy of the
Cowrie source tree. The edit landed there. The service reads `/home/cowrie/cowrie`.

> **LESSON — `~` is user-relative, and `su` does not always change your directory.**
> This is the same failure class as the OpenClaw duplicate-gateway incident: a stale
> second copy of an application existing alongside the live one, with a command
> silently affecting the wrong instance. Check `whoami` and `pwd`, not just the
> hostname in the prompt.

### Proving it is dead before deleting

```bash
stat /root/cowrie/src/cowrie/commands/__init__.py
stat /home/cowrie/cowrie/src/cowrie/commands/__init__.py
grep -rl "/root/cowrie" /etc/systemd/system/ 2>/dev/null
crontab -l 2>/dev/null | grep root/cowrie
```

- `stat` — compare `Modify` and `Birth` times. `/root/cowrie`'s files were born at
  `13:10:23`, exactly when the misdirected `sed` ran. Nothing older, nothing else
  ever touched it.
- `grep -rl` on `/etc/systemd/system/` — `-l` lists only filenames. Confirms no unit
  file points at the stale tree.
- `crontab -l` — confirms no scheduled job references it.

Both checks clean:

```bash
rm -rf /root/cowrie
```

> Do not `rm -rf` a root-owned application tree on the strength of "it is probably
> unused." `stat` plus a reference search is thirty seconds and converts a guess
> into evidence.

### The actual fix, in the correct tree

```bash
su - cowrie
cd ~/cowrie
sed -i 's/    "ifconfig",/    "ifconfig",\n    "ip",/' src/cowrie/commands/__init__.py
grep -n '"ip"' src/cowrie/commands/__init__.py
```

- `sed -i` — edit in place.
- The `s/OLD/NEW/` anchors on `    "ifconfig",` **with its exact four-space indent**,
  keeping the list alphabetical and the insertion unambiguous.
- `\n    "ip",` — insert a new line after the anchor.
- The `grep` afterward is mandatory: it confirms the substitution landed **exactly
  once.** A sloppy pattern can match twice or zero times, and `sed -i` reports neither.

`ls` was already in `command_modules` (which is why plain `ls` always worked), so
the `ll`/`la` subclasses started working from the same restart.

---

## 7. Extending the keystroke dispatch for `nano`

`nano` is saved with `Ctrl+O` and exited with `Ctrl+X`. Cowrie routes control keys
through a **fixed dispatch table**, not by naming convention:

```python
# src/cowrie/shell/protocol.py
self.keyHandlers.update(
    {
        b"\x00": self.handle_NUL,
        b"\x01": self.handle_HOME,       # CTRL-A
        b"\x03": self.handle_CTRL_C,
        b"\x04": self.handle_CTRL_D,
        b"\x0b": self.handle_CTRL_K,
        b"\x0c": self.handle_CTRL_L,
        b"\x15": self.handle_CTRL_U,
        b"\x16": self.handle_CTRL_V,
        b"\x1b": self.handle_ESC,
    }
)
```

`\x0f` (Ctrl+O) and `\x18` (Ctrl+X) are absent. Defining `handle_CTRL_X` on a command
would do nothing — nothing routes to it. The bytes get absorbed into the input buffer
as garbage.

This required editing **shared infrastructure** that every command inherits. The
safety requirement: the new handlers must be **no-ops everywhere by default**, so
that only `nano` — which overrides them — behaves differently. Nothing else in the
honeypot changes.

Three classes need the methods, because `cmdstack[-1]` can be any of them:

| File | Class | Why it needs the method |
|---|---|---|
| `shell/protocol.py` | `HoneyPotInteractiveProtocol` | Receives the keystroke, delegates down |
| `shell/command.py` | `HoneyPotCommand` | Base for every command; default no-op |
| `shell/honeypot.py` | `HoneyPotShell` | Is `cmdstack[-1]` when **no** command is running |

Miss `HoneyPotShell` and pressing Ctrl+O at an idle prompt raises `AttributeError`
on every session — a far worse bug than the missing feature.

### The patch script pattern

`sed` is the wrong tool for multi-line Python blocks with significant indentation.
This pattern verifies **every** anchor across **all** files before writing **any** of
them:

```bash
cd ~/cowrie
python3 << 'PYEOF'
import pathlib

edits = [
    ("src/cowrie/shell/protocol.py",
     '''                b"\\x0e": self.handle_DOWN,  # CTRL-N
                b"\\x10": self.handle_UP,  # CTRL-P''',
     '''                b"\\x0e": self.handle_DOWN,  # CTRL-N
                b"\\x0f": self.handle_CTRL_O,  # CTRL-O
                b"\\x10": self.handle_UP,  # CTRL-P'''),

    ("src/cowrie/shell/protocol.py",
     '''                b"\\x16": self.handle_CTRL_V,  # CTRL-V
                b"\\x1b": self.handle_ESC,  # ESC''',
     '''                b"\\x16": self.handle_CTRL_V,  # CTRL-V
                b"\\x18": self.handle_CTRL_X,  # CTRL-X
                b"\\x1b": self.handle_ESC,  # ESC'''),

    ("src/cowrie/shell/protocol.py",
     '''    def handle_CTRL_V(self) -> None:
        pass

    def handle_ESC(self) -> None:''',
     '''    def handle_CTRL_V(self) -> None:
        pass

    def handle_CTRL_O(self) -> None:
        if self.cmdstack:
            self.cmdstack[-1].handle_CTRL_O()

    def handle_CTRL_X(self) -> None:
        if self.cmdstack:
            self.cmdstack[-1].handle_CTRL_X()

    def handle_ESC(self) -> None:'''),

    ("src/cowrie/shell/command.py",
     '''    def handle_TAB(self) -> None:
        pass

    def eofReceived(self) -> None:''',
     '''    def handle_TAB(self) -> None:
        pass

    def handle_CTRL_O(self) -> None:
        pass

    def handle_CTRL_X(self) -> None:
        pass

    def eofReceived(self) -> None:'''),

    ("src/cowrie/shell/honeypot.py",
     '''        self.showPrompt()

    def handle_TAB(self) -> None:''',
     '''        self.showPrompt()

    def handle_CTRL_O(self) -> None:
        pass

    def handle_CTRL_X(self) -> None:
        pass

    def handle_TAB(self) -> None:'''),
]

by_file = {}
for path, old, new in edits:
    by_file.setdefault(path, []).append((old, new))

# PASS 1 — verify every anchor everywhere before touching anything
for path, file_edits in by_file.items():
    text = pathlib.Path(path).read_text()
    for old, new in file_edits:
        count = text.count(old)
        if count != 1:
            print(f"ABORT: anchor found {count} times (expected 1) in {path}")
            print(repr(old[:80]))
            raise SystemExit(1)

# PASS 2 — only now write
for path, file_edits in by_file.items():
    p = pathlib.Path(path)
    text = p.read_text()
    for old, new in file_edits:
        text = text.replace(old, new)
    p.write_text(text)
    print(f"Patched {path}: {len(file_edits)} edit(s) applied")

print("All edits applied successfully.")
PYEOF
```

- **Two passes.** The first version of this script verified and wrote per-file, so an
  abort on file 2 left file 1 already modified. Splitting verification from writing
  makes it genuinely all-or-nothing across the whole change set.
- `text.count(old) != 1` — rejects both "anchor not found" (0) and "anchor ambiguous"
  (2+). An anchor matching twice would corrupt an unintended location.
- `\\x0e` in the Python heredoc — the shell heredoc is `'PYEOF'` quoted so no shell
  expansion occurs, but Python still processes `\x` escapes in a normal string. `\\x`
  keeps the literal two characters `\x` that appear in the source file.
- `if self.cmdstack:` — guard against an empty stack, mirroring the existing
  `handle_CTRL_C` implementation exactly.

### The whitespace lesson

This script aborted **four times** before succeeding. Every failure was a blank line
between methods that `sed -n 'X,Yp'` had displayed but which was easy to miss:

```bash
sed -n '/def handle_CTRL_V/,/def handle_ESC/p' src/cowrie/shell/protocol.py | cat -A
```

```text
    def handle_CTRL_V(self) -> None:$
        pass$
$                                       <-- this line
    def handle_ESC(self) -> None:$
```

- `cat -A` — show all non-printing characters. `$` marks end-of-line, `^I` marks tabs,
  trailing spaces become visible. **Use `cat -A` to build any exact-match anchor.**
  Reading `sed` output by eye is how four consecutive aborts happened.

> **LESSON — the abort is the feature.** Four failed runs cost minutes and left the
> source untouched every time. One un-verified `sed` could have silently corrupted
> shared shell infrastructure on a live honeypot.

---

## 8. `nano.py` — and the content-storage discovery

### Scoping: what attackers actually do with nano

A real nano is a full-screen, cursor-addressable editor — header bar, live status
line, arrow-key navigation. That is a large build.

But that is **not how nano gets used on a honeypot.** Attackers run `nano script.sh`,
paste a block of content, `Ctrl+O`, `Ctrl+X`. They use it as a paste-and-save
mechanism to stage a payload. Emulating that path faithfully covers the real use case.

### The discovery: `tee` writes files that cannot be read back

`tee.py` looked like the model to copy, since it writes stdin to a file:

```python
def write_to_file(self, data: bytes) -> None:
    self.writtenBytes += len(data)
    for outf in self.teeFiles:
        self.fs.update_size(outf, self.writtenBytes)
```

It only updates **size**. It never stores content. Checking what `file_contents`
actually reads:

```bash
grep -n "def file_contents" -A 20 src/cowrie/shell/fs.py
```

```python
if f[A_TYPE] == T_FILE and f[A_REALFILE]:
    return Path(f[A_REALFILE]).read_bytes()      # 1) honeyfs / fsctl `load`
if f[A_TYPE] == T_FILE and isinstance(f[A_CONTENTS], bytes):
    return bytes(f[A_CONTENTS])                  # 2) bytes in the pickle
# 3) zero-byte fallback
```

Two content sources: `A_REALFILE` (a pointer to a real file on disk — the mechanism
`fsctl load` uses for pre-planted decoys) and `A_CONTENTS` (raw bytes in the pickle).
`tee` sets neither.

**Consequence:** `echo hello | tee out.txt && cat out.txt` shows a non-zero size in
`ls -l` and returns **nothing** from `cat`. That is a pre-existing Cowrie realism bug,
found incidentally, logged as an open item.

For live content created during a session, `A_CONTENTS` is correct. `A_REALFILE`
would require writing to the real disk.

### Confirming the index

`mkfile` appends a 10-element list. `A_CONTENTS` had to be the right index or the
write would silently corrupt a different field:

```bash
sed -n '/^(/,/list(range/p' src/cowrie/shell/fs.py
```

```python
(
    A_NAME, A_TYPE, A_UID, A_GID, A_SIZE,
    A_MODE, A_CTIME, A_CONTENTS, A_TARGET, A_REALFILE,
) = list(range(0, 10))
```

`A_CONTENTS = 7`, matching `mkfile`'s node layout position-for-position. Confirmed,
not assumed.

### Adding `update_contents` to the filesystem

```bash
python3 << 'PYEOF'
import pathlib

old = '''    def update_size(self, filename: str, size: int) -> None:
        self._ensure_private_fs()
        f: Node | None = self.getfile(filename)
        if not f:
            return
        if f[A_TYPE] != T_FILE:
            return
        f[A_SIZE] = size'''

new = old + '''

    def update_contents(self, filename: str, contents: bytes) -> None:
        self._ensure_private_fs()
        f: Node | None = self.getfile(filename)
        if not f:
            return
        if f[A_TYPE] != T_FILE:
            return
        f[A_CONTENTS] = contents
        f[A_SIZE] = len(contents)'''

p = pathlib.Path("src/cowrie/shell/fs.py")
text = p.read_text()
if text.count(old) != 1:
    print("ABORT"); raise SystemExit(1)
p.write_text(text.replace(old, new))
print("Patched fs.py")
PYEOF
```

- Mirrors `update_size` exactly — `_ensure_private_fs()` first (copy-on-write for the
  pickle), `getfile()` returns the live node **by reference**, mutate by index.
- Sets `A_SIZE` alongside `A_CONTENTS` so `ls -l` and `cat` agree. Setting content
  without size is how `tee` ended up inconsistent.

### The command

```bash
cat > src/cowrie/commands/nano.py << 'EOF'
from __future__ import annotations

import posixpath

from cowrie.shell.command import HoneyPotCommand
from cowrie.shell.fs import FileNotFound

commands = {}


class Command_nano(HoneyPotCommand):
    def start(self) -> None:
        if not self.args:
            self.errorWrite("Usage: nano [file]\n")
            self.exit()
            return

        self.filename = self.args[0]
        self.pname = self.fs.resolve_path(self.filename, self.cwd)
        self.lines: list[str] = []
        self.awaiting_filename = False

        if self.fs.exists(self.pname):
            try:
                existing = self.fs.file_contents(self.pname)
                if existing:
                    self.lines = existing.decode("utf-8", errors="replace").split("\n")
                    if self.lines and self.lines[-1] == "":
                        self.lines.pop()
            except Exception:
                self.lines = []

        self.write(f"  GNU nano 7.2                {self.filename}\n\n")
        for line in self.lines:
            self.write(line + "\n")

    def lineReceived(self, line: str) -> None:
        if self.awaiting_filename:
            self.awaiting_filename = False
            target = line.strip() or self.filename
            self.do_save(target)
            return
        self.lines.append(line)

    def do_save(self, target_name: str) -> None:
        pname = self.fs.resolve_path(target_name, self.cwd)
        folder = posixpath.dirname(pname)
        if not self.fs.isdir(folder):
            self.write(f"\n[ Error writing {target_name}: No such directory ]\n")
            return
        if not self.fs.exists(pname):
            try:
                self.fs.mkfile(pname, self.user["uid"], self.user["gid"], 0, 0o644)
            except FileNotFound:
                self.write(f"\n[ Error writing {target_name} ]\n")
                return

        # An empty document is zero bytes, not a bare newline -- only join a
        # trailing "\n" onto real content, never onto nothing.
        if self.lines:
            content = ("\n".join(self.lines) + "\n").encode("utf-8")
        else:
            content = b""

        self.fs.update_contents(pname, content)
        self.pname = pname
        self.filename = target_name
        self.write(f"\n[ Wrote {len(self.lines)} lines ]\n")

    def handle_CTRL_O(self) -> None:
        # Real nano interrupts whatever is mid-typed to show the save prompt --
        # clear the pending buffer so it does not bleed into the filename answer.
        self.protocol.lineBuffer = []
        self.protocol.lineBufferIndex = 0
        self.write(f"File Name to Write: {self.filename}")
        self.awaiting_filename = True

    def handle_CTRL_X(self) -> None:
        self.protocol.lineBuffer = []
        self.protocol.lineBufferIndex = 0
        self.exit()

    def eofReceived(self) -> None:
        self.exit()


commands["/bin/nano"] = Command_nano
commands["nano"] = Command_nano
EOF

sed -i 's/    "nc",/    "nano",\n    "nc",/' src/cowrie/commands/__init__.py
grep -n '"nano"' src/cowrie/commands/__init__.py
```

- `def start()` not `def call()` — `start()` runs and **does not exit**, leaving the
  command on the stack to keep receiving input. Same pattern `python.py` uses for its
  fake REPL. A `call()`-based command exits when it returns.
- `lineReceived` override — captures each line as **content** instead of letting it
  return to the shell as a command. This is what "being inside an editor" means here.
- `self.awaiting_filename` — a mode flag. After Ctrl+O, the next line is the answer to
  the filename prompt, not document text.
- `line.strip() or self.filename` — bare Enter accepts the default; typing a different
  name is a real save-as.
- `if self.lines: ... else: content = b""` — **the empty-file bug.** The original wrote
  `("\n".join([]) + "\n")` = `"\n"`, giving every empty file a phantom blank line that
  then loaded back as real content on reopen. An empty document must be zero bytes.
- `self.protocol.lineBuffer = []` in **both** handlers — mid-typed characters that were
  never Enter-committed live in the raw terminal buffer. Without the reset they bleed
  into the next shell command (observed as `sheofhcat test.txt` → `sheofhcat: command
  not found`). `handle_CTRL_C` in `honeypot.py` does exactly this; `handle_CTRL_X` must
  too.
- `0o644` — octal. See §10 for what happens when this is written as decimal.

---

## 9. `ls` column formatting

Real `ls` output looked wrong: enormous inconsistent gaps between short filenames.

```python
# original do_ls_normal
maxlen = max(len(x) for x in line)
perline = int(wincols / (maxlen + 1))
self.write(f.ljust(maxlen + 1))
```

**One global width, applied to every filename.** With
`api_credentials_dump.json` (25 chars) in the directory, `backups` (7 chars) gets
padded to 26. GNU `ls` instead lays out **down-then-across** with each column sized to
its own longest entry.

```python
    @staticmethod
    def columnize(names: list, wincols: int) -> str:
        n = len(names)
        if n == 0:
            return "\n"
        spacing = 2

        def col_widths(cols: int):
            rows = -(-n // cols)
            widths = []
            for c in range(cols):
                chunk = names[c * rows : (c + 1) * rows]
                if not chunk:
                    break
                widths.append(max(len(x) for x in chunk))
            return widths

        best_widths = [max(len(x) for x in names)]
        for cols in range(min(n, wincols), 0, -1):
            widths = col_widths(cols)
            if not widths:
                continue
            total = sum(widths) + spacing * (len(widths) - 1)
            if total <= wincols:
                best_widths = widths
                break

        best_cols = len(best_widths)
        rows = -(-n // best_cols)
        lines = []
        for r in range(rows):
            row_items = []
            for c in range(best_cols):
                idx = c * rows + r
                if idx < n:
                    row_items.append((names[idx], best_widths[c]))
            cells = []
            for i, (name, width) in enumerate(row_items):
                if i == len(row_items) - 1:
                    cells.append(name)
                else:
                    cells.append(name.ljust(width + spacing))
            lines.append("".join(cells))
        return "\n".join(lines) + "\n"
```

- `-(-n // cols)` — ceiling division without importing `math`. `//` floors; negating
  both sides converts it to a ceiling.
- `names[c * rows : (c + 1) * rows]` — slices the **column**, because GNU `ls` fills
  down first. `idx = c * rows + r` is the same mapping in reverse when emitting rows.
- Descending `range(min(n, wincols), 0, -1)` — try the most columns first, take the
  first layout that fits the terminal width. That is GNU `ls`'s actual algorithm.
- `if i == len(row_items) - 1: cells.append(name)` — no padding on the last column.
  Trailing whitespace at end-of-line is itself a subtle tell.

---

## 10. Two bugs found by testing, not by reading

### The duplicate home directory

Logging in twice as the same non-existent user produced **two identically-named
directories** in `/home`:

```text
drwxr-xr-x 1 root  root  4096 2026-08-29 17:20 poop
d-wxrw--wt 1 5015  5015  4096 2026-08-30 17:04 poop
```

Real Linux cannot do this. Root cause:

```python
# fs.py mkfile -- HAS a dedupe check
if outfile in [x[A_NAME] for x in _dir]:
    _dir.remove(next(x for x in _dir if x[A_NAME] == outfile))
_dir.append([...])

# fs.py mkdir -- NO dedupe check
directory.append([posixpath.basename(path), T_DIR, ...])
```

`mkdir` appends blindly. `session.py` calls it on every login for a temporary user.
Fix — mirror `mkfile`'s pattern:

```python
        dirname = posixpath.basename(path)
        if dirname in [x[A_NAME] for x in directory]:
            directory.remove(next(x for x in directory if x[A_NAME] == dirname))
        directory.append(
            [
                dirname,
                T_DIR,
                ...
```

- `next(x for x in directory if ...)` — generator expression, stops at the first match
  without building an intermediate list.
- Fixed at the **filesystem layer**, not in `session.py`, so every future `mkdir`
  caller is protected, not just the login path.

### The permission bug

Every auto-created home directory showed `d-wxrw--wt` instead of `drwxr-xr-x`:

```python
# src/cowrie/shell/session.py
self.server.fs.mkdir(self.avatar.home, self.uid, self.gid, 4096, 755)
```

`755` is **decimal**, not octal. Decimal 755 = octal 1363 — which sets the sticky bit
and scrambles the permission triads into exactly the observed garbage. Every
temporary user's home directory has been wrong.

```python
self.server.fs.mkdir(self.avatar.home, self.uid, self.gid, 4096, 0o755)
```

Two characters. Verified with a fresh username (an existing user keeps its
already-created directory).

> **LESSON — permission literals are always octal.** `755` and `0o755` are both valid
> Python integers, so nothing errors; the mode is just silently wrong. Same trap as
> `chmod` in C.

---

## 11. `cd -` and OLDPWD

```python
# ORIGINAL -- src/cowrie/commands/fs.py
if pname == "-":
    self.errorWrite("bash: cd: OLDPWD not set\n")
    return
```

Unconditional. `cd -` **never** worked — there was no OLDPWD tracking anywhere.

Real bash keeps `OLDPWD` and `PWD` in the environment. Cowrie already has a per-session
environment dict (`session.py` populates `HOME`, `USER`, `PATH`, …), and
`command.py:60` binds it by reference:

```python
self.environ = self.protocol.cmdstack[-1].environ
```

So state written there persists across commands in the session.

```python
    def call(self) -> None:
        print_target = False
        if self.args and self.args[0] == "-":
            oldpwd = self.environ.get("OLDPWD")
            if not oldpwd:
                self.errorWrite("bash: cd: OLDPWD not set\n")
                return
            pname = oldpwd
            print_target = True
        elif not self.args or self.args[0] == "~":
            pname = self.user["home"]
        else:
            pname = self.args[0]
        newpath = ""
        try:
            newpath = self.fs.resolve_path(pname, self.cwd)
            inode = self.fs.getfile(newpath)
        except Exception:
            inode = None
        if inode is None or inode is False:
            self.errorWrite(f"bash: cd: {pname}: No such file or directory\n")
            return
        if inode[fs.A_TYPE] != fs.T_DIR:
            self.errorWrite(f"bash: cd: {pname}: Not a directory\n")
            return
        self.environ["OLDPWD"] = self.shell.cwd
        self.shell.cwd = newpath
        self.environ["PWD"] = newpath
        if print_target:
            self.write(newpath + "\n")
```

- `self.environ.get("OLDPWD")` — the error message is now **conditional.** First `cd -`
  of a session still prints `OLDPWD not set`, exactly like real bash.
- `print_target` — real bash echoes the destination path on `cd -` (and only on `cd -`).
- `self.environ["OLDPWD"] = self.shell.cwd` **before** reassigning `cwd` — ordering
  matters; after the reassignment the old value is gone.
- Set **after** all validation. A failed `cd` must not clobber OLDPWD.
- `self.environ["PWD"]` — kept in sync so `echo $PWD` agrees with the prompt.

---

## 12. Verification

`py_compile` proves syntax only. It does not execute the code, so a `NameError` from
an undefined constant would pass it. Every change was verified by **function.**

```bash
python3 -m py_compile src/cowrie/shell/protocol.py \
                      src/cowrie/shell/command.py \
                      src/cowrie/shell/honeypot.py \
                      src/cowrie/shell/fs.py \
                      src/cowrie/shell/session.py \
                      src/cowrie/commands/fs.py \
                      src/cowrie/commands/ls.py \
                      src/cowrie/commands/ip.py \
                      src/cowrie/commands/nano.py
echo "Compile check: $?"
```

```bash
exit                      # back to root
systemctl restart cowrie
systemctl is-active cowrie
```

### Regression check — did shared-infrastructure edits break anything?

From a **fresh** session on port 22:

| Test | Expected |
|---|---|
| `ls`, tab-completion | unchanged |
| `Ctrl+C` mid-typing | clears line, redraws prompt |
| `Ctrl+L` | clears screen, cursor home |
| `ifconfig` | unchanged |

### Feature checks

```bash
ssh root@62.238.47.215

ip a            # link/ether MUST equal ifconfig's HWaddr in the same session
ll
la

nano blank.txt  # Ctrl+O, Enter
cat -A blank.txt          # MUST be empty -- no lone "$"

nano notes.txt
hello
hi
                # Ctrl+O -> "File Name to Write: notes.txt", Enter, Ctrl+X
cat notes.txt   # hello / hi
ls              # no bleed-through from the editor buffer

cd /tmp
cd /root
cd -            # prints /tmp
pwd
cd -            # toggles back to /root
```

Duplicate-home check — log in twice as a **new** username:

```bash
ssh testuser1@62.238.47.215
ls -la /home    # exactly ONE testuser1, mode drwxr-xr-x
```

---

## 13. Files changed

| File | Change |
|---|---|
| `src/cowrie/commands/ip.py` | **NEW** — `ip a` / `ip addr` / `ip link`, MAC imported from `ifconfig.py` |
| `src/cowrie/commands/nano.py` | **NEW** — editor with Ctrl+O save prompt, Ctrl+X exit, real content storage |
| `src/cowrie/commands/__init__.py` | Registered `ip` and `nano` in `command_modules` |
| `src/cowrie/commands/ls.py` | Added `Command_ll` / `Command_la`; rewrote `do_ls_normal` with GNU-style columnization |
| `src/cowrie/commands/fs.py` | `Command_cd` — real OLDPWD tracking, working `cd -` |
| `src/cowrie/shell/fs.py` | Added `update_contents()`; `mkdir()` dedupe |
| `src/cowrie/shell/session.py` | `755` → `0o755` home directory mode |
| `src/cowrie/shell/protocol.py` | Ctrl+O / Ctrl+X dispatch entries + delegating handlers |
| `src/cowrie/shell/command.py` | Default no-op `handle_CTRL_O` / `handle_CTRL_X` |
| `src/cowrie/shell/honeypot.py` | Default no-op `handle_CTRL_O` / `handle_CTRL_X` on the idle shell |
| `/root/cowrie` | **DELETED** — stale duplicate source tree |

---

## 14. Lessons

- **Let the logs pick the targets.** `cowrie.command.failed` frequency is a
  priority list built from real adversary behavior.
- **The loudest signal is not the fault.** 3,300 hits on `enable`/`system`/`shell`
  needed zero code — real bash rejects them identically. Verify what "correct" looks
  like before building a fix.
- **`~` is user-relative.** `sudo`/`su` without changing directory is how an edit
  lands in `/root/cowrie` while the service reads `/home/cowrie/cowrie`. Check
  `whoami` and `pwd`.
- **Prove a duplicate is dead before deleting it.** `stat` birth/modify times plus a
  reference search through systemd and cron.
- **Silent `except ImportError` hides broken modules.** Reproduce the loader loop by
  hand with the exception printed.
- **Cowrie does not auto-discover commands.** New files must be added to
  `command_modules` in `src/cowrie/commands/__init__.py`.
- **Verify anchors with `cat -A`.** Four consecutive aborts were all invisible blank
  lines. Reading `sed` output by eye is not sufficient for exact-match patching.
- **Separate verification from writing.** Check every anchor in every file before
  modifying any of them, so an abort leaves nothing half-changed.
- **Permission literals are octal.** `755` and `0o755` both compile; only one is right.
- **Fix at the lowest correct layer.** `mkdir` dedupe belongs in `fs.py`, not in the
  login path, so all future callers inherit it.
- **Test like a careless attacker.** The buffer bleed-through and the phantom blank
  line only appeared because Ctrl+O was pressed before Enter.
- **`py_compile` is a gate, not proof.** It parses; it does not execute. Verify by
  function.

---

## 15. Open items created by this session

- **`tee` writes unreadable files** — sets `A_SIZE` but never `A_CONTENTS` or
  `A_REALFILE`, so `cat` returns nothing for a file `ls -l` shows as non-empty. Same
  `update_contents()` fix applies.
- **`nano` is line-oriented** — no cursor movement, no arrow keys, no live status bar.
  Sufficient for the paste-and-save workflow attackers actually use.
- **`ip` subcommand coverage** — only `addr`/`link`. `ip route`, `ip neigh`, and
  `ip -s` are unimplemented and will fall through to the unknown-object error.
- **`/etc/skel` and `.bashrc`** — auto-created home directories are empty; a real
  Ubuntu user home would contain `.bashrc`, `.profile`, `.bash_logout`.
- **Upstream contribution** — the `mkdir` dedupe and the `0o755` mode bug are genuine
  upstream Cowrie defects, not local-only issues. Worth a PR.

---

## 16. Next session

Attack surface expansion, per the sequence agreed this session:

1. **Dionaea** — SMB/RDP-style honeypot, captures malware binaries directly. A
   different interaction model and log format from Cowrie; new correlation work.
2. **High-interaction Cowrie backend** — real Docker containers as the shell. Gated on
   the VPS upsize (needs headroom for a container pool) and on hardened container
   isolation, since it materially changes the box's risk model.

Capacity note: 3.7 GB total / ~2.1 GB available, 2 vCPU, 32 GB disk free, 517 MB swap
already in use at idle. Docker daemon overhead alone is ~150–250 MB. The CX33 upsize
($9.99/mo) is unavailable in hel1; CPX32 is $41.99/mo. Migrating to a
Cost-Optimized-capable location would mean a new IP and a full honeypot rebuild.
