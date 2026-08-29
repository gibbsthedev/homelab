# Session 19 — The OOM Problem Came Back (and What Finally Fixed It)

Date: 2026-08-28 into 2026-08-29

The `MemoryMax=1.5G` fix from session18 held for two days, then Cowrie started
getting killed again — this time by a completely different attacker cluster,
and at a level that finally justified adding real swap. Also covers the
honeypot's first real human web traffic and a few attacker profiles that
hadn't been written up yet.

---

## PART 1 — "THE BOX WAS NOT DOWN. COWRIE WAS." (again)

Investigating an apparent outage, the same discipline from session18 applied
immediately: check host health before assuming the honeypot itself failed.

```
hostname; uptime; who -b
dmesg -T | grep -Ei 'oom|killed process' | tail -30
systemctl is-active cowrie nginx wp-honeypot mailoney ssh
```
Result: the **host** was fine — 2 days 19 hours of clean uptime, disk at 10%,
plenty of free RAM, and nginx/mailoney/ssh all staying active the whole time.
**Only Cowrie was cycling.** Same shape of problem as session18, confirming
the pattern: a memory-cgroup kill looks like "the server is down" from
outside while the host itself never wavers.

**This time the kill line read differently:**
```
oom_memcg=/system.slice/cowrie.service
Killed process (twistd) anon-rss: ~1569 MB
```
Cowrie was dying at ~1.57GB — well *above* the 512MB from session18's original
incident, and even above the 1.5GB cap that fix had raised it to. **The
earlier fix wasn't wrong, it just wasn't the ceiling.** A `MemoryMax` override
drop-in genuinely does take precedence over the unit file's own value — worth
confirming directly rather than assuming, since the unit file on disk still
literally said `512M`:
```
systemctl show cowrie.service -p MemoryCurrent,MemoryMax,MemoryHigh
```
confirmed the live values (1.2GB high, 1.5GB max) came from the override, not
the base unit — drop-ins win, and it's worth checking `show` rather than
trusting what the unit file appears to say.

**Also surfaced:** `TimeoutStopUSec=1min 30s` explained why `systemctl stop`
sometimes looked hung — Twisted was ignoring SIGTERM for the full 90 seconds
before systemd escalated to SIGKILL. Not a new bug, just a slow, expected
shutdown path that looked alarming without knowing the timeout existed.

## PART 2 — ROOT CAUSE: A DIFFERENT MECHANISM THAN LAST TIME

Session18's leak came from a single cluster running lightweight one-command
sessions. This time, correlating event counts against source IPs pointed at a
**different, heavier pattern from a new cluster** (`103.134.26.228`–`.231`) —
thousands of connect/login/close events in a tight window, almost no actual
commands or downloads.

**This mattered for ruling things out, not just ruling one thing in:**
checking file-download and direct-tcp counts confirmed those were NOT the
leak driver (77 downloads, 358 relay attempts — real, but nowhere near enough
volume to explain a gigabyte-plus of growth). **The actual mechanism was
session churn** — Cowrie holds transport, TTY, and key-exchange objects per
session, and thousands of rapid connect-then-close cycles compounded with
incomplete garbage collection walked memory straight up to the cap.

**Lesson, sharpened from session18:** a memory leak doesn't have one universal
cause just because it happened before under the same service. Re-diagnose
each recurrence against the actual traffic at the time rather than assuming
the previous root cause repeated.

## PART 3 — SWAP, ACTUALLY APPLIED THIS TIME

Session18 considered 1GB of swap and decided it wasn't needed yet. This time
it was:
```
fallocate -l 1G /swapfile
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile
echo '/swapfile none swap sw 0 0' >> /etc/fstab
```
**Important distinction documented here:** swap does NOT raise or bypass
`MemoryMax` on Cowrie's cgroup — the cap still kills Cowrie before it can
lean on swap heavily. What swap actually protects is the *host* — sshd,
nginx, and everything else — if the box overall ever gets tight. Two
different safety nets for two different failure modes, not a substitute for
each other.

**Deliberately left alone:** the cap stayed at 1.5GB rather than being
lowered back to 512MB. A nightly scheduled Cowrie restart (as a cheap
insurance policy against slow leaks) and a shorter `TimeoutStopSec` were
both named as available future hardening, not applied this session.

## PART 4 — THE WORDPRESS HONEYPOT'S FIRST REAL HUMANS

The HTTP honeypot had been live but quiet. This chapter brought its first
identifiably human traffic, plus one notable cross-honeypot correlation:

- **The SSH user `poop`** (previously seen doing SFTP-only exploration) showed
  up again on the *web* honeypot from a different IP, trying `poop/admin`,
  `poop/poop`, and `poop@<the VPS IP>` as login attempts — the same person,
  confirmed by username reuse across two completely different honeypot
  services. A useful reminder that attacker/curious-visitor identity can
  sometimes be tracked across services, not just within one log.
- **A combined Firefox + Nmap NSE session** from one IP — a human browsing,
  interleaved with Nmap's scripted `system.listMethods` / VMware SOAP `/sdk`
  probes — then posting a plain `hi`/`hi` login attempt. The same visitor also
  pulled a breadcrumb file (`notes.txt`) over SSH, meaning they were actively
  working both honeypot surfaces in the same visit.
- Otherwise mostly reconnaissance-shaped traffic: crawler bots, generic HTTP
  fingerprinting tools, and WordPress-path probing — no brute-force flood yet,
  consistent with the earlier finding that mass `wp-login.php` attacks tend to
  lag a few days behind a port becoming visible.

## PART 5 — MAILONEY'S ONLY REAL INBOUND CONTENT SO FAR

Out of 273 SMTP sessions captured, still zero credentials and an empty
`captured_mail/` — but exactly one session broke the banner-only pattern:
```
out  220 app-prod-01 ESMTP Service Ready
in   shell     → 502 command not recognized
in   sh        → 502
in   exit      → 502
```
Someone tried to treat the SMTP channel as an interactive shell rather than
speaking SMTP to it — almost certainly a human or a generic-tunnel tool
probing what was on the other end of an SSH `direct-tcp` forward, not a real
mail-relay attempt. Interesting as a single data point, but not enough to
change the standing "reconnaissance-only" conclusion about this traffic
class.

## PART 6 — A NEW CLASSIFIER CLUSTER, AND WHY IT EXPLAINED THE CPU SPIKE

A related IP family (`92.118.39.49`/`.50`/`.71`) runs one large fingerprinting
script per login: PATH, `uname`, architecture, uptime, core count, CPU model,
GPU detection via `lspci`, then a shell-behavior self-test. **This is a
miner/compute classifier** — sizing a host's hardware (specifically checking
for a GPU) before deciding whether it's worth deploying anything to.

**Why this mattered for the earlier CPU-hours question:** Cowrie has to
emulate every piped command in that script, which produced a real, measurable
CPU spike (8.5% CPU observed within a minute of one login) — a good
explanation for a stretch of elevated CPU usage that had looked unexplained.
**Lesson: `ps` RSS is reported in kilobytes, and a value like `200000` is
~195MB, not 200GB — worth double-checking units before reacting to a number
that looks alarming at a glance.**

## PART 7 — A FEW ATTACKER PROFILES WORTH RECORDING (from the same window)

- **`188.252.185.200`** — Windows OpenSSH client, logged in as `poop`, used
  **SFTP only**, and only ever touched `/home/poop` (empty) — never reached
  `/root` or the planted decoys at all. A visitor who explored far less than
  most, worth noting as a contrast case.
- **`174.224.59.156`** — identified itself as `SSH-2.0-WebSSH_32.8` (a
  phone-based web SSH client) and never completed the connection — a key
  exchange mismatch Cowrie's Twisted version couldn't negotiate. Not an
  attack, just an incompatible client — recorded so the same "failed KEX"
  pattern isn't re-investigated as suspicious later.
- **HTTP recon crawlers** (a Nokia-attributed crawler, an internet-measurement
  crawler) hit the web honeypot in its first days doing generic path
  cataloging (`/admin`, Cisco VPN login paths, generic CGI paths) rather than
  anything WordPress-specific — confirms the earlier read that initial HTTP
  traffic is broad internet cartography before it narrows to WP-specific
  brute forcing.

## LESSONS

- **A fix that resolves one incident doesn't guarantee the next one shares
  the same root cause** — the second OOM wave came from an entirely different
  traffic pattern (session churn, not download volume) and needed its own
  diagnosis, not an assumption that "the same thing happened again."
- **Confirm which config layer is actually in effect** (`systemctl show`, not
  just reading the unit file) — a systemd override drop-in silently wins over
  what the base file says, and trusting the file alone would have given a
  wrong answer.
- **Swap and a memory cgroup cap protect different things.** Swap helps the
  host as a whole; a cgroup `MemoryMax` still governs one service's own
  ceiling regardless of how much swap exists.
- **Identity can cross honeypot services** — the same username or behavioral
  signature showing up on both the SSH and HTTP honeypots is real
  corroborating evidence, not a coincidence to ignore.
- **Always sanity-check units on a number that looks dramatic** — RSS in KB
  vs. MB vs. GB is an easy misread under pressure.

---
END. This closes out the honeypot's build-through-hardening arc as currently
documented. Next up: the AWS GuardDuty trial (session20).
