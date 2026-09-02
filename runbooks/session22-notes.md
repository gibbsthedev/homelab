# SESSION 22 — Dionaea Deployment & Traffic Analysis

**Date:** 2026-08-31
**Box:** Hetzner Cloud VPS `ubuntu-4gb-hel1-2` (62.238.47.215)
**Continuity:** Follows `SESSION-20-cowrie-shell-realism-source-mods-2026-08-30.md`.
Session 21 (`SESSION-21-honeypot-threat-analysis.md`) covers a separate threat-analysis
thread — this session picks up where session 20's "Next session" section left off:
attack-surface expansion.

**Scope:** Dionaea deployed as the first attack-surface addition beyond Cowrie/mailoney/
wp-honeypot. High-interaction Cowrie backend remains deferred (see session 20, §0) —
not attempted this session.

---

## 0. Decision: Dionaea via Docker, not from source

The scoped plan was to build Dionaea the same way everything else in this lab has been
built — from source, understanding every dependency. That plan hit a dead end.

```bash
# research finding, not a command run on the box
```

Official Dionaea documentation states plainly: **avoid Ubuntu 20.04 or higher.** The
reason is `libemu`, Dionaea's shellcode-emulation dependency, which was dropped from
Debian/Ubuntu's package repositories and has not been repackaged since. Every
from-source install path (the `cmake` build, the `honeynet/nightly` PPA) either fails
outright on 24.04 or requires manually compiling a years-stale `libemu` fork first.

This is a real dependency-hell case, not a solvable "read the error and fix it"
situation — the upstream ecosystem piece is genuinely gone. Building it by hand would
mean fighting infrastructure decay unrelated to honeypot skill-building.

**Decision:** use the official `dinotools/dionaea` Docker image for this one component.
This is a deliberate, scoped exception to the "build it yourself" approach that governed
every other change this session and last — the learning value here is in the captured
*data* Dionaea produces, not in compiling a decade-old C dependency tree. Cowrie's own
shell internals (session 20) remain hand-built; only this one component uses a
maintained upstream container.

```bash
docker --version
# Docker version 29.1.3, build 29.1.3-0ubuntu4.1
```

Installed via Ubuntu's packaged `docker.io` — sufficient for running one container,
avoids the overhead of adding Docker's own APT repository for a single-purpose install.

---

## 1. Port 80 — why Dionaea does not get it

Dionaea's documented default port list includes `80`. Port 80 on this box is already
owned by nginx, reverse-proxying to the hand-written wp-honeypot.

**Decision: nginx/wp-honeypot keeps port 80. Dionaea's port list drops it.**

Two reasons, not one:

1. **Realism loss.** Dionaea's HTTP module is a generic responder — it has no idea it
   is supposed to look like a WordPress login page. Replacing the purpose-built
   `wp-login.php` decoy (session-19-era work, still capturing traffic — see §4) with
   Dionaea's generic HTTP handler would be a downgrade in realism for the specific
   attacker population (credential-stuffing bots) that decoy targets.
2. **Diversity, not duplication.** Dionaea's actual value is protocol coverage
   *outside* what's already captured: SMB, RDP-adjacent, MSSQL, FTP, TFTP, MQTT, SIP,
   memcached. An SMB exploit or an FTP-delivered payload is a structurally different
   capture than "attacker types `wget` in an SSH shell" (Cowrie) or "attacker POSTs
   fake WordPress credentials" (wp-honeypot). Handing Dionaea port 80 would trade a
   distinct new capability for a worse version of something that already works.

---

## 2. Install

```bash
ssh root@62.238.47.215 -p 2200

apt-get update
apt-get install -y docker.io
systemctl enable --now docker

mkdir -p /opt/dionaea/etc /opt/dionaea/var/lib /opt/dionaea/var/log

docker run -d \
  --name dionaea \
  --restart unless-stopped \
  -v /opt/dionaea/etc:/opt/dionaea/etc \
  -v /opt/dionaea/var/lib:/opt/dionaea/var/lib \
  -v /opt/dionaea/var/log:/opt/dionaea/var/log \
  -p 21:21 -p 42:42 -p 69:69/udp -p 135:135 -p 443:443 -p 445:445 \
  -p 1433:1433 -p 1723:1723 -p 1883:1883 -p 1900:1900/udp \
  -p 3306:3306 -p 5060:5060 -p 5060:5060/udp -p 5061:5061 -p 11211:11211 \
  dinotools/dionaea
```

- `-v /opt/dionaea/{etc,var/lib,var/log}:...` — bind mounts, not Docker named volumes.
  Dionaea's own documentation specifically calls out these three paths as requiring
  persistence. Without them, `docker restart` (or a crash) silently discards every
  captured binary and every log line, same failure class as an unmounted Cowrie pickle
  losing edits on restart.
- `--restart unless-stopped` — survives host reboot and container crash, matching the
  reliability standard already set for `cowrie`/`mailoney`/`nginx`/`wp-honeypot` via
  `systemctl enable`.
- `-p 80` deliberately **absent** — see §1.
- Every other port from Dionaea's documented default list is included as-is: `21`
  (FTP), `42` (WINS), `69/udp` (TFTP), `135` (MSRPC), `443` (HTTPS), `445` (SMB),
  `1433` (MSSQL), `1723` (PPTP), `1883` (MQTT), `1900/udp` (UPnP/SSDP), `3306`
  (MySQL), `5060`/`5060 udp`/`5061` (SIP), `11211` (memcached).

### First-boot behavior

```text
'template/etc/dionaea/services-enabled/smb.yaml' -> './etc/dionaea/services-enabled/smb.yaml'
'template/lib/dionaea/binaries' -> 'var/lib/dionaea/binaries'
...
Starting dionaea ...
```

This block of `template/... -> ...` lines is the container populating the empty bind
mounts with default config and directory structure on its first run. Expected
first-boot scaffolding, not an error — a second `docker run`/restart against the same
mounts will not repeat it.

---

## 3. Verification

```bash
docker ps
docker logs dionaea --tail 30
ss -tlnp | grep -E ':(21|42|135|443|445|1433|3306)\b'
free -h
```

**Result:**

```text
CONTAINER ID   IMAGE               STATUS          PORTS
358195b81065   dinotools/dionaea   Up 16 seconds   0.0.0.0:21->21/tcp, [::]:21->21/tcp,
                                                    0.0.0.0:42->42/tcp, 0.0.0.0:135->135/tcp,
                                                    0.0.0.0:443->443/tcp, 0.0.0.0:445->445/tcp,
                                                    0.0.0.0:1433->1433/tcp, 0.0.0.0:1723->1723/tcp,
                                                    0.0.0.0:69->69/udp, 0.0.0.0:1883->1883/tcp,
                                                    0.0.0.0:3306->3306/tcp, 0.0.0.0:1900->1900/udp,
                                                    0.0.0.0:5060-5061->5060-5061/tcp,
                                                    0.0.0.0:11211->11211/tcp, 0.0.0.0:5060->5060/udp
```

```text
LISTEN  0.0.0.0:3306   docker-proxy
LISTEN  0.0.0.0:1433   docker-proxy
LISTEN  0.0.0.0:21     docker-proxy
LISTEN  0.0.0.0:42     docker-proxy
LISTEN  0.0.0.0:135    docker-proxy
LISTEN  0.0.0.0:445    docker-proxy
LISTEN  0.0.0.0:443    docker-proxy
(+ IPv6 equivalents)
```

- `docker ps` confirms the process is up **and** the port mappings match what was
  requested — a container can report "Up" while a port mapping silently failed to
  bind, so cross-checking the `PORTS` column against the intended list matters.
- `ss -tlnp` is the independent proof: it queries the kernel's actual socket table, not
  Docker's self-report. `docker-proxy` as the owning process is expected — Docker
  routes host-port traffic into the container's network namespace through its own
  userspace proxy, so the real Dionaea process itself never appears directly in `ss`
  output on the host.

```text
               total        used        free      shared  buff/cache   available
Mem:           3.7Gi       916Mi       1.5Gi       4.9Mi       1.6Gi       2.8Gi
Swap:          1.0Gi          0B       1.0Gi
```

- **2.8 GB available** with Dionaea running, up from the pre-Dionaea 3.0 GB baseline
  taken at the start of this session (§ below) — Docker daemon plus one lightweight
  container cost roughly 200 MB, well inside the headroom the earlier capacity check
  (session-20-era) projected. Swap untouched.

---

## 4. Reading Dionaea's captured data

Three separate places, by data type. **`sqlite3` is not installed on the host by
default** — `apt install -y sqlite3` first.

```bash
# Human-readable event log — DEBUG verbosity by default, very noisy.
# Filter to the lines that actually matter:
grep "accepted connection\|attackid" /opt/dionaea/var/log/dionaea/dionaea.log

# Confirm nothing is crashing (should stay near-empty)
wc -l /opt/dionaea/var/log/dionaea/dionaea-errors.log

# Anything actually dropped as a payload lands here
ls -la /opt/dionaea/var/lib/dionaea/binaries/

# Structured connection/session data
apt install -y sqlite3
sqlite3 /opt/dionaea/var/lib/dionaea/dionaea.sqlite ".tables"
sqlite3 /opt/dionaea/var/lib/dionaea/dionaea.sqlite \
  "select * from connections order by connection_timestamp desc limit 20;"
```

- `dionaea.log` / `dionaea-errors.log` — confirmed real filenames (found via `ls` on
  the mounted `var/log/dionaea/` directory). The main log runs at debug verbosity by
  default — every C-level connection ref/unref and stream-processing step is logged —
  so `grep` for the `info`-level lines that matter rather than raw `tail -f`.
- **`logsql.sqlite` is a dead end — do not use it.** The container touches an empty
  `logsql.sqlite` placeholder at startup that is never actually written to.
  Confirmed via `docker exec dionaea ls -la /opt/dionaea/var/lib/dionaea/*.sqlite*`:
  the real, actively-growing database is **`dionaea.sqlite`**, despite the ihandler
  being named `logsqlhandler` in the startup log (`Starting
  <dionaea.logsql.logsqlhandler object...>`). The handler's class name and its actual
  output filename do not match in this image version — not documented anywhere
  obvious, only found by checking file sizes from inside the running container.
- `var/lib/dionaea/binaries/` — the actual payoff of running Dionaea: any malware an
  attacker attempts to drop via SMB/FTP/TFTP/etc. lands here as a real file, ready for
  the same `file`/`xxd`/`strings` triage already used for Cowrie's `downloads/`
  directory. **Do not execute anything found here** — same standing rule as Cowrie
  downloads (session-19-era KB: a pot becomes a bot the moment a captured binary is
  run).
- `dionaea.sqlite` schema (confirmed via `.tables`): `connections`, `dcerpcbinds`,
  `dcerpcrequests`, `dcerpcserviceops`, `dcerpcservices`, `downloads`, `emu_profiles`,
  `emu_services`, `logins`, `mqtt_*`, `mssql_*`, `mysql_*`, `offers`, `p0fs`,
  `resolves`, `sip_*`, `virustotals`, `virustotalscans`. Far broader than Cowrie's
  single-stream JSON — one table per protocol/module, matching the protocol diversity
  described in §7.

### First real capture — same night as deployment

```text
32|accept|tcp|smbd|1788184672.45853|32||172.17.0.2|445|35.205.99.86||44028
31|accept|tcp|smbd|1788184667.91833|31||172.17.0.2|445|35.205.99.86||44024
30|accept|tcp|smbd|1788184667.82974|30||172.17.0.2|445|35.205.99.86||44014
...
23|accept|tcp|smbd|1788184636.03452|23||172.17.0.2|445|35.205.99.86||64600
```

**32 connections from a single IP (`35.205.99.86`) to SMB (port 445), spaced roughly
4–9 seconds apart over several minutes** — persistent automated scanning, not a
one-off probe, arriving within the first hour of Dionaea being live. Confirms the
core thesis from §1: this is genuinely new attack-surface data, a source that neither
Cowrie nor wp-honeypot could have captured.

```bash
sqlite3 /opt/dionaea/var/lib/dionaea/dionaea.sqlite \
  "select * from dcerpcrequests order by 1 desc limit 10;"
sqlite3 /opt/dionaea/var/lib/dionaea/dionaea.sqlite "select * from downloads;"
```

Both returned **zero rows** — clean empty result, not an error. The `35.205.99.86`
traffic never progressed past connect/disconnect: no DCERPC exploit call was ever
made, and nothing was offered as a download. Read as **recon-only** — almost
certainly a port/service scanner (Shodan/Censys-style discovery, or a worm's initial
probing phase) confirming 445 is open and reachable, not a live exploitation attempt.
Same category as mailoney's recon-only SMTP behavior already documented in the KB: a
real, useful negative result, not a dead end. Whether this IP (or others) escalates
to an actual exploit attempt on a later connection is the thing to watch for going
forward — the `connections` table is the place to check for repeat visitors.

---

## 5. wp-honeypot traffic review — a live finding, not silence

Going in, the assumption was "zero hits on the wp-honeypot." That assumption was
wrong — the log was never actually reviewed until this session.

```bash
tail -20 /var/log/nginx/honeypot-wp.access.log
tail -20 /var/log/wp-honeypot/http.jsonl
```

### Finding: Vite dev-server scanning, not WordPress credential attempts

```text
67.213.121.153 - - "GET /src/index.css HTTP/1.1" 404
67.213.121.153 - - "GET /@hmr HTTP/1.1" 404
67.213.121.153 - - "GET /vite/hmr HTTP/1.1" 404
67.213.121.153 - - "GET /vite/client HTTP/1.1" 404
67.213.121.153 - - "GET /@vite/client HTTP/1.1" 404
67.213.121.153 - - "GET /bundledDevClient.mjs HTTP/1.1" 404
67.213.121.153 - - "GET /__hmrClient HTTP/1.1" 404
```

Nineteen requests from one IP inside a one-second window, every path a **Vite
dev-server hot-module-reload endpoint** (`/@hmr`, `/vite/client`, `/__hmrClient`, and
variants), not a single WordPress path among them (no `/wp-login.php`, no
`/wp-admin`, no `/xmlrpc.php`).

**What this means:** a scanner population is actively probing this IP for exposed Vite
dev servers left running in production — a real, current vulnerability class distinct
from WordPress credential stuffing. Misconfigured Vite dev servers can leak source
files, environment variables, or in some configurations allow arbitrary file reads over
the dev WebSocket. This box was never built to emulate that stack, so every request
correctly 404'd — but the *attempt* itself is the useful data point: a second, distinct
scanner population is hitting this IP that the current decoys were never built to
attract in the first place.

### Fingerprint detail worth keeping

```text
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 15) ... Safari/605.1.15
User-Agent: Mozilla/5.0 (Knoppix; Linux i686) ... Chrome/131.0.0.0 Safari/537.36
User-Agent: Mozilla/5.0 (Macintosh, Intel Mac OS X 10_15_7) ... Safari/605.1.15
```

Every request in the burst carries a **different, randomized User-Agent** despite
coming from the same IP within the same second, against identical paths. That is
automated fingerprint rotation, not nineteen different humans with nineteen different
browsers — a tell worth citing in future adversary-taxonomy writeups the same way the
IoT botnet handshake sequence was in session 20.

### The two legitimate entries in the log

```text
62.238.47.215 - - "HEAD / HTTP/1.1" 200 0 "curl/8.18.0"
62.238.47.215 - - "HEAD /wp-login.php HTTP/1.1" 200 0 "curl/8.18.0"
```

These are the session's own `curl -I` verification checks (§ "on wp-honeypot/mail
silence"), sourced from the box's own IP with the `curl` user-agent. Confirmed
correctly excluded from adversary analysis — same standing practice as excluding
`104.241.55.33` (documented in the KB) — not a false "hit."

**Corrected conclusion:** wp-honeypot is not silent. Zero WordPress-specific credential
attempts so far, but real, live, distinct scanner traffic exists and is logging
correctly end-to-end (nginx → Python backend → `http.jsonl`).

---

## 6. Mailoney review

```bash
python3 -c "import sqlite3;c=sqlite3.connect('/home/mailoney/mailoney/mailoney.db');
print(c.execute('select count(*) from smtp_sessions').fetchone())"
```

```text
(456,)
```

**456 recorded SMTP sessions.** Mailoney has been capturing real traffic historically —
the earlier "no mail" impression was based on an instantaneous `ss -tlnp | grep :25`
check (nothing connected at that exact moment) rather than the actual session database.

```bash
ss -tlnp | grep :25
```

returning nothing means only that no connection was open in that instant — not that
port 25 is unreachable or that mailoney is broken. The 456-row count is the real proof
of function, same "verify by function, not by status output" standard applied
throughout session 20.

*Open item, not resolved this session:* `tail` against `/home/mailoney/mailoney/logs/*.log`
returned nothing — the glob likely does not match mailoney's actual log filename.
Confirming the correct path is a fast follow, not urgent given the sqlite count already
proves function.

---

## 7. Attack surface, before and after this session

| Component | Protocol(s) | Interaction style | Primary capture |
|---|---|---|---|
| Cowrie | SSH/Telnet (22/23) | Emulated shell | Commands, session behavior, `wget`/`curl`-delivered ELF binaries |
| mailoney | SMTP-over-SSH | Banner-only recon | Connection metadata (456 sessions to date) |
| wp-honeypot + nginx | HTTP (80) | Static decoy + credential capture | WordPress-targeted credential stuffing, generic web scanner recon (incl. Vite probing, this session) |
| **Dionaea** *(new)* | FTP, SMB, MSRPC, MSSQL, TFTP, MQTT, SIP, memcached, UPnP | Low-interaction protocol emulation | Malware binaries dropped via non-SSH, non-HTTP delivery paths |

Four distinct interaction models now running concurrently on one 4 GB / 2 vCPU box, at
2.8 GB RAM headroom remaining.

---

## 8. Open items

- ~~Confirm exact log filenames~~ — **resolved this session**, see §4. Real database
  is `dionaea.sqlite`, not `logsql.sqlite`.
- ~~Check `dcerpcrequests`/`downloads` for the `35.205.99.86` scanner~~ — **resolved
  this session**: zero rows in both, traffic was recon-only. See §4.
- Locate mailoney's actual log file path (the `logs/*.log` glob missed it).
- `tee`-created files still cannot be read back via `cat` (session 20, §15) — unrelated
  to this session, still open.
- High-interaction Cowrie backend remains deferred — gated on VPS upsize decision
  (CX33 unavailable in hel1; CPX32 is $41.99/mo, deferred as of session 20).
- No Dionaea traffic observed yet this session — container is new; nothing to analyze
  until real scans/connections accumulate.

---

## 9. Lessons

- **A documented upstream incompatibility beats a guessed build.** The Dionaea/`libemu`
  dead end was confirmed via current documentation before spending budget on a doomed
  from-source attempt — the "avoid Ubuntu 20.04+" line in official docs was the
  deciding fact, not an assumption.
- **A scoped exception to a principle is still a decision, and worth writing down as
  one.** Using Docker for Dionaea specifically, while everything else in this lab stays
  hand-built, is deliberate and bounded — not a drift back toward "just
  docker-compose everything."
- **"No hits" is a claim that needs the log, not the absence of a memory of checking
  it.** The wp-honeypot was assumed silent until this session actually read
  `honeypot-wp.access.log` — it was catching a real, previously-unnoticed scanner
  population the whole time.
- **A point-in-time socket check is not the same as a historical function check.**
  `ss -tlnp | grep :25` showing nothing proved only that nothing was connected in that
  instant; the sqlite session count was the real evidence mailoney works.
