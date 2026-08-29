# Session 13 — Making the Honeypot Survive: The systemd Persistence Fight

Date: 2026-08-23 into 2026-08-24

The honeypot had no persistence at first — a Hetzner reboot or crash would
just leave it dead with no auto-restart. This session covers three passes at
fixing that: an initial systemd unit that looked fine but had a hidden flaw,
a recurring bug that flaw caused, and the redesign that finally closed it
for good.

---

## PASS 1 — THE FIRST SYSTEMD UNIT (looked fine, wasn't)

Built `/etc/systemd/system/cowrie.service` using `Type=forking`, since
Cowrie's own `cowrie start` command is a wrapper script that forks into the
background — `Type=forking` is the standard systemd pattern for exactly that
shape of program. Ran as the `cowrie` user, `Restart=on-failure`,
`AUTHBIND_ENABLED=yes` baked in as an environment variable (read internally
by the `cowrie start` wrapper script to add the authbind prefix).

Added on top, once real problems showed up:
- `MemoryMax=512M` — a hard memory ceiling, added after an actual OOM
  incident. Lets systemd kill and cleanly restart Cowrie at the cap, instead
  of the kernel's blunt OOM killer potentially taking down the whole VPS.
- `KillMode=control-group` + `StartLimitIntervalSec=0` — added to fight a
  recurring stale-process bug (below). Helped, but didn't fully fix it.

## PASS 2 — THE RECURRING ORPHAN-PROCESS BUG

Hit repeatedly, in two different failure shapes:

**Shape A (early on):** after a failed start (bad config, port collision), an
*old* `twistd` process would survive holding ports 22/23, while systemd's
tracked PID was a different, already-dead process. `ss` and `systemctl
status` disagreed on which PID actually held the port.

**Shape B (more dangerous, showed up later):** `twistd` alive and genuinely
serving real attacker traffic, while `systemctl status` reported `inactive
(dead)` — because the PID systemd was tracking from the fork had been
replaced, and a completely untracked PID had taken over the listening ports.
**This meant `Restart=on-failure` would never fire** if that live-but-
invisible process ever actually died — persistence was silently non-
functional despite traffic flowing normally the whole time. A service that
*looks* healthy while its actual safety net is disconnected is worse than
one that's visibly broken.

**Root cause:** `Type=forking` requires systemd to *guess* which child PID to
track after the wrapper script forks — and that guess goes stale under
real-world conditions like crashes mid-restart or orphans from failed
starts.

## PASS 3 — THE REAL FIX: Type=simple

Stopped forking entirely and let systemd own the process directly:

```ini
[Service]
Type=simple
User=cowrie
Group=cowrie
WorkingDirectory=/home/cowrie/cowrie
Environment=PATH=/home/cowrie/cowrie-env/bin:/usr/bin:/bin
ExecStart=/usr/bin/authbind --deep /home/cowrie/cowrie-env/bin/twistd --umask=0022 --nodaemon --logger cowrie.python.logfile.logger cowrie
Restart=on-failure
RestartSec=10
MemoryMax=512M
KillMode=control-group
```

What changed and why it matters:
- **`Type=forking` → `Type=simple`** — systemd now tracks the exact process
  it launches. No fork, no PID-guessing, no staleness possible.
- **`ExecStart` calls `twistd` directly**, bypassing the `cowrie start`
  wrapper, with `--nodaemon` added (required for `Type=simple` — keeps the
  process in the foreground so systemd can own it).
- **`authbind --deep` moved into the command itself.** The old
  `AUTHBIND_ENABLED=yes` env var only ever worked because the wrapper script
  read it and added the authbind prefix internally — bypassing the wrapper
  means nothing reads that variable anymore. First attempt at this redesign
  missed that and failed with `CannotListenError: Permission denied` on
  ports 22/23, since Cowrie's non-root user can't bind privileged ports
  without authbind. Fixed by invoking `authbind --deep` explicitly.
- `ExecStop=` and `PIDFile=` removed — not needed for `Type=simple`; systemd
  sends SIGTERM straight to the PID it's tracking.

**New verification standard adopted from here on:** after any restart,
confirm the PID in `ps aux | grep twistd`, the PID in `ss -tlnp` (bound to
the actual ports), and the `Main PID` in `systemctl status` are *all the
same number*. Only that three-way match counts as real proof of health —
`systemctl status` alone is not sufficient, given it had already been shown
wrong in both directions (said dead while alive, and vice versa).

**Confirmed durable via a genuine stress test:** ran `systemctl restart`
twice back-to-back — the exact sequence that reliably orphaned processes
under the old design — and got exactly one clean `twistd` process both
times. Also survived a full unattended overnight run: one consistent PID
across all three checks, both ports live, 1038 new connections and 76 file
downloads captured with zero manual intervention.

## MAILONEY GETS THE SAME TREATMENT

Mailoney had been running manually in a foreground terminal — confirmed
dead on wake-up the next morning, as expected. Given the same `Type=simple`
pattern immediately, no history of a `Type=forking` design to fight through:

```ini
[Service]
Type=simple
User=mailoney
Group=mailoney
WorkingDirectory=/home/mailoney/mailoney
ExecStart=/home/mailoney/mailoney/venv/bin/python main.py -i 127.0.0.1 -p 12525 -s app-prod-01 --mail-dir /home/mailoney/mailoney/captured_mail
Restart=on-failure
RestartSec=10
MemoryMax=256M
```

No `authbind` needed — port 12525 is unprivileged and never internet-facing
(only Cowrie connects to it, locally). Worked cleanly on the first try.

**One self-caught mistake:** right after starting the systemd service, a
manual `python main.py` got run out of old habit — briefly created two
processes fighting over the same port. Resolved by killing the manual one
and confirming via `ps aux` that only the systemd-owned PID remained.
**Lesson: once a service is under systemd, the old manual-launch muscle
memory becomes the wrong instinct — has to be deliberately unlearned.**

## LESSONS

- **A service reporting "running" and a service actually serving traffic are
  two different claims** — `Type=forking`'s PID-guessing let them silently
  diverge. `Type=simple` removes the guess entirely.
- **The real danger case isn't the service that's visibly down — it's the
  one that looks fine while its restart safety net is quietly
  disconnected.** That's worse than an honest failure, because nothing
  flags it.
- **Verify health with a three-way PID match** (process list, socket table,
  systemd's own tracked PID), not a single status check.
- **A stress test that reproduces the original bug is the only real proof a
  fix worked** — back-to-back restarts, the exact sequence that used to
  orphan processes, now produce one clean process every time.

---
END. Continues in session14 (log-analysis training arc, deep-dive findings).
