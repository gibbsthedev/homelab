# SESSION 43 — Final pull and lifetime close-out (2026-10-30)

**Host:** `ubuntu-4gb-hel1-2` (Hetzner, `62.238.47.215`)
**Thread:** honeypot. The previous honeypot doc is `SESSION-40-four-sensor-pull-and-corrections-2026-09-24.md`.
**Status:** ✅ **COMPLETE — honeypot decommissioned.** This was the last data pull; the VPS was deleted on 2026-10-30. The honeypot thread is closed.

Source: `hp-part-00…09` (a split `.tar.xz`, about 260 MB compressed). The archive was rebuilt and analyzed off-box. Nothing captured was run.

> **Conventions used in this doc**
> - **"Lifetime"** means first record → 2026-10-30 ~21:00 UTC.
> - **"Since S40"** means 2026-09-25 00:00 UTC onward.
> - Every aggregate excludes `104.241.55.33` (analyst) and `127.0.0.1` (mailoney tests).
> - All times are UTC.

---

## 0. Known gaps in the final data set

The VPS is gone, so these gaps are **permanent unless the missing files turn up on Mint**. A possible second copy on Mint is still to be checked. The VPS commands below are kept only as a record of what a complete final pull should have included.

| Missing / damaged | Why it matters |
|---|---|
| `/var/log/wp-honeypot/http.jsonl*` | Not in the archive. The only HTTP data here is nginx's 14-day rotation (Oct 16–30), so **lifetime HTTP totals can't be closed** from this pull. |
| `dionaea.sqlite` copied **while Dionaea was writing to it** | The copy ends early: index pages point past the end of the file (`invalid page number 348516`; the file has 348,497 pages). I recovered it with `.dump` → rebuild, and all 29 tables came back. Row counts below are **floors**, but the loss looks small because the newest connection row is 21:04, minutes before the pull. |
| Cowrie `downloads/`, Dionaea `binaries/` | Not in the archive. Only the hashes survive. The S40 open items about pan-chan and the ARM64 ELF need these files. |

**[VPS — as root, admin SSH on 2200] Stop Dionaea first so the database copy is clean, then archive what's missing:**
```bash
docker ps --format '{{.Names}}  {{.Image}}'          # find the dionaea container name
docker stop <dionaea-container-name>
python3 -c "import sqlite3;print(sqlite3.connect('file:/opt/dionaea/var/lib/dionaea/dionaea.sqlite?mode=ro',uri=True).execute('PRAGMA integrity_check').fetchone())"
tar -cJf /root/hp-final-2.tar.xz /var/log/wp-honeypot /home/cowrie/cowrie/var/lib/cowrie/downloads /opt/dionaea/var/lib/dionaea
ls -lh /root/hp-final-2.tar.xz
```
The integrity check should print `('ok',)`. The `downloads/` and `binaries/` folders are live malware samples. They're safe inside the tarball, but the standing rule still applies: **never extract them onto a machine where they could be executed.**

---

## 1. Lifetime closing numbers

### Cowrie (SSH 22 / Telnet 23) — 2026-08-23 → 2026-10-30

| Metric | Lifetime | Since S40 | Share since S40 |
|---|---:|---:|---:|
| Events | **1,305,741** | 797,865 | 61% |
| Sessions | **164,611** (SSH 100,525 · Telnet 64,086) | 102,182 | 62% |
| Unique source IPs | **10,380** | 5,021 (4,247 never seen before) | — |
| Login attempts | **136,544** (60,100 accepted · 76,444 rejected) | 93,245 | — |
| Unique credential pairs | **13,907** (3,909 users · 8,702 passwords) | — | — |
| Commands entered | **292,501** (9,071 unique) | 180,983 (6,587 new) | 62% |
| File downloads | **9,119** events | 6,897 | 76% |
| File uploads | **636** | 537 | 84% |
| Unique file hashes (down + up) | **655** | 509 new | 78% |
| URLs / filenames fetched | **340** | 252 new | — |
| Proxy (direct-tcpip) requests | **15,457** | 11,770 | 76% |
| TTY logs closed | **51,748** events (2,227 TTY files retained) | — | — |
| CVE-2026-24061 telnet `-f root` | **40 attempts · 2 successes** | 8 · 1 | — |
| `chpasswd` root-takeover attempts | **18** | 7 | — |
| Distinct SSH client fingerprints (HASSH) | **139** | — | — |

**Hourly coverage: 74.7%** (1,230 of 1,647 hours have at least one event). Most of the downtime falls in October; see §2.5.

Top SSH client strings over the full run: `SSH-2.0-Go` (70,037), `libssh_0.9.6` (9,718), `libssh2_1.8.1` (4,217), `PuTTY_Release_0.84` (1,361).

### Dionaea (multi-protocol) — 2026-08-31 → 2026-10-30

| Metric | Lifetime | Since S40 |
|---|---:|---:|
| Connection records | **3,438,404** (2,291,429 accepted) | 3,321,316 |
| Unique source IPs | **23,322** | 20,762 (19,292 new) |
| Login attempts (MSSQL / FTP / MySQL) | **2,026,722** | — |
| MSSQL post-login commands | 4,782 | — |
| SIP commands | **1,213,781** (REGISTER 503,978 · OPTIONS 362,434 · INVITE 341,911) | — |
| Downloads | **1,004** (362 unique MD5) | 763 (251 new MD5) |

Connections by service: MSSQL 2,038,537 · SIP 1,202,567 · SMB 126,513 · HTTPS 26,174 · PPTP 15,435 · FTP 7,854 · MySQL 7,284 · UPnP 6,156 · Memcache 2,673 · EPMAP 2,252 · MQTT 1,627.

### mailoney (SMTP 25) — DB records 2026-09-15 → 2026-10-30

| Metric | Lifetime | Since S40 |
|---|---:|---:|
| SMTP sessions | **3,957** | 2,296 |
| Unique source IPs | **714** | 581 (536 new) |
| AUTH credential submissions | **54** from 7 IPs | 48 |
| Captured messages (`.eml`) | **79** | 64 |

### wp-honeypot / nginx (HTTP 80) — **partial: 2026-10-16 → 2026-10-30 only**

| Metric | Oct 16–30 |
|---|---:|
| Requests | 23,648 (306 lines unparsed) |
| Unique IPs | 1,148 |
| Response codes | 404 = 18,800 · 200 = 3,853 · 400 = 471 |

Lifetime HTTP totals stay **open** until `/var/log/wp-honeypot/` is pulled (§0).

### All sensors combined

- **33,700 unique source IPs** across the four sensors (HTTP counted for Oct 16–30 only).
- About **4.8 million** recorded interactions (Cowrie events + Dionaea connections + SMTP sessions + HTTP requests).

---

## 2. What's new since SESSION-40

### 2.1 Cowrie traffic roughly quadrupled in the final week

| Window | Sessions per day |
|---|---|
| Sep 25 – Oct 3 (last stretch with full-day coverage) | ~2,100 |
| Oct 23 – Oct 30 | **~7,850** (peak: Oct 25, 10,667) |

```
Sessions/day (Cowrie)
Sep 25-Oct 3  ████                       ~2,100
Oct 11-14     ██████                     ~3,300
Oct 23-30     ████████████████           ~7,850
```

Drivers of the surge:

- **`109.160.32.0/24` — 106 IPs, 30,207 SSH sessions.** First seen 2026-09-06. In the final week many of these IPs logged **exactly 1,198 sessions each**. Identical per-host counts across a whole /24 mean one operator splitting a single job across hosts.
  - Behavior: a credential list that tries 25 of each pair; an empty-username login that Cowrie accepts (122); then `uname -s -v -n -r -m` (281).
  - This is recon, not payload delivery.
- **New high-volume single IPs:**
  - `188.166.65.22`: Telnet, first seen Oct 29, 15,936 events.
  - `164.92.115.22`: first seen Oct 28, 10,908 events.
  - `64.207.185.5`: first seen Sep 30, 30,907 events. It's the #1 IP since S40.
- **Mirai-family loaders dominate the commands** (window starting Oct 23):

  | Indicator | Count | What it is |
  |---|---:|---|
  | `sh .k` with a `/proc/*/exe` walker script | 2,430 | Competitor-killer: it enumerates processes so it can kill other bots |
  | `./.d; ./dvrHelper selfrep` | 1,059 | Mirai self-replication on DVR targets |
  | `/bin/busybox HISILICON` | 1,235 | Checks for a HiSilicon (camera/DVR) chipset |
  | `>/<dir>/.f && chmod 777 … && /<dir>/.f` across 10 directories | 449 each | Writable-directory sweep: the same idea as the `kla.sh` `echo WRITABLE` probe from S40, in a new form |

### 2.2 Telnet logins started being rejected on about Oct 11 — cause unconfirmed

- Before Oct 11, failed logins averaged **~30/day**. From Oct 11 on, they ran **thousands per day**; Oct 30 alone had 12,229.
- The clearest case is `root / xc3511`:

  | Period | Accepted | Rejected |
  |---|---:|---:|
  | Before Oct 11 | 1,228 | 0 |
  | After Oct 11 | 5 | 419 |

- A new Mirai variant also arrived using **NUL-terminated** credentials (`enable\0 / linuxshell\0`, `system\0 / shell\0`, `sh\0 / /bin/busybox UNSTABLE\0`). Cowrie rejects every one of them.
- The data can't explain the change. It lines up with the restarts during the Oct 4–11 outages (§2.5).
- **Check on the box before shutdown:** whether `etc/userdb.txt` or the Cowrie version changed around Oct 10–11.
- **Why it matters:** fewer accepted Telnet logins means fewer post-login commands captured per Telnet session in the last three weeks.

### 2.3 A second CVE-2026-24061 success, and the `94.154.43.0/24` actor moves on

- **`150.241.87.134` triggered `exploit_success` on 2026-10-01.** It's in the same /24 as the first success (`150.241.87.249`, Sep 21), and `.134` had already attempted on Sep 23. That's two successes from one /24, ten days apart.
- **`94.154.43.196`** made its last attempt on **2026-09-29**. That's 20 attempts in all, on its usual 1–3 day cadence.
- Five days later, **`94.154.43.138`** (same /24) appeared on Oct 4. It sent `echo ancient_telnet_ok` / `busybox` probes over raw `session.input` (289 events) and no CVE string.
  - **Possibly** the same operator switching technique. The evidence is only the shared /24 and the timing.
- There have been no CVE-2026-24061 attempts since Oct 1.

### 2.4 Proxy abuse grew; the SMTP relay through Cowrie mostly stopped

- **76% of all direct-tcpip proxy requests came after S40** (11,770 of 15,457).
- The proxied HTTP `Host:` headers are connectivity checks — google.com (5,148), yahoo.com (3,802), yandex.ru (2,080) — so the tunnels were being tested before resale.
- The top proxy users are all returning actors first seen Aug 29:
  - `185.246.128.133` (4,154)
  - `179.43.133.154` (3,793)
  - `179.43.139.58` (1,716)
- `91.92.242.247` is the new addition (first seen Sep 24, 1,357).
- **Port-25 pivots to Yandex** (`77.88.21.158:25`) fell from 682 before S40 to **26** after. That spam route went quiet.

### 2.5 Cowrie outages — and Dionaea kept running

| Outage | Length |
|---|---|
| Oct 4 21:00 → Oct 5 21:00 | 25 h |
| Oct 6 04:00 → Oct 7 13:00 | 34 h |
| Oct 8 03:00 → Oct 9 16:00 | 38 h |
| Oct 10 15:00 → Oct 11 08:00 | 18 h |
| **Oct 16 15:00 → Oct 21 01:00** | **107 h** |
| Oct 21 18:00 → Oct 23 00:00 | ~29 h (with one stray event Oct 22 05:06) |

- **Dionaea logged 3,500–5,600 accepts every day from Oct 16 to 21.** The host and network were up, so **the outages were Cowrie-only**.
- That fits the freeze pattern S38 flagged. S38's request to capture `gdb-bt.txt` on the next freeze was never done; it closes unresolved with the shutdown.

### 2.6 New droppers and payload infrastructure (Cowrie)

| Source | Seen from | Notes |
|---|---|---|
| `http://176.65.139.40/wget.sh` | **127 IPs** (143 events) | The most widely shared new dropper |
| `http://130.12.180.212/{x86_64,aarch64,mipsel,mips,armv5l}` | 3–4 IPs | A full multi-architecture set from one host |
| `http://176.65.139.196/bins/p{x86,mips,mpsl}` | 2 IPs | Same /24 as the `wget.sh` host above |
| `http://89.163.157.131:8080/persist.sh` | 1 IP (25 events) | The name says persistence — worth a look if the sample is recovered |
| `http://202.110.70.60:800/wget.sh`, `alya.*` (15 architectures) | 1 IP | "alya" botnet naming |
| `http://64.89.160.48/bins/parm7` | 1 IP | — |
| Upload `.16` (sha256 `a151d3f4…`) | 13 IPs: `92.118.39.0/24`, `195.178.110.227`, `193.32.162.84` | 115 events |

The three most frequent "new hashes" — `f41f11ca…` (90 IPs), `7c34c76b…` (107 IPs) and `4685a9b9…` (45 IPs) — are **redirect-built files** (`Saved redir contents`). Attackers typed them in with `echo … > file`; they weren't fetched from anywhere. Count them as dropper scripts, not downloads (the S40 gotcha).

### 2.7 Dionaea: one 24-hour MSSQL brute-force run makes up 57% of all its records

- **Who:** seven hosts in `62.210.0.0/16` and `195.154.0.0/16`. Both ranges are usually Scaleway / Online SAS; confirm with whois.
- **When:** all seven started between **2026-10-02 18:43 and 18:47** and stopped around **Oct 3 19:15**.
- **What:** **1,941,723 `sa` logins.**
- **How it was coordinated:** each host sent about 268k–288k passwords. Across all seven, 1,941,723 attempts used **1,940,313 distinct passwords (99.93% unique)**. One password list was split into seven slices, one per worker. It's the clearest case of coordinated brute-forcing in the whole project.
- Leave this burst out of trend lines, or every Dionaea chart becomes one spike.

**Other Dionaea findings since S40:**

- **MSSQL post-login tooling:** SweetPotato and `exegod` CLR assembly drops (206 each), `xp_cmdshell` re-enabling (~110). It's the same small set of IPs over weeks: `190.64.90.158` was still active on Oct 29.
- **MySQL:**
  - `SELECT COUNT(*) FROM vicidial_live_agents` from `45.86.202.0/24` and `91.217.249.0/24`. This looks for VICIdial, open-source call-center software.
  - `lib_mysqludf_sys.dll` UDF load attempts (6).
  - The **only non-SMB download** in the whole run: `http://156.225.17.60:122/cn.exe`, fetched by `192.140.178.164` on Oct 23.
- **SIP toll-fraud scanning** at scale. User agents: `Softphone/2.1.0`, `pplsip`, `VOIP`, `PBX`, `Avaya`.
- **MQTT:** wildcard subscriptions to `$SYS/#` (179) and `#` (151) — broker reconnaissance.
- **SMB downloads:** 251 new MD5s, mostly artifacts of Dionaea's own DoublePulsar emulation (S40 note). **Don't submit them as new variants.**

### 2.8 mailoney: the staged `235` works, and one bot used the decoy's hostname

**S40 open item answered:** yes, `160.250.128.49` received the staged `235` in session 2111 (Sep 23 20:45, credentials `scanner/scanner`), then **sent `QUIT` immediately**. It came back on Sep 27 over STARTTLS, re-checked `scan` and `scanner`, got `235` again, and still sent no `MAIL`.

That turns out to be one tool family, now seen four times:

| IP | Dates | Spacing | Passwords |
|---|---|---|---|
| `160.250.128.49` | Sep 23 | ~38 min | username = password |
| `196.251.121.220` | Sep 30, Oct 4 | 5–25 min | all `Passwd` |
| `196.251.121.113` | Oct 4 | 6 min | all `12345` |
| `157.66.48.107` / `.88` | Oct 9, 21, 23 | 20–38 min | `12345` or username = password; adds `postmaster` |

- They share a username list (`info`, `admin`, `test`, `user`, `sales`, `scan`, `scanner`), the `EHLO User` greeting, and **slow, evenly spaced attempts — low-and-slow, to stay under rate limits.**
- **None of them sent mail after `235`.** They validate logins and stop. The only later activity is `196.251.121.220` on Oct 7 under a different `EHLO`: `MAIL FROM:<test@test.com>` and nothing more.
- Inference: the bait worked — this server is probably on a list of SMTP logins someone believes are valid.

**New — a bot built a login from the server's own banner.** On Oct 24, `196.251.121.215` tried `admin@app-prod-01.local` with six passwords (`admin`, `passwd123`, `admin1234`, `admin123`, `admin12`, `admin666`) in a single second. `app-prod-01` is the decoy hostname in mailoney's `220` banner and the WordPress decoy pages. The bot **read the greeting and built the username from it**, instead of using a fixed list.

**Relay checkers keep coming back.** Recipient mailboxes are masked; the full addresses are in the `.eml` files.

| Subject | Senders | Since S40 |
|---|---|---:|
| `Valid SMTP 62.238.47.215` | `178.16.54.51`, `178.16.54.59`, `91.92.241.235`, `185.169.4.42` → gmail/hotmail/outlook drops | 4 |
| `Open Relay Test` (sometimes `- 62.238.47.215`) | `91.92.41.82` (also 6 subject-less on Oct 23) | 7 |
| `Direct SMTP Test - 62.238.47.215 (Local Domain)` | `45.148.10.5`, to the same yopmail address each time, on **Oct 4, 18, 27**. It sent `AUTH PLAIN admin/password` on the same days. | 3 |
| `… NEW SCRIPT [PLAIN] <id> <n>` | `158.94.210.8`, `178.16.54.144`, to the same recipient domain | 2 |

Outbound TCP/25 is still blocked at Hetzner, so no captured message ever left the box.

### 2.9 HTTP (Oct 16–30 window)

- **Most of the traffic hunts for secrets:**

  | Target | Hits |
  |---|---:|
  | `.env` variants | 3,728 |
  | `.git` | 1,103 |
  | `phpunit` | 854 |
  | `.aws/credentials` | 468 |
  | `wp-config` | 437 |
  | `wp-login` | 537 |
  | PHP-CGI argument injection (CVE-2024-4577 / CVE-2012-1823 forms) | 110 |

- **Tooling seen:**
  - `WebSiphon/next-0.1`: 1,610 requests, mostly `163.192.0.0/16`.
  - `AssetEye-KeyScan/1.0`: 437 requests, from `52.128.241.98`.
  - `libredtail-http`: 1,034 requests. RedTail is still active.
- **Fake Googlebot from a new source:** `79.124.58.82` sent 600 requests with a Googlebot user agent. It's not a Google address. S40's fake-Googlebot range was `94.154.46.243–.250`.
- **Busiest range:** `45.138.12.22/.24/.27`, 3,998 requests combined.

### 2.10 Cross-sensor overlap

| Seen on | IPs |
|---|---:|
| 2 sensors | 1,317 |
| 3 sensors | 236 |
| All 4 | 36 |

Because HTTP only covers 15 days here, **these numbers can't be compared with S40's 793 / 11**. As in S40, the recognizable all-four IPs are research scanners (Censys `167.94.146.x`, Shodan `71.6.x`, Shadowserver `65.49.1.x` / `64.62.156.x`). Breadth across sensors still doesn't mean a threat.

---

## 3. Status of S40's open items

| Item | Status |
|---|---|
| mailoney session 2111: did `160.250.128.49` get `235`, and did it send anything? | **Closed.** Yes, it got `235`; no, it never sent. See §2.8. |
| Disassemble the 348-byte ARM64 ELF | **Open.** It's in `/root/quarantine/` on the VPS — copy it off before deletion. |
| Confirm pan-chan for `7a9da7d1`, `c71b30e5`, `e23dbf6f`, `062ba629`, `47b268c2` | **Open.** Needs Cowrie `downloads/` (§0). |
| Raise `download_limit_size` to 64 MiB | **Moot** with the shutdown. |
| Match `23.135.92.87`'s TTY file to the decoy-string hits | **Open.** The TTY files are in this pull (2,227), so this can be done off-box. |
| Capture `gdb-bt.txt` on the next freeze | **Closed unresolved.** The freezes happened (§2.5) without a capture. |
| Manual VirusTotal lookups | Carry forward. The highest-value hashes: `176.65.139.40/wget.sh`, the `130.12.180.212` set, `persist.sh`, `cn.exe`. |

---

## 4. Shutdown — done (2026-10-30)

- [x] Final pull taken (this doc)
- [x] VPS deleted
- [ ] *Later, optional:* look on Mint for a copy of `/var/log/wp-honeypot/`, Cowrie `downloads/` or `/root/quarantine/`. If any turns up, add the HTTP lifetime totals and close the remaining §3 items as an addendum to this doc.
- [ ] *Later, optional:* in the Hetzner console, confirm no snapshots, volumes or primary IPs are left billing.
- The Telnet change in §2.2 **can no longer be confirmed** — it stays an observation with no confirmed cause.
- Update `PROJECT-KNOWLEDGE-BASE.md`: the honeypot is **decommissioned**, and this doc holds the final numbers.

---

## 5. Portfolio summary

> Ran a four-sensor honeypot (Cowrie SSH/Telnet, Dionaea, an SMTP honeypot, a WordPress decoy) on a public VPS for ~10 weeks: **~4.8M recorded interactions from 33,700+ unique IPs**, **655 unique malware/dropper hashes**, 2 CVE-2026-24061 exploitation events, and a coordinated MSSQL brute-force run: 1.94M attempts split across 7 hosts with 99.9% unique passwords. Found a slow SMTP credential-validation campaign across four IP ranges and a bot that built its login from the server's own banner. Patched Cowrie for shell realism, added staged SMTP AUTH responses, and documented the work as a numbered series of session write-ups with corrections tracked between them.

---

### Method notes (this session)

- The data arrived as a **split xz tarball** (`hp-part-00`–`09`). Join the parts in order: `cat part-00 … part-09 > hp.tar.xz`, then `xz -t` before extracting.
- **Recovering a live-copied SQLite file:** `.recover` needs `sqlite_dbpage`, which the stock 3.45 CLI lacks.
  1. Run `.dump`.
  2. Remove the `/****** CORRUPTION ERROR *******/` marker lines.
  3. Change the trailing `ROLLBACK; -- due to errors` to `COMMIT;`.
  4. Pipe the result into a fresh database.
- **Coverage first:** counting hours with any event found the October outages before they could be mistaken for quiet periods. Add it next to the `eventid` histogram as a first-pass check.
- **Check mailoney outcomes in `session_data` (the JSON transcript)**, not just the `credentials` table — the `235` → `QUIT` pattern only appears there.
