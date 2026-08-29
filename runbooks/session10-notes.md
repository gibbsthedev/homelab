# Session 10 — Migrating Hermes Off AWS: RHEL VM Build to Gateway Cutover

Date: 2026-08-21 (overnight session)

Hermes was the last major AI agent still living on the AWS EC2 box. This
session built a new RHEL VM on pve-node1 from scratch and moved Hermes's
entire gateway — state, credentials, dependencies — onto it, ending with a
live cutover verified by actually messaging the bot, not just checking that
a service said "running."

---

## PART 1 — WHY A NEW VM, AND WHY RHEL

Rather than repurpose an existing VM, Hermes got its own: **VM 101, RHEL
10.2**, registered under a free Red Hat Developer subscription. Real RHEL
(not a rebuild like Rocky) matters for resume/certification relevance — it's
what a lot of enterprise environments actually run.

Three users had to be recreated with **exact matching UIDs/GIDs** from the
AWS box, because Hermes had hardcoded `/home/ec2-user` paths in its config:

- `ec2-user` — UID 1000, GID 1000
- `asl-app` — UID 989, GID 988
- `hermes-asl` — UID 988, GID 986 (note: `asl-app`'s GID equals `hermes-asl`'s
  UID — deliberate, not a typo, and had to be created in the right order)

Getting these wrong would have silently broken every path reference in
Hermes's config without necessarily erroring loudly — matching them exactly
was cheaper than debugging permission ghosts later.

## PART 2 — THE TRANSFER: WHY PUSH, NOT PULL

The original plan was to pull data from AWS the same way OpenClaw's earlier
migration worked. This time it failed — AWS's SSH config was old and tangled
from months of ad-hoc changes, and pulling from it kept hitting silent auth
failures.

**Lesson: when one direction of an SSH transfer is a fight, flip the
direction.** Pushing *from* AWS *to* the clean new VM worked immediately.
General rule going forward: don't debug the messy end when you can push from
it instead.

One transfer bug worth remembering: exclude patterns have to match exact
directory names. `.cache/` does not match `audio_cache/` — the fix was a
wildcard pattern (`*cache*/`), and the first dry-run transfer came out
roughly 2x larger than it should have been until that was caught.

## PART 3 — REBUILDING, NOT COPYING, THE RUNTIME

Python and Node dependencies were rebuilt fresh on the new VM rather than
copied over, to avoid architecture/library mismatches:

```
python3 -m venv venv
source venv/bin/activate
pip install -e .
```

Node hit a real error on first try:
```
npm install
# FAILS: node-pty native build error — "not found: make"
```
Minimal RHEL has no compiler installed by default. Fix:
```
sudo dnf install -y gcc gcc-c++ make
npm install
```
`node-pty` is the module Hermes uses for shell/PTY features — a real
dependency, not something to skip.

**Deliberately avoided** `hermes update` and `npm audit fix --force` during
the migration. The goal was matching AWS's exact known-good versions, not
getting the newest/cleanest ones — auto-upgrading mid-migration risks
breaking something that worked, for no benefit.

## PART 4 — CUTOVER: PROVING IT WORKS, NOT JUST THAT IT STARTED

First attempt at starting Hermes on the new VM (while AWS was still running)
failed on purpose — Telegram rejected the connection with a polling conflict,
because AWS still held the bot token. This confirmed a real constraint: only
one instance can poll a given Telegram bot token at a time, which fixed the
required cutover order.

Sequence used:
1. Stop the AWS-side gateway.
2. Run one final delta sync (now safe — nothing on AWS is still writing to
   the database).
3. Start the gateway on the new VM in the foreground — Telegram connected
   cleanly this time.
4. **Function test: sent the bot a real message and got a real reply from
   the new VM.** This is the actual proof of cutover, not the service status.
5. Installed it as a proper `systemctl --user` service with lingering
   enabled, so it starts on boot without a login session.
6. **Rebooted the VM and confirmed it came back on its own** — Hermes
   resumed responding on Telegram with zero manual intervention. This is
   what "durable across power cycles" actually looks like, versus "worked
   once in this terminal session."

One extra caution baked in here: the live SQLite databases (state, kanban,
verification evidence) were only trustworthy once copied *after* the AWS
gateway stopped writing to them. Copying "live" databases while the old
side is still active risks partial/inconsistent files.

## PART 5 — WHAT'S STILL OPEN

- **Trading bot's final home is unconfirmed.** It's currently paused (0 of
  24 scheduled jobs enabled) — deliberately not decided yet, since the money
  involved is minimal and there's no rush to get this wrong.
- AWS EC2 was kept fully intact as a fallback — nothing decommissioned yet.
- Several web apps/APIs were still being migrated over as this session
  closed; each one gets verified by actually loading it, not by trusting a
  status report.

## LESSONS

- **Verify by function, not by status.** "Service running" and a confident
  status report are not proof something works — sending a real message and
  getting a real reply is.
- **When an SSH transfer direction is fighting you, flip it.** Push from the
  messy/old side rather than pulling into the clean new one.
- **A reboot is the real durability test** for anything installed as a user
  service with lingering. If it comes back on its own, it's actually
  permanent — not just working in the current session.
- **Match known-good versions during a migration; don't auto-upgrade.**
  "Newer" and "audit-clean" are goals for later, not during a cutover.
- **Minimal RHEL installs have no compiler by default** — native npm/pip
  modules need `gcc gcc-c++ make` installed first.

---
END. See `services/hermes-vm.md` for the current-state reference (to be added).
