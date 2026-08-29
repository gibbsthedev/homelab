# Session 12 — Building the Honeypot: Hetzner VPS to First Captured Attacks

Date: 2026-08-23 (overnight session)

This is the start of the honeypot track — the first project built
specifically for the SOC-analyst portfolio rather than for infrastructure
cost or convenience. By the end of this session, a live Cowrie SSH honeypot
was catching real internet attacker traffic, before the setup was even fully
finished.

---

## PART 1 — THE VPS

Tried Oracle Cloud's free tier first; hit signup friction (card verification
failed, then a request-rate lockout — a known common pain point with Oracle,
not specific to this account) and pivoted to **Hetzner Cloud**, CX23 plan
(~$6/mo). Ubuntu 26 LTS, public IP `62.238.47.215`.

A few deliberate decisions here, worth explaining since they're not obvious:

- **A dedicated, passphrase-protected SSH key**, never reused from any other
  homelab key. This box is intentionally internet-exposed bait — if its key
  were ever compromised, it shouldn't be able to touch anything else.
- **Generic server naming**, not honeypot-flavored. The server's own name in
  the Hetzner console is visible only to the operator and to Hetzner, never
  to an attacker — an attacker only ever sees what Cowrie presents them,
  completely independent of the real console name. A honeypot-flavored name
  doesn't fool attackers at all; it just risks drawing unwanted scrutiny from
  Hetzner's own fraud-review systems on a brand-new account.
- **Real admin SSH moved off port 22, onto port 2200**, and *verified
  working* before Cowrie ever touched port 22 — same "test the new door
  before locking the old one" discipline used during the Hermes migration.
  Ubuntu uses systemd socket activation for sshd, so a plain
  `systemctl restart sshd` does **not** move the listening port — it takes
  `systemctl daemon-reload` followed by `systemctl restart ssh.socket`. This
  cost real time to catch.

## PART 2 — COWRIE, AND THE VERSION-DRIFT TRAP

Cowrie 3.0 restructured its install process significantly, and most guides
findable by search still describe the old process. Documenting the current
one here so it doesn't cost time again:

| Old guides say | Current (3.0.x) reality |
|---|---|
| `bin/cowrie start/stop/status` | `bin/` is gone — install with `pip install -e .` inside a venv, which registers a `cowrie` command directly |
| `cp etc/cowrie.cfg.dist etc/cowrie.cfg` | `cowrie init` creates the config from the bundled template automatically |

**Real gotcha caught:** after a stop/restart, `ps aux | grep twistd` showed a
stale orphaned process still holding the port — the success message from
`cowrie stop` wasn't proof the process actually died. Had to manually `kill`
it. **Lesson: verify the process list, not just the command's exit message.**

Also expected (not a bug): moving port 22 from real sshd to Cowrie means the
SSH host key behind that port genuinely changes, since Cowrie generates its
own fake key. Any client (PuTTY, Windows OpenSSH) will show a "REMOTE HOST
IDENTIFICATION HAS CHANGED" warning on reconnect — safe to clear *only*
because the cause was known and deliberate:
```powershell
ssh-keygen -R 62.238.47.215
```

## PART 3 — THE HONEYTOKEN

Planted `/root/customer_list_private.csv` — fabricated customer data (fake
names/emails, `example.com` addresses, `555-01xx` numbers) inside Cowrie's
fake filesystem, so an attacker who logs in can find and read something that
looks worth taking.

Key mechanic: a fake file needs **two separate things** registered — a
metadata entry (so it shows up in `ls`) and loaded content (so `cat` returns
something). Missing either half is a common mistake:
```bash
fsctl var/lib/cowrie/fs.pickle
fs.pickle:/$ touch /root/customer_list_private.csv 41460
fs.pickle:/$ load /root/customer_list_private.csv ~/fake_files/customer_list_private.csv
fs.pickle:/$ exit
cowrie restart
```
Verified end-to-end via an external SSH connection: `ls -la` showed the file
at a realistic size, `cat` returned the actual fake data.

## PART 4 — FIRST CAPTURED ATTACKS (same night, before setup was even done)

**1. SSH key persistence attempt — "mdrfckr" botnet, 2 source IPs.** Two
different attacker IPs ran the *identical* command sequence within seconds
of connecting — the tell of one automated campaign hitting from multiple
compromised hosts, not two separate humans:
```
cd ~ && rm -rf .ssh && mkdir .ssh && \
  echo "ssh-rsa AAAA...mdrfckr" >> .ssh/authorized_keys && \
  chmod -R go= ~/.ssh
```
This wipes any existing keys and plants the attacker's own — a standard
persistence technique that lets them log back in anytime without needing
credentials again, even if a password is later changed. The `mdrfckr` key
comment is a recognizable signature that shows up across many public
honeypot reports — a known, widely-circulated botnet, not a targeted attack.
The planted key exists only inside Cowrie's fake filesystem; the attacker
will find nothing when they try to reconnect with it.

**2. SMTP relay pivot attempt.** An attacker logged in with guessed
credentials (`support` / `support2013`), then immediately tried to open a
connection *through* the SSH session to port 25 (SMTP) on a third-party IP —
a classic attempt to use a compromised box as a spam relay, so the spam
appears to originate from the victim's IP rather than the attacker's. Cowrie
logs this `direct-tcp` request in its default mode but never actually
forwards it — captured as evidence with zero real risk.

**3. Background scanner noise**, explicitly distinguished from the above:
most connections show no login attempt at all — a TCP connect, sometimes
just enough of a handshake to log a fingerprint, then a disconnect within
milliseconds. This is normal internet-wide cataloging (Shodan/Censys-style
scanners confirming the port is open), not deliberate attacks. Recognizing
this distinction is its own skill — not every log line is "an attacker."

## LESSONS

- **Verify a stopped process by checking the process list, not the success
  message.** `cowrie stop` reporting success didn't mean the old process was
  actually gone.
- **A changed host-key warning is expected after intentionally swapping what
  answers a port** — but only safe to dismiss when you know exactly why it
  changed.
- **Fake files need both a directory entry and loaded content** — registering
  only one half is a common, easy-to-miss mistake.
- **One attacker action can produce multiple log entries under different
  event types.** Recognizing when several data points are one story, not
  several, is part of reading logs well.
- **Not every connection is an attack.** Distinguishing automated background
  scanning from deliberate attacker behavior is a real analytical skill, not
  just noise to ignore.

---
END. Continues in session13 (systemd persistence fixes) and session14
(log-analysis training arc).
