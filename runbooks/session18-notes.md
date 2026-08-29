# Session 18 — The redtail_bot Payload Drop and the OOM Outage

Date: 2026-08-26

Two threads that turned out to be connected: a threat actor (redtail_bot)
dropping a full multi-architecture malware kit over SFTP, and a real
outage — Cowrie repeatedly getting killed by the kernel — that took real
diagnosis to root-cause and fix properly.

---

## PART 1 — REDTAIL DROPS THE FULL KIT

`77.90.185.20` connected, authenticated as `root` instantly (wide-open
userdb doing its job), and used an SFTP subsystem — not a shell one-liner —
to push a complete kit onto the honeypot:

- `clean.sh` — disables competing malware/miners already "installed"
  (`systemctl disable/stop` on rival services), strips cron and `.bashrc`
  of other backdoor patterns, wipes `/tmp`/`/var/tmp`/`/dev/shm` except its
  own directory. **Classic botnet turf war behavior** — clearing out
  competitors so this payload owns the CPU uncontested.
- `setup.sh` — detects CPU architecture, finds a writable+executable
  directory (explicitly skipping `noexec` mounts), copies the matching
  architecture binary to a randomized hidden filename, and launches it.
- An `authorized_keys` file with an RSA key whose comment matches
  previously published threat-intel writeups on this same botnet family.
- **Five architecture-specific ELF binaries** — ARM (32/64-bit), i386,
  RISC-V, and x86_64 — all static, stripped.

**Same operator, later stage:** the SSH client fingerprint (hassh) and
Go-based client signature matched an earlier-seen actor (`130.12.180.51`,
the one that embedded a key and pulled a loader from a separate C2 host) —
confirming this was the same campaign progressing to its payload-delivery
stage, not an unrelated new actor.

**VirusTotal came back empty on the hashes** — expected, not a failure of
the process. This malware family is known to rebuild its ELF payloads
frequently, so binary hashes drift while the surrounding artifacts stay
stable. Correct identification method here is the stable fingerprints —
filenames, the SSH key comment, the hassh, the C2 address — not the binary
hash alone. **Lesson: a hash miss on VirusTotal doesn't mean "unknown
malware," it can mean "known family, freshly rebuilt."** Everything was
hashed, cataloged, and left unexecuted regardless.

## PART 2 — "SOMETHING KEEPS TAKING THE SERVER DOWN"

Investigating apparent downtime, ruled out the obvious suspects first:
not a disk-space crash (downloads and logs were both well under capacity),
not a duplicate-process conflict, not mailoney (idle and light). The actual
cause, confirmed via `dmesg -T`:

```
oom_kill_process constraint=CONSTRAINT_MEMCG
oom_memcg=/system.slice/cowrie.service
Killed process (twistd) anon-rss:~522988kB
```

**systemd's own memory cgroup cap was killing Cowrie**, over and over, each
time it grew to almost exactly the `MemoryMax=512M` ceiling set back during
the persistence work. `Restart=on-failure` then brought it straight back
up, which grew again, which got killed again — a loop that made the
honeypot look intermittently "down" from the outside while the host itself
stayed perfectly healthy the whole time.

**Root cause of the growth, traced to actual traffic:** a cluster of four
sequential IPs in one /24 block, all sharing the same SSH client
fingerprint — one operator running from four adjacent hosts — ran roughly
840 single-command sessions (`uname -a`) over 9.5 hours, starting right at
midnight. Each session means Cowrie builds a fresh simulated shell
environment; at that volume, with imperfect per-session cleanup, memory
climbed until it hit the cap.

**An early misreading worth documenting:** RSS sitting pinned right at the
cap looked, at first glance, like Cowrie was mid-crash-loop. Checking
process *uptime* (not just memory) disproved that — the process had
actually been stable for 6+ hours, plateaued at the ceiling rather than
still climbing. **Lesson: check how long a process has been running before
concluding it's actively failing — a memory number alone can't distinguish
"stable at a hard cap" from "about to die again."**

## PART 3 — THE FIX, AND A REAL GOTCHA IN APPLYING IT

Raised the memory ceiling via a systemd override rather than editing the
unit file directly, so a future package update or reinstall can't silently
wipe the change:
```bash
sudo systemctl edit cowrie.service
```

**The gotcha:** `systemctl edit` opens an editor showing the *current* unit
file's settings as commented-out reference lines. Those commented lines are
not live config — they're just shown for reference. An empty override (or
one left fully commented out) changes nothing at all, silently. The actual
override has to be typed in as real, uncommented lines:
```ini
[Service]
MemoryMax=1536M
MemoryHigh=1200M
```
```bash
sudo systemctl daemon-reload
sudo systemctl restart cowrie.service
systemctl show cowrie.service -p MemoryMax,MemoryHigh,MemoryCurrent
```
Confirmed applied and holding steady well under the new ceiling afterward —
no more kill-restart cycling.

**Considered, not done:** adding 1GB of swap as an extra safety margin,
since the host currently runs with zero swap. Decided unnecessary for now
given the new cap already resolved the actual symptom — flagged as a
future hardening step if memory pressure returns.

## LESSONS

- **A honeypot "going down" can be the safety mechanism working correctly**
  — the memory cap doing exactly its job (protecting the host) can still
  look, from outside, indistinguishable from an actual failure. Root cause
  before assuming the safeguard itself is the problem.
- **A VirusTotal hash miss isn't proof of an unknown threat.** Some malware
  families rebuild their binaries often; identify by the stable artifacts
  (filenames, key comments, client fingerprints, C2 addresses) alongside
  the hash, not the hash in isolation.
- **`systemctl edit` shows commented reference lines from the current
  config — those aren't the file being written.** An override has to
  contain real, active directives, or it silently does nothing.
- **Process uptime, not just current memory, distinguishes "stable at a
  cap" from "actively crash-looping."** Check both before diagnosing.
- **Sequential IPs sharing one client fingerprint are one operator, not
  several** — the same correlation technique from the log-analysis
  training arc, applied here to root-cause an actual outage instead of
  just documenting an attacker profile.

---
END. This closes out the honeypot-arc catch-up — sessions 12 through 18 now
cover the full build, hardening, training, and this incident.
