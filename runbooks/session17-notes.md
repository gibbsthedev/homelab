# Session 17 — Honeypot Deep Dive: Filters, Confirmed Attribution, Best Captures

Date: 2026-08-25 into 2026-08-26

A long stretch of refining how to separate real humans from bots in the
live traffic, plus the strongest evidence-backed captures of the project so
far — including a fully-verified real file upload and a genuine CVE
exploitation attempt.

---

## PART 1 — REFINING THE LIVE FILTERS (three iterations)

**v1 (too chatty):** filtered on raw command vocabulary (`cat`, `nano`,
`cd`, plus common greetings). Problems surfaced immediately: `Command
found:` and `CMD:` both logged the same action, so every match printed
twice, and it missed a genuinely human greeting (`echo iH Reddit` — "hi"
spelled backwards) because the greeting list wasn't broad enough.

**v2 (the fix):** scoped to `CMD:` only (one line per action) and widened
the greeting list to match how people actually type on mobile — `hi`, `ih`,
`hii`, `hey`, `yo`. Settled workflow: run three terminal windows at once —
raw `tail -f`, a broad "interesting events" filter (downloads, uploads,
crashes), and this human-hunter filter — rather than trying to build one
filter that does everything.

**The bigger lesson, proven again here:** command vocabulary alone still
isn't identity — botnet scripts also run `cat`, `cd`, `ls`. The filters are
a triage aid to surface candidates faster, never the final verdict.
Attribution still requires reading the actual session.

## PART 2 — MANUAL ATTRIBUTION: CONFIRMED HUMANS VS. CONFIRMED BOTS

After the filters flagged candidates, each one got manually reviewed
against the intent-shaped-behavior doctrine from the log-analysis training
(persistence, payload delivery, exfiltration-shaped reads, scale). Result —
a confirmed list on each side, not a guess:

**Confirmed human, by evidence type:**
- Explicit human context stated in-session (`echo hii from my phone :3`,
  `Hello, do you like ducks?`)
- Reddit-thread readers testing a leaked IP (see Part 4 — an important
  caveat, not just a clean "human" bucket)
- Genuinely exploratory, non-destructive commands (`nano` on a decoy file,
  poking into `.aws/` out of curiosity rather than harvesting it at scale)

**Confirmed bot, by evidence type:**
- High-volume Telnet traffic running the Mirai `enable → shell → busybox`
  sequence on a loop
- The `mdrfckr` key-planting sequence, and the same sequence's heavier
  cousin, `redtail_bot`'s SSH loader
- A known cryptomining recon bot running hardware-profiling commands at
  scale (see session 14's finding on this same actor, `117.50.122.20`)

## PART 3 — THE STRONGEST CAPTURE OF THE PROJECT: A REAL FILE UPLOAD, FULLY VERIFIED

Visitor `201.222.49.29` produced the most thoroughly corroborated human
capture to date, with both behavioral and forensic evidence pointing the
same direction:

- **Session 1:** real terminal negotiation, then an unmistakably human
  greeting (explicitly mentioning being on a phone), reading the honeytoken
  CSV, general directory poking, ending on a bare `apt` before an idle
  timeout — no persistence, no payload, consistent with every other
  human-attributed session.
- **Session 3:** opened an SFTP subsystem (not a shell) and genuinely
  uploaded a file — a short MP4, ~1MB, named in a pattern typical of mobile
  screen-recorder apps.

**Forensic verification performed, hash-first, never executed or opened
directly:**
- File size and magic-byte check (`head -c 32 | xxd`) confirmed a genuine
  MP4 container — not a renamed executable.
- SHA256 checked against VirusTotal — clean, zero vendor flags.
- Only after every check passed was the file pulled to an isolated machine
  and actually played, confirming it was exactly what it claimed to be: a
  short kitten video.

This is the reference example for what a fully-closed-loop "confirmed
benign" writeup looks like — distinct from most other human-traffic
entries, which are well-argued but still somewhat inferential.

## PART 4 — AN IMPORTANT CAVEAT: REDDIT CONTAMINATION

The honeypot's real public IP got posted on a subreddit, in a thread where
the original poster was asking for help logging into what they believed was
their *own* broken server — including their actual attempted root password
in the comments. For some window of time, some traffic hitting the honeypot
was curious Reddit readers testing that IP, not deliberate attackers.

Two sessions initially flagged as possible sophisticated human attackers
got **re-attributed** to this source after review — their behavior (typing
`dir` out of Windows muscle memory, then correcting to `ls`; checking
whether `apt update` worked, as if troubleshooting "their own" box) read as
someone trying to fix a server they believed was theirs, not as an attacker
demonstrating tradecraft.

**This is a genuinely useful control group, but only once the distinction
is made correctly** — treating Reddit-sourced curiosity as sophisticated
attacker behavior would have been a real attribution error, caught by
paying attention to the specific shape of the commands rather than just
counting them as "interactive activity."

## PART 5 — A REAL CVE EXPLOITATION ATTEMPT, CAUGHT CLEAN

Cowrie's own exploit detector fired correctly on a live attempt:
```
src_ip: 31.76.20.19
protocol: telnet
eventid: cowrie.telnet.exploit_attempt
cve: CVE-2026-24061
```
The attacker sent the Telnet negotiation sequence associated with that
CVE, then a crafted `USER=-f root` string — a real, named vulnerability
being probed, not a generic credential guess. No shell followed; it was a
probe, not a full exploitation chain, but the emulated environment
correctly recognized and logged it as the specific CVE it was.

## PART 6 — MALWARE INVENTORY, EXPANDED

Filtered downloads by size (anything under ~10KB is usually a marker or
failed grab, not a real payload) and cataloged the rest: multiple
architecture-specific Mirai-family ELF binaries (ARM, MIPS, i386), several
dropper shell scripts, a captured OpenSSH keypair tied to `redtail_bot`'s
own persistence attempt, and the earlier-confirmed cryptominer sample.
Every file gets hashed and checked, never executed — the standing rule from
day one of the build.

## LESSONS

- **A filter is a triage tool, not a verdict.** It narrows a large log down
  to candidates worth reading — attribution still requires actually reading
  the session.
- **`Command found:` and `CMD:` can double-log the same action** — scope a
  filter to one event type to avoid counting things twice.
- **Publicity changes your traffic mix.** Once a honeypot's IP is shared
  publicly, some incoming traffic will be curiosity, not attack — and
  conflating the two is a real analytical error, not a harmless one.
- **A "confirmed benign" finding needs both behavioral AND forensic
  evidence pointing the same way** — content that states human context
  plus an independently verified, clean file is a much stronger claim than
  either alone.

---
END. Continues in session18 (redtail_bot OOM incident and outage response).
