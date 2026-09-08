# SESSION 26 — Dionaea Disk-Full Incident: Diagnosis, Data Recovery, Root-Cause Fix

**Date:** 2026-09-08
**Box:** Hetzner Cloud VPS `ubuntu-4gb-hel1-2` (62.238.47.215)
**Continuity:** Follows `SESSION-22-dionaea-deployment-and-traffic-analysis-2026-08-31.md`.
**Trigger:** Cowrie sessions failing with unhandled Python tracebacks. Root cause
turned out to have nothing to do with Cowrie.

**Scope:** one honeypot-stack outage, diagnosed to its actual root cause; one
opportunistic forensic review of eight days of accumulated Dionaea capture data
(entirely because the fix required looking at it first); one confirmed, permanent
config fix; one recurrence-prevention safety net.

---

## 0. The presenting symptom — and why it was a red herring

```text
Error executing command "b'/bin/./uname -s -v -n -r -m'"
Traceback (most recent call last):
  File ".../twisted/conch/ssh/session.py", line 98, in request_exec
    self.session.execCommand(pp, f)
  File "cowrie/shell/session.py", line 107, in execCommand
    self.protocol.makeConnection(processprotocol)
  File ".../twisted/internet/protocol.py", line 564, in makeConnection
    self.connectionMade()
  File "cowrie/insults/insults.py", line 83, in connectionMade
    ttylog.ttylog_open(self.ttylogFile, self.startTime)
```

Read at face value, this looks like a Cowrie bug in the non-interactive `exec`
code path (`ssh host command`, distinct from the interactive shell — a different
entry point than anything patched in session 20). It was not. The traceback was cut
off mid-stack by the terminal pane; the actual exception — always the *last* line of
a Python traceback, not the call chain above it — was never visible in the first
paste.

```bash
grep -c "Traceback" cowrie.log
ls -la /home/cowrie/cowrie/var/lib/cowrie/tty/ | head -5
df -h /
```

```text
/dev/sda1        38G   36G     0 100% /
```

**Root cause, found in one line: the disk was completely full.** `ttylog_open`
couldn't create a new file because there was nowhere left to write one — Cowrie's
own log two lines above the traceback even said so verbatim: `[Errno 28] No space
left on device`. Cowrie was not broken. It had nowhere left to write.

> **LESSON — read the whole traceback, especially the part that got scrolled past.**
> The exception type/message is always the last line. Everything above it is the call
> path *to* the failure, not the failure itself. A partial paste that stops at a
> library-internals frame is missing the one line that actually says what's wrong.

---

## 1. Finding the actual disk hog — top-down, not guesswork

```bash
du -h --max-depth=1 / 2>/dev/null | sort -rh | head -15
```

```text
35G     /
30G     /opt
2.1G    /usr
1.7G    /home
```

`/opt` was the target. One level down:

```bash
du -h --max-depth=2 /opt 2>/dev/null | sort -rh | head -20
```

```text
30G     /opt/dionaea/var
30G     /opt/dionaea
```

All of it inside Dionaea. Checked the obvious suspects in parallel rather than
assuming — Docker's own storage was one candidate, and Dionaea's captured-binaries
directory was another:

```bash
du -sh /home/cowrie/cowrie/var/lib/cowrie/downloads/   # 1.2G
du -sh /home/cowrie/cowrie/var/lib/cowrie/tty/          # 219M
du -sh /opt/dionaea/var/lib/dionaea/binaries/           # 207M
sudo du -sh /var/lib/docker/                            # 202M
```

Docker: clean. Binaries: only 207 MB — not the culprit despite being the most
"obvious" Dionaea directory. One more level down pinned it exactly:

```bash
du -h --max-depth=3 /opt/dionaea/var 2>/dev/null | sort -rh | head -20
```

```text
30G     /opt/dionaea/var
29G     /opt/dionaea/var/log/dionaea
1.1G    /opt/dionaea/var/lib/dionaea
905M    /opt/dionaea/var/lib/dionaea/bistreams
```

**`bistreams`** — Dionaea's raw two-way byte-stream capture for every connection —
was the leading suspect going in (it's structurally unbounded by design). It turned
out to be a red herring too, at a comparatively modest 905 MB. The actual sink:

```bash
ls -la /opt/dionaea/var/log/dionaea/
```

```text
-rw-r--r-- 1 root root       2809856 dionaea-errors.log
-rw-r--r-- 1 root root   30755893248 dionaea.log
```

**`dionaea.log` alone: 30,755,893,248 bytes (≈28.6 GiB).** Essentially the entire
disk, in one plain-text log file.

> **LESSON — the most "obviously unbounded" directory is not automatically the
> answer.** `bistreams` looked like the likely culprit by design; the actual cause
> was a boring, ordinary application log with no rotation. Verify with `du`
> top-down at each level rather than jumping straight to the theoretically riskiest
> component.

---

## 2. Decision: read the data before truncating

The reflexive move here is `truncate -s 0` immediately and move on. That was
deliberately delayed. Eight days of Dionaea capture data had never been reviewed —
only ever spot-checked once, the night it was deployed (session 22). Truncating the
log without checking first risks discarding something irretrievable. The structured
database (`dionaea.sqlite`) is a much smaller, purpose-built haystack than 28 GB of
raw debug text, so it was checked first:

```bash
sqlite3 /opt/dionaea/var/lib/dionaea/dionaea.sqlite "select count(*) from connections;"    # 41,369
sqlite3 /opt/dionaea/var/lib/dionaea/dionaea.sqlite "select count(*) from downloads;"        # 73
sqlite3 /opt/dionaea/var/lib/dionaea/dionaea.sqlite "select count(*) from offers;"            # 2
sqlite3 /opt/dionaea/var/lib/dionaea/dionaea.sqlite "select count(*) from emu_profiles;"      # 2
sqlite3 /opt/dionaea/var/lib/dionaea/dionaea.sqlite "select count(*) from logins;"             # 2,492
sqlite3 /opt/dionaea/var/lib/dionaea/dionaea.sqlite "select count(*) from dcerpcrequests;"     # 203
sqlite3 /opt/dionaea/var/lib/dionaea/dionaea.sqlite "select count(*) from mssql_fingerprints;" # 283
sqlite3 /opt/dionaea/var/lib/dionaea/dionaea.sqlite "select count(*) from mysql_commands;"     # 1,300
```

This flatly contradicted the "probably just recon" assumption carried over from
session 22's single-night snapshot (32 connect/disconnect-only SMB probes, zero
downloads, zero emu_profiles). Eight days of real internet exposure produced a
completely different picture. Nonzero counts outside `connections` meant real
capture data existed and had to be pulled before the log was touched.

> **LESSON — one night of data is not representative.** Session 22's finding
> ("recon-only") was accurate *for that night*. Assuming it still held eight days
> later, without checking, would have been exactly the kind of unverified carry-over
> assumption this lab keeps catching in other forms (the wp-honeypot "zero hits"
> assumption in session 22 was the same error, one level up).

---

## 3. What was actually captured

### 3.1 Two genuine shellcode emulation captures

```bash
sqlite3 /opt/dionaea/var/lib/dionaea/dionaea.sqlite "select * from emu_profiles;"
```

Profile 1 — a real, traced Windows API call sequence:

```json
[
  {"call": "_lcreat", "args": [".exe", "6"], "return": "4711"},
  {"call": "LoadLibraryA", "args": ["ws2_32.dll"], "return": "0x71a10000"},
  {"call": "socket", "args": ["2", "1", "6"], "return": "65"},
  {"call": "bind", "args": ["65", {"sin_port": "9988", "sin_addr": {"s_addr": "0.0.0.0"}}, "16"], "return": "0"},
  {"call": "listen", "args": ["65", "16"], "return": "0"}
]
```

Read in order: create an `.exe`, load Windows' networking library, open a raw
socket, bind it to `0.0.0.0:9988`, then `listen()`. This is the textbook shape of
**bind-shell backdoor shellcode** — the payload doesn't just crash the target
service, it tries to open a listening backdoor the attacker can connect back to
later on port 9988. Dionaea's `emu` module (the shellcode emulation engine — see
`modules=curl,python,emu` in `dionaea.cfg`) actually executed and traced this,
which is the single most valuable thing this honeypot can produce: real payload
*behavior*, not just "a connection happened." Profile 2 returned an empty call
list — most likely a truncated or partially-failed emulation from a different
connection, not a second full capture.

### 3.2 73 real payload-drop attempts

```bash
sqlite3 /opt/dionaea/var/lib/dionaea/dionaea.sqlite \
  "select download_md5_hash, count(*), group_concat(distinct download_url) from downloads group by download_md5_hash order by count(*) desc limit 20;"
```

A long tail: a handful of MD5s repeating 8–11 times (the same malware family
dropped by multiple bots or multiple times by one), many appearing once — the
normal shape of internet-wide opportunistic scanning rather than a single targeted
campaign.

`download_url` came back **empty for every single row.** Checked directly rather
than assumed to be a bug:

```bash
sqlite3 /opt/dionaea/var/lib/dionaea/dionaea.sqlite "select download_url, download_md5_hash from downloads limit 5;"
```

Confirmed genuinely always empty — not a bug. SMB-delivered files are **pushed**
directly into the fake share by the attacker; there is no URL to log because
nothing was ever fetched from anywhere, unlike an HTTP-delivered payload. The two
rows in the `offers` table (`smb://185.226.197.27/...`, `smb://152.32.206.181/...`)
are share-enumeration requests (`\srvsvc`), a different event type entirely — not
download sources.

### 3.3 2,492 credential-stuffing attempts

```bash
sqlite3 /opt/dionaea/var/lib/dionaea/dionaea.sqlite \
  "select login_username, login_password, count(*) from logins group by login_username, login_password order by count(*) desc limit 20;"
```

```text
root|          |555
admin|          |409
sa|             |251
sa|!QAZ2wsx     |30
sa|123456       |21
anonymous|anonymous@|11
```

Real column names (`login_username`, `login_password`) confirmed via `.schema
logins` after an initial guess (`username`/`password`) failed — same "verify, don't
assume the schema" discipline as the `dionaea.sqlite`-vs-`logsql.sqlite` naming
mismatch from session 22.

`sa` dominating is expected — it's SQL Server's default admin account, and this box
has MSSQL (1433) exposed. Two fingerprint details worth keeping for future
adversary-pattern writeups:
- `!QAZ2wsx` — a common "keyboard-walk" password (traces the left side of a
  QWERTY keyboard top-to-bottom), a standard entry in credential-stuffing
  wordlists.
- `IEUser@` — the default username on Microsoft's free "Internet Explorer testing"
  VMs. Its presence is a specific tell that this particular wordlist was built
  against exposed RDP/Windows test-environment targets, not a generic top-1000
  password list.

### 3.4 1,165 real MySQL query attempts — and a schema limitation

```bash
sqlite3 /opt/dionaea/var/lib/dionaea/dionaea.sqlite \
  "select mysql_command_cmd, count(*) from mysql_commands group by mysql_command_cmd order by count(*) desc limit 10;"
```

```text
3  | 1165   <- COM_QUERY (real query attempts)
1  | 75     <- COM_QUIT
14 | 60     <- COM_PING
```

`mysql_command_cmd` stores the MySQL **wire-protocol opcode**, not literal SQL
text — confirmed via `.schema mysql_commands` (`mysql_command_cmd NUMBER NOT
NULL`). 1,165 genuine `COM_QUERY` calls is real attacker interaction well past
handshake-and-leave, but the actual query strings are not recoverable from this
table by design.

**Checked whether the query text existed anywhere else — specifically, whether it
had been in the 28 GB debug log before truncation:**

```bash
grep -i "mysql" /opt/dionaea/var/log/dionaea/dionaea.log | grep -i "query\|select\|union\|insert" | head -50
```

```text
[31082026 19:34:20] scapy .../packet.py:627-debug: ###[ MySQL Command QUERY sizeof(27) ]###
```

Even at full debug verbosity, Dionaea's `scapy`-based packet dissector only logs
the **packet size**, never the query content. No SQL injection payloads or literal
commands exist anywhere in this system, at any log level. This confirmed the log
held nothing recoverable for this specific question — safe to truncate with
certainty rather than a guess.

---

## 4. The fix — root cause, not just the symptom

### 4.1 Immediate: reclaim disk

```bash
truncate -s 0 /opt/dionaea/var/log/dionaea/dionaea.log
df -h /
```

```text
/dev/sda1        38G  7.2G   29G  21%
```

`truncate -s 0` rather than `rm` — the container holds an open file descriptor to
this path; truncating in place lets it keep writing from byte 0 with zero
disruption. Deleting the file would leave the open descriptor pointing at
unlinked, invisible data still consuming disk space until the container restarted
anyway.

### 4.2 Root cause: the config

```bash
find /opt/dionaea/etc -type f
grep -A 15 "^\[logging\]" /opt/dionaea/etc/dionaea/dionaea.cfg
```

```text
[logging]
default.filename=var/log/dionaea/dionaea.log
default.levels=all
default.domains=*
errors.filename=var/log/dionaea/dionaea-errors.log
errors.levels=warning,error
errors.domains=*
```

**`default.levels=all` is the entire root cause.** Every log level, unfiltered,
forever — the C-level connection ref/unref, byte-stream copy operations, and
thread-timing lines that made up nearly all of the 28.6 GB were `debug`-level
noise this setting explicitly includes.

```bash
sed -i 's/^default.levels=all/default.levels=info,warning,error,critical/' /opt/dionaea/etc/dionaea/dionaea.cfg
grep "default.levels" /opt/dionaea/etc/dionaea/dionaea.cfg
docker restart dionaea
docker ps
```

`docker ps` confirmed the container back `Up` cleanly with every port mapping
intact after the restart — config change required a restart to take effect, no
live-reload mechanism.

### 4.3 This is a known, long-standing upstream footgun — not a local misconfiguration

```text
GitHub Issue #109 — DinoTools/dionaea, filed 2017:
"dionaea.log gets a few gig of data in under 24hrs."
```

`default.levels=all` has been the shipped default since at least 2017 and remains
unchanged in the current `dinotools/dionaea` Docker image used in session 22 — a
known issue, never fixed upstream, still present in a maintained image nine years
later. Official Dionaea documentation is explicit on this exact point:

> "This log is meant to be used for debugging and to track errors. It is **not
> recommended** to analyse this file to track attacks."

That confirms, directly from upstream, that using `dionaea.sqlite` as the primary
data source (established in session 22, reinforced by every real finding in this
session) was always the correct approach — the raw log was never meant to be the
analysis surface, even before it became a disk-filling liability.

> **LESSON — a defect discovered locally is often a known, unfixed upstream issue.**
> Same pattern as the two genuine Cowrie defects found in session 20 (`mkdir` dedupe,
> decimal-vs-octal permissions) — checking whether a problem is "just mine" or
> "everyone's" takes one search and changes whether the fix belongs only in this
> lab's config or is worth reporting upstream.

### 4.4 Recurrence prevention — a second, independent safety net

Lowering the log level slows the growth rate but does not eliminate it — a
bind-mounted file has no self-rotation regardless of verbosity, and Dionaea's
connection volume trends upward over time (41,369 connections in the first eight
days alone). Added host-level `logrotate` as a second, independent control:

```bash
cat > /etc/logrotate.d/dionaea << 'EOF'
/opt/dionaea/var/log/dionaea/*.log {
    daily
    rotate 7
    compress
    missingok
    notifempty
    copytruncate
}
EOF

logrotate -d /etc/logrotate.d/dionaea
```

- `copytruncate` — standard `logrotate` renames the file and signals the owning
  process to reopen a new one. Dionaea's container has no such reload hook.
  `copytruncate` copies existing content out, then truncates the original in
  place — safe against a process that just holds an open file descriptor and
  never expects the file to move, same mechanism as the manual `truncate -s 0`
  used in §4.1.
- `logrotate -d` (dry-run/debug mode) confirmed the config parses correctly and
  matches both log files without actually rotating anything — verified the config
  is live and will fire correctly on its own daily cron schedule, since "no
  rotation needed yet" in the dry-run output meant *not due*, not *broken*.

Two independent controls now bound this permanently: the config fix reduces
volume at the source; logrotate caps whatever still accumulates regardless of
level, the same way it would for a different log growing for a completely
different reason later.

---

## 5. Verification

```bash
docker ps
```

```text
358195b81065   dinotools/dionaea   Up Less than a second   [all 15 ports intact]
```

```bash
df -h /
# 38G   7.2G   29G   21%
```

All four systemd services and the Dionaea container confirmed live, disk usage
back to a healthy baseline, both fixes (config + logrotate) confirmed in place and
functioning.

---

## 6. Files changed

| File | Change |
|---|---|
| `/opt/dionaea/var/log/dionaea/dionaea.log` | Truncated to 0 bytes (data reviewed and confirmed non-recoverable/non-essential first — see §3.4) |
| `/opt/dionaea/etc/dionaea/dionaea.cfg` | `default.levels=all` → `default.levels=info,warning,error,critical` |
| `/etc/logrotate.d/dionaea` | **NEW** — daily rotation, 7-day retention, `copytruncate` mode |

---

## 7. Lessons

- **Read the whole traceback.** The exception type and message are always the
  last line. A paste that stops mid-stack is missing the one line that says what
  actually broke.
- **`df -h /` is a five-second check that should run early in any "something's
  wrong" investigation** — this incident's real cause was visible in the very
  first diagnostic command run, buried under a Cowrie stack trace that pointed
  somewhere else entirely.
- **Drill down with `du --max-depth` one level at a time rather than guessing
  the culprit.** The "obviously risky" candidate (`bistreams`, unbounded by
  design) was not the actual cause — an ordinary misconfigured log was.
- **Check structured data before destroying raw data**, even under time pressure.
  Eight days of real captures — two shellcode profiles, 73 downloads, 2,492
  credential attempts — would have been invisible if the log had been truncated
  on sight without checking `dionaea.sqlite` first.
- **One good night is not eight good days.** Session 22's "recon-only" finding
  was accurate for its one-night sample and silently stopped being representative
  well before this session — the same class of stale-assumption error as
  wp-honeypot's "zero hits" claim, one layer removed.
- **A local defect is often a known upstream one.** A single search confirmed
  this exact failure mode was reported against Dionaea in 2017 and never fixed in
  the default config — worth checking before assuming a problem is unique to this
  lab.
- **Official docs said not to use this log for attack analysis — and were right,
  for a reason beyond the obvious one.** The raw log doesn't just risk filling a
  disk; it structurally cannot answer some questions (MySQL query text) that the
  structured database also can't answer, because Dionaea never captures that
  detail at any verbosity. Confirmed empirically in §3.4, not just taken on faith
  from the documentation.
- **Fix the cause and add an independent backstop.** The config change reduces
  volume; logrotate bounds whatever remains regardless of cause. Neither alone
  is sufficient on its own for a component logging unattended for days at a time.

---

## 8. Open items

- `emu_profiles` id=2 (empty call list) — worth a closer look at what connection
  produced it and why the emulation trace came back empty; likely a truncated
  capture, not confirmed.
- The 73 downloaded-hash list has not been checked against VirusTotal —
  `virustotalscans` table is present but at 0 rows (feature never configured/run).
  A handful of the most-repeated hashes (`ae12bb54...`, `996c2b2c...`,
  `0ab2aeda...`) would be the highest-value first checks.
- Same `default.levels=all` question is worth checking against every other
  Dionaea-adjacent config on this box, if any exist, for the same footgun.
- Mailoney's real log file path is still unlocated (carried over from session
  22, unrelated to this incident).
