# Session 15 — Syncing node1.md: From a Month-Old Doc to Current State

Date: 2026-08-23

A pure documentation session — no infrastructure changed. The goal was
bringing `hardware/m720q.md` (frozen at its early-July state) fully current
through the Hermes cutover, by cross-referencing a hardware photo, an
earlier build-session transcript, and the up-to-date project notes.

---

## WHY THIS MATTERED

`node1.md` still only described the original single-VM setup — it had no
record of Hermes, the RAM investigation, the NIC fix, or the backup system,
even though all of that had already happened and was verified. A doc that's
a month behind doesn't just look outdated — it actively hides real state
from anyone (including a future me) trying to understand the lab from the
repo alone.

## WHAT GOT ADDED OR CORRECTED

- **RAM closed out definitively via `dmidecode`**, not inference: confirmed
  2×8 GB DDR4-2666, both slots physically full (ChannelA-DIMM0, ChannelB-
  DIMM0). This matters for the upgrade path — with no free slot, reaching
  32 GB means removing both existing sticks and installing a fresh 2×16 GB
  kit, not just adding one.
- **Connectivity section written for the first time**: the dead apartment
  wall jack, the GL.iNet Beryl AX repeater path, the private 192.168.8.0/24
  LAN, and the static node1 address.
- **Proxmox post-install detail captured**: default ext4 + LVM-thin storage
  (inferred from `local-lvm` being present, not ZFS as originally intended
  for the lab — fine on a single disk since there's no redundancy either
  way, flagged to revisit if a second disk gets added), no-subscription
  repo switch, git + SSH key setup.
- **VM 100 corrected to reality**: it's OpenClaw's home now, not a generic
  Docker box — resized down to 3 vCPU/6GB, moved to a static IP, given a
  second no-backup disk for its own internal backups.
- **VM 101 (Hermes) documented for the first time** — the whole new RHEL VM
  didn't exist in this doc at all until now.
- **Host-level infrastructure added**: the e1000e NIC driver bug and its
  persistent fix, and the vzdump + Cloudflare R2 backup pipeline.

## A REAL DECISION MADE ABOUT WHAT *NOT* TO WRITE DOWN

Some open questions from the KB — where the trading bot ends up living, the
exact ownership/authority split between OpenClaw and Hermes — were
deliberately left **out** of `node1.md`, even though they were fresh in
mind. **Rule applied: a machine doc records confirmed facts; unsettled
decisions stay in the KB's decision log until they're actually resolved.**
Baking a proposal into a hardware doc as if it were settled fact creates a
different, quieter kind of staleness than simply falling behind on time.

## OPEN ITEMS FLAGGED FOR LATER

- Storage layout (ext4+LVM vs. ZFS) was inferred, not directly verified —
  worth a direct check.
- Some VM 100 specifics came from a session summary rather than the raw
  transcript — worth a 10-second confirm against `qm config 100` directly.
- **The KB itself is still stale** — PART 4 and PART 9 still describe
  Hermes as "being built." Not fixed this session; still open.
- A dedicated `network.md` for the Beryl AX / topology was proposed, since
  that's shared lab infrastructure rather than something specific to this
  one machine — not done yet.

## LESSONS

- **`dmidecode` is the source of truth for hardware — not a photo, not
  memory of what was ordered.** Confirmed vs. assumed is worth the extra
  ten seconds.
- **A doc a month behind hides real state, it doesn't just look old.**
  Keeping per-machine docs moving with the actual build — not batched up for
  later — is what keeps them trustworthy.
- **Confirmed and proposed belong in different places.** A machine doc
  should only ever say what's true right now; anything still being decided
  belongs in a decision log, clearly marked as unsettled.

---
END. Paired with the current `node1.md` in `hardware/` as the artifact of
this session.
