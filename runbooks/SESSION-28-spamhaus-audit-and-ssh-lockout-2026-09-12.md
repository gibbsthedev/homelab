# SESSION 28 — Spamhaus Abuse Report, Full Host Audit, and a Real Lockout

**Date:** 2026-09-12
**Box:** Hetzner Cloud VPS `ubuntu-4gb-hel1-2` (62.238.47.215)
**Continuity:** Follows `SESSION-27-loader-capture-and-interaction-ceiling-2026-09-09.md`.
**Trigger:** Hetzner forwarded a Spamhaus abuse report alleging this IP hosts a live
`win.pure_rat` botnet command-and-controller, with a 24-hour reply deadline.

**Scope:** one abuse investigation (resolved: false positive, root-caused to session
27's Dionaea capture); one previously-unknown firewall gap discovered and fixed; one
genuine operational incident during that fix (a real, if brief, SSH lockout) — caused
by an unrelated unattended reboot colliding with the hardening work, not by any error
in the hardening itself; full recovery; and the final submitted response to Hetzner.

---

## 0. The report

Spamhaus, via Hetzner's abuse desk, alleged IP `62.238.47.215` was listed on their
Botnet Controller List (BCL) for hosting a live C2 for `win.pure_rat` malware. Hetzner
gave a 24-hour window to respond or risk the IP being locked.

**Immediate framing decision, made explicitly rather than assumed:** this could not
be treated as "probably just the honeypot being a honeypot" on the strength of that
assumption alone. A report specifically alleging a *live, currently-controlling* C2 —
not "this IP scanned us" or "this IP is on a blocklist for spam" — is a categorically
different and more serious claim. It was treated as a possible genuine compromise
until proven otherwise, not waved off.

> **LESSON — the specific wording of an abuse report changes how seriously to take
> it.** "Hosting an active C2" is a stronger claim than "sent unwanted traffic." The
> honeypot's entire purpose is to look compromised to outside observers; the question
> that actually mattered was whether it was *only* looking that way, or genuinely was.

---

## 1. Full host audit — verify before drafting anything

Ground-truth checks, run before any explanation was drafted:

```bash
ss -tulnp                          # every real listening port
ss -tlnp | grep 9988                # the specific port from session 27's capture
docker ps -a                        # confirm no unauthorized containers
ps aux --sort=-%cpu | head -30      # unexpected processes by CPU
ps aux --sort=-%mem | head -30      # unexpected processes by memory
ss -tnp | grep ESTAB                # active outbound connections
crontab -l; crontab -u root -l      # persistence
ls -la /etc/cron.d/
last -20; w                         # logins
```

**Result: clean on every check.**
- Port **9988 — the exact port session 27's captured shellcode tried to bind to —
  was completely absent** from the listening-socket table. Nothing real was ever
  bound there.
- `docker ps -a` showed only the known `dionaea` container.
- Every other listening port traced to a known, accounted-for service (Cowrie's
  `twistd` on 22/23, nginx on 80, Dionaea's documented port list via
  `docker-proxy`, mailoney on loopback).
- No cron persistence beyond Ubuntu's stock defaults.
- Process list was entirely expected service processes.

**No evidence of an actual running C2 anywhere on the host.**

---

## 2. A genuine, unrelated finding surfaced during the audit

The `ss -tnp | grep ESTAB` check turned up two simultaneous connections to the real
admin port (2200) — one confirmed as the current legitimate session, one
(`59.0.83.160`) unrecognized and still mid-authentication. Given the context (an
active abuse investigation), this was treated as the higher immediate priority than
the original report until cleared.

```bash
ss -tnp | grep 59.0.83.160
grep "59.0.83.160" /var/log/auth.log | tail -30
```

**Resolved as ordinary, unrelated internet background noise** — a mass automated SSH
scanner cycling through common usernames (`nursing`, `yuyi`, `guest`, `root`, etc.),
each attempt failing and disconnecting within ~2 seconds, the standard shape of
opportunistic scanning rather than a targeted attack. Not connected to the Spamhaus
report in any way.

**But it exposed a real, separate, worth-fixing gap:** checking `sshd_config` showed
`PermitRootLogin yes` with no `PasswordAuthentication` override — meaning password
login for `root` was genuinely permitted on the real admin port, confirmed
empirically by the log itself (a real password prompt was issued to the scanner's
attempts rather than an immediate rejection). This became the second, independent
workstream for the day.

> **LESSON — investigating one alert can surface a second, real, unrelated problem.**
> The SSH scanner had nothing to do with the Spamhaus report. Finding it anyway, by
> checking `ESTAB` connections as part of a different investigation, is exactly the
> value of doing a full audit rather than only checking the one specific thing that
> was reported.

---

## 3. The firewall that never existed

Attempting to restrict port 2200 to a known IP revealed something more fundamental:

```bash
ufw status numbered
# Command 'ufw' not found
```

**No firewall had ever been active on this box, at any point in this project.** Every
port opened for every service — Cowrie, mailoney, wp-honeypot, all fifteen-plus
Dionaea ports — has been reachable by its own service-level binding alone, with
nothing in front of it. This explained, in hindsight, why the admin port was so
easily found and brute-forced by casual internet scanning: 2200 looked like any other
open port to a scanner, because nothing distinguished it as restricted.

```bash
apt install -y ufw
```

**This single install had an unflagged side effect, caught by reading the apt output
carefully rather than skimming it:**

```text
REMOVING:
  iptables-persistent  netfilter-persistent
```

`ufw` and `iptables-persistent` are conflicting rule-persistence managers — installing
one removes the other. This mattered immediately: mailoney's SMTP redirect (real port
25 → its loopback listener on 12525) has been a manually-added `iptables` NAT rule
this whole project, kept alive across reboots specifically by `iptables-persistent`.
Removing that package meant the *next* reboot — whenever it happened — would silently
drop mail-honeypot traffic with no error message anywhere.

```bash
iptables -t nat -L PREROUTING -n -v --line-numbers
iptables-save -t nat | grep "dport 25"
```

Confirmed the live rule and captured its exact syntax before it could be lost. Fix:
fold it into `ufw`'s own `/etc/ufw/before.rules`, in a new `*nat` block placed above
the existing `*filter` block (NAT must be evaluated before filter rules), so `ufw`
becomes the single system managing all firewall/NAT state going forward rather than
two overlapping systems.

> **LESSON — read what a package manager says it's removing, not just what it says
> it's installing.** `apt install ufw`'s incidental removal of `iptables-persistent`
> would have been a silent, delayed-onset outage — invisible until the next reboot,
> whenever that happened to be.

---

## 4. The incident: an unrelated reboot collided with the hardening work

With the `ufw` rules staged (default-deny incoming, explicit allows for every
honeypot port plus the restricted admin rule) and the `before.rules` NAT fix written,
the SSH hardening was applied:

```bash
sed -i 's/^PermitRootLogin yes/PermitRootLogin prohibit-password/' /etc/ssh/sshd_config
echo "PasswordAuthentication no" >> /etc/ssh/sshd_config
```

**Total lockout followed within minutes** — every SSH attempt returned `Connection
reset` or `Connection timed out`, including from the terminal that had been open and
working the entire session.

### Diagnosis — via the Hetzner web Console, independent of network/SSH entirely

```bash
journalctl -u ssh -n 40 --no-pager
uptime
```

**The actual cause: an unattended, unrelated reboot**, triggered by
`unattended-upgrades.service` acting on the pending kernel upgrade the earlier `apt
install ufw` output had flagged (`Pending kernel upgrade! ... Restarting the system
... will not be handled automatically`). The log showed a clean `-- Boot ... --`
marker at the exact time of the lockout, and `uptime` confirmed only ~6 minutes of
runtime. **This was not caused by any error in the SSH or firewall changes** — the
box was simply down for the ~90 seconds a reboot takes, and every connection attempt
during that window failed exactly as any connection attempt to a rebooting host
would. The reboot also meant `ufw` (never yet enabled) reset to inactive, and the
`sshd_config` edit — written but not yet applied via a restart at the time of the
reboot — came back as whatever the fresh boot's defaults were.

> **LESSON — not every failure during a risky change was caused by the change.**
> The instinct when a lockout follows a security edit is to assume the edit broke
> something. Here, `journalctl`'s boot marker and `uptime`'s six minutes were the
> actual proof of an entirely separate cause. Diagnosing via the true log, rather
> than assuming the most recent action was the culprit, is what kept this from being
> mis-fixed.

### The real gap the incident revealed

Recovering via Hetzner's console and re-applying the `sshd_config` hardening (this
time actually taking effect, confirmed via `sshd -t` before restarting), a second
connection attempt was **still rejected — `Permission denied (publickey)`.**

**Root cause: no SSH key had ever existed for root.** Every login throughout this
entire project — every session in every prior write-up — had been password-based.
Disabling `PasswordAuthentication` did not hardening anything against an attacker; it
removed the *only* access method that had ever existed, and the sole surviving path
back in was the one console session still open from before the change.

> **LESSON — verify the replacement access method exists and works *before* removing
> the one currently in use.** The correct order is: generate the key, install it in
> `authorized_keys`, confirm a working key-based login from a fresh session, **then**
> disable password auth. Doing it in the other order — as happened here — makes the
> safety net's existence a matter of luck (an already-open session) rather than
> verified fact.

### Recovery

```powershell
# On the Windows machine
ssh-keygen -t ed25519 -C "rich-admin-hetzner"
type $env:USERPROFILE\.ssh\id_ed25519.pub
```

```bash
# In the still-open root session on the server
mkdir -p /root/.ssh
echo "PASTED_PUBLIC_KEY" >> /root/.ssh/authorized_keys
chmod 700 /root/.ssh
chmod 600 /root/.ssh/authorized_keys
```

`chmod` on both the directory and the file specifically — `sshd` silently refuses
overly-permissive `authorized_keys`/`.ssh` permissions, which would otherwise produce
a second, harder-to-diagnose failure that looks identical to a missing key.

A fresh third window confirmed key-based login worked. Only then was the original
session finally closed.

---

## 5. Redo — this time with a rollback safety net proven in advance

Given the day already included one real lockout, the firewall step was redone with
an automatic, guaranteed undo armed *before* the risky change, rather than relying on
a currently-open session as the only fallback:

```bash
which at || apt install -y at
systemctl enable --now atd
echo "ufw disable" | at now + 5 minutes
atq
```

If `ufw enable` locked anything out again, this scheduled job would turn the firewall
back off on its own five minutes later — no console trip required, no dependence on
any session staying open. Confirmed `atq` showed the pending job before proceeding.

```bash
ufw enable
ufw status numbered
```

**Result: clean.** Every honeypot port open to the internet as intended, port 2200
restricted to the real admin IP alone, the mailoney NAT rule correctly reloaded from
the `*nat` block in `before.rules` (confirmed via `iptables -t nat -L PREROUTING`,
rule present and back in the chain). A fresh window confirmed key-based login worked
immediately. The rollback job was then explicitly cancelled (`atrm 1`) so it
wouldn't fire and undo a working configuration.

**Full-stack verification, not just "the firewall looks right on paper":**

```bash
systemctl is-active cowrie mailoney nginx wp-honeypot   # all active
docker ps                                                # dionaea Up, correct ports
ss -tlnp | grep :12525                                   # mailoney still listening
```

Every service confirmed to have survived the reboot and the reconfiguration intact.

> **LESSON — prove the rollback works before you need it, not after.** The first
> attempt at this exact firewall change had no tested undo path and produced a real
> lockout. The second attempt, with a pre-armed `at` job, succeeded — and the
> difference wasn't the firewall rules themselves (which were correct both times),
> it was having a guaranteed recovery mechanism that didn't depend on luck.

---

## 6. Root cause conclusion and the submitted statement

**Conclusion, backed by the full audit in §1:** the Spamhaus BCL listing was very
likely triggered by Dionaea's `emu` shellcode-emulation module. Session 27's captured
loader script contained shellcode that, when emulated (Dionaea's documented,
intended behavior — tracing what a payload's Windows API calls *would* do), executed
a `bind()`/`listen()` sequence on TCP port 9988. External automated scanning
(plausibly Spamhaus's own detection infrastructure) most likely observed this
sandboxed emulation and classified it as genuine live C2 behavior. No real listener,
unauthorized process, container, persistence mechanism, or outbound C2 session was
found anywhere on the host.

**Statement submitted to Hetzner** (via the link in the original report, referencing
`[AbuseID:122097D:28]`), explaining: what Dionaea's `emu` module does and why, the
specific correlation to the September 9 loader capture and port 9988, the full list
of checks performed (listening ports, containers, processes, persistence, outbound
connections), the conclusion that no live C2 exists, and a note that the module's
network exposure is under review to prevent recurrence.

---

## 7. Open items

- **Decide on Dionaea `emu` module restriction.** The statement to Hetzner said this
  is "under review" — an actual decision (restrict its network reach further via
  Docker network policy, or accept the small recurrence risk as a cost of the
  valuable shellcode-capture data) has not been made yet.
- **Confirm the mailoney NAT rule survives a *second*, deliberate reboot.** It
  survived this session's unattended one, now correctly owned by `ufw` — worth a
  planned, controlled reboot test to be fully certain, rather than only having proof
  from one unplanned event.
- **No SSH key existed for any other purpose before this incident** — worth checking
  whether any other project machine (Proxmox, the VMs) has the same gap, given this
  was discovered here purely by accident.
- **The pending kernel upgrade** that triggered the unattended reboot — confirm the
  box is now on the expected kernel version and no further reboot is pending.

---

## 8. Lessons (consolidated)

- A report's specific wording changes how seriously to treat it — "hosting a live C2"
  warranted full incident-response rigor, not an assumed false positive.
- A full audit, done for one alert, can and did surface a second, real, unrelated
  problem — worth doing regardless of whether the original alert turns out to be
  nothing.
- Read what a package manager removes, not just what it installs — `apt install ufw`
  silently removing `iptables-persistent` was a delayed-onset outage waiting to
  happen.
- Not every failure that follows a risky change was caused by that change — verify
  the actual cause (`journalctl`, `uptime`) before assuming and re-fixing the wrong
  thing.
- Verify a replacement access method works before removing the one currently in use
  — the correct order is add-and-confirm, then remove, never the reverse.
- A tested, pre-armed rollback beats an untested one. The second firewall attempt
  succeeded specifically because of the `at`-job safety net, not because the rules
  were written any more carefully than the first time.
