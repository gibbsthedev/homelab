# Session 11 — RAM Ballooning Investigation and the Case for a Managed Switch

Date: 2026-08-23

Two connected pieces of work: figuring out whether Proxmox's RAM allocation
was actually a problem, and then using that answer to make a deliberate call
on what hardware to buy next.

---

## PART 1 — THE RAM QUESTION

`free -h` on the Proxmox host showed 15Gi total, 10Gi used, only 2.7Gi
"free" — but 5.5Gi genuinely "available" once reclaimable buffer/cache is
counted. Available, not free, is the number that actually matters on Linux.

Checking memory *inside* both VMs told a different story:
- **VM 100 (OpenClaw):** 5.8Gi allocated, only 1.1Gi actually used
- **VM 101 (Hermes):** 3.6Gi allocated, only 932Mi actually used

Both guests had plenty of headroom internally, yet the host showed the full
10GB pinned and unavailable. Root cause: **KVM reserves a VM's full
configured RAM the moment it boots**, regardless of how much the guest
actually uses — that's a static reservation, not a live number. The host's
tightness was a configuration artifact, not real demand.

Checked `qm config` for both VMs — no `balloon:` line existed for either. In
Proxmox, no `balloon:` line silently defaults to the max, so even if
ballooning looks enabled in the hardware tab, there's no floor set below the
max for Proxmox's stat daemon to actually shrink toward. Functionally
identical to no ballooning at all.

**Decision: left it as-is.** A `balloon` value could be set to give the host
real headroom back, but current usage is light and stable — the added
complexity wasn't worth it yet. Flagged to revisit if VM memory pressure
increases.

## PART 2 — WHAT TO BUY NEXT

Framed as a direct question: given a limited budget, what's the next
highest-value hardware purchase? Four real candidates were weighed:

- **A second compute node** — rejected on the same measured-need principle
  from Part 1: both existing VMs are using a fraction of their allocation.
  No demonstrated need for more compute.
- **More storage** — no actual pressure (plenty of free space after the
  Hermes migration), and SSD prices were running well above normal due to a
  2026 supply shortage. Bad timing either way.
- **A UPS** — genuinely useful insurance against dirty shutdowns and disk
  corruption, but nothing had actually failed from lacking one yet. Not
  urgent.
- **A managed switch — the winner.** Directly serves the Network+ / SOC
  learning track: VLANs and subnetting make up a real chunk of the exam, and
  the lab had zero hands-on network segmentation in it at the time.

**Bought:** a Cisco SG300-10 (SRW2008-K9 V02), $29.98, from a verified
reputable seller. Purchased but not yet deployed — VLAN segmentation is
flagged as the next major infrastructure phase once other work settles.

## LESSONS

- **"Free" memory and "available" memory are different numbers** — available
  (which counts reclaimable cache) is the one that matters.
- **KVM's RAM reservation is static at boot, not live** — a VM shows as using
  its full allocation on the host side even if the guest itself is nearly
  idle. Don't diagnose host memory pressure without checking inside the
  guests too.
- **Measured need beats felt need.** Buying more compute "because it feels
  tight" would have been the wrong move — the actual data said otherwise.
- **Tie hardware purchases to the stated goals.** The switch won not because
  it was cheapest or flashiest, but because it was the one purchase that
  directly built toward the Network+/SOC-analyst track.

---
END. Pair with `hardware/` for the switch's own spec entry once deployed.
