# SESSION 56: pve-node1 RAM 16 → 64 GB, OpenClaw back online, Hermes to 20 GB
**Date:** 2026-11-13 (evening, about 18:40 → 21:00 CST)
**Machines touched:** pve-node1 (M720q), VM 100 docker-agents (OpenClaw), VM 101 hermes, Mint (checked only)
**Identifiers:** serial numbers, MAC addresses and FRU labels are written as `xxxx`.

**Continuity:** carries out the RAM plan from `SESSION-55` §1 and reverses the parking from
`SESSION-48`/`SESSION-49`. RAM was the reason OpenClaw was parked. With 64 GB, both agent VMs fit on one node again.

---

## 1. What I had (the RAM lot from SESSION-55)

| Stick | Size | Spec |
|---|---|---|
| A-Tech | 16 GB | DDR4 PC4-21300 (2666) SO-DIMM, 2Rx8, 1.2 V |
| A-Tech | 16 GB | same |
| Samsung M471A4G43MB1-CTD | 32 GB | DDR4 PC4-2666V SO-DIMM, 2Rx8 |
| A-Tech | 32 GB | DDR4 PC4-21300 (2666) SO-DIMM, 2Rx8, 1.2 V |

**Placement decision:** the two 32 GB sticks go into pve-node1 (64 GB). It's the hypervisor, and Wazuh (Phase 4)
and the purple-team targets will land on it. Mixing Samsung and A-Tech in one pair is fine because size, speed,
rank layout and voltage all match. The 2×16 GB pair is spare for t640-mon (4 → 32 GB) in a later session.

**Mint laptop checked too:** `dmidecode -t memory` shows **DDR3**. DDR3 and DDR4 notches are in different places,
so none of these sticks fit. It stays a cockpit machine.

---

## 2. Before touching hardware

### 2.1 Make sure no backup is running
```bash
# pve-node1
pgrep -a vzdump                                           # no output = no backup running
grep -i "finished\|error" /var/log/vzdump/qemu-101.log | tail -3
tail -5 /var/log/vzdump/qemu-101.log                      # "Finished Backup of VM 101" ✅
```
Pulling power during a vzdump leaves a broken dump. With a short keep-last, that can rotate out a good copy.

### 2.2 Finding: the guest agent can't freeze one of Hermes's filesystems
The latest VM 101 backup log showed:
```
ERROR: unable to freeze guest fs … failed to open /srv/storage: permission denied
```
- **What it means:** before a snapshot, Proxmox asks the QEMU guest agent inside the VM to briefly freeze
  ("fs-freeze") the filesystems so the backup is application-clean. It couldn't open `/srv/storage`, so the
  freeze failed. **vzdump carried on anyway**, so the backup is still good, just *crash-consistent*
  (like pulling the power and booting back up) rather than *freeze-consistent*.
- **Likely cause (not yet confirmed):** Hermes is RHEL 10, and SELinux confines `qemu-guest-agent`. A directory
  that isn't labeled for the agent gets refused even though the agent runs as root.
- **Next session:** check `ausearch -m avc -ts recent | grep qemu-ga` on Hermes, then fix the label or
  policy, and confirm the next backup log has no freeze error.

### 2.3 Baseline: what's installed now
```bash
dmidecode -t memory | grep -E "Size:|Locator:|Speed:"
```
| Slot | Size | Speed | Configured speed |
|---|---|---|---|
| ChannelA-DIMM0 (BANK 0) | 8 GB | 2667 MT/s | 2666 MT/s |
| ChannelB-DIMM0 (BANK 2) | 8 GB | **3200 MT/s** | 2666 MT/s |

The second 8 GB stick was rated 3200, but it ran at 2666 because that's the fastest the **i5-8400T** supports.
So the new 2666 sticks give up no speed.

### 2.4 Clean shutdown
```bash
qm shutdown 101          # guest shuts itself down
qm list                  # 101 stopped
shutdown -h now
```
Unplugged power, touched metal before handling the sticks.

---

## 3. The swap

### 3.1 Finding the slots: they're on the bottom
I first opened the **top** of the M720q: fan, heatsink and the black 2.5" drive caddy. No RAM there.
**On the M720q the SO-DIMM slots and the M.2 NVMe are on the underside.** Flip it over and remove the bottom cover.

Underside layout:
```
┌───────────────────────────────────────────────┐
│  [ RAM upper slot ]   ← stacked over the lower │   [ M.2 NVMe (Proxmox) ] ← blue latch
│  [ RAM lower slot ]                            │   [ SSD2: empty M.2 slot ]
└───────────────────────────────────────────────┘
```
The empty **SSD2** M.2 slot is a future storage upgrade (second NVMe), in addition to the empty 2.5" bay.

### 3.2 Remove and install
- Removed the upper stick first (it sits over the lower one), then the lower one. Clips outward, the stick tilts up about 30°, pull it out along that angle.
- Installed the lower slot first, then the upper one: insert at about 30°, push until the gold edge almost disappears, press flat until both clips snap.

### 3.3 First power-on: immediate beeps
It **beeped right away** with no display. On ThinkCentres, beeps before video almost always mean
**no usable memory found**.

The question was whether it was **seating, a bad stick, or the BIOS rejecting 32 GB modules**
(Lenovo's official M720q limit is 32 GB total). I tested one variable at a time:

| Test | Result | What it proves |
|---|---|---|
| One 32 GB stick, lower slot only | **Booted** | The BIOS accepts 32 GB modules → **no BIOS update needed** |
| Both sticks reseated | **Booted, 64 GB** | Neither stick is bad → the first failure was **seating** |

With one stick, `dmidecode` showed `ChannelA-DIMM0: 32 GB` and `ChannelB-DIMM0: No Module Installed`, and Proxmox booted normally.

**The BIOS update from the SESSION-55 plan was skipped.** It was a precaution, and the one-stick test showed it wasn't needed.

### 3.4 Verified
```bash
dmidecode -t memory | grep -E "Size:|Locator:|Speed:"
```
| Slot | Size | Speed | Configured speed |
|---|---|---|---|
| ChannelA-DIMM0 (BANK 0) | **32 GB** | 2667 MT/s | 2666 MT/s |
| ChannelB-DIMM0 (BANK 2) | **32 GB** | 2667 MT/s | 2666 MT/s |
| **Range Size** | **64 GB** | | |

Dual-channel at full speed. Hermes (101, `onboot 1`) came back up on its own.

---

## 4. VM memory: OpenClaw back, Hermes grown

### 4.1 Sizing
| | Before | After |
|---|---|---|
| VM 101 hermes | 12 GB | **20 GB** (20480 MiB) |
| VM 100 docker-agents (OpenClaw) | 5 GB (5120 MiB, parked) | **12 GB** (12288 MiB), running |
| Proxmox host | | about 4 GB |
| **Free for Wazuh and lab VMs** | | **about 28 GB** |

KVM reserves a VM's **full** configured RAM when it starts (no ballooning configured), so the
allocations, not the guests' actual usage, decide what's left on the host.

### 4.2 OpenClaw: unparked
```bash
qm set 100 --memory 12288 --onboot 1
qm start 100
```
- I considered booting it with its virtual NIC down (`link_down=1`) so I could inspect scheduled jobs before it
  reached the network. **Skipped:** OpenClaw has no scheduled jobs left and cleans up after itself.

### 4.3 Hermes: 12 → 20 GB
Memory can only change while the VM is off (no memory hotplug), so the public sites were down for about a minute.
```bash
qm shutdown 101
qm list                    # 101 stopped
qm set 101 --memory 20480
qm start 101
```

### 4.4 Verified
```bash
qm list
```
| VMID | Name | Status | MEM (MB) |
|---|---|---|---|
| 100 | docker-agents | running | 12288 |
| 101 | hermes | running | 20480 |

---

## 5. State at end of session

| | Value |
|---|---|
| pve-node1 RAM | **64 GB** (2×32 GB DDR4-2666, dual-channel) |
| VM 100 OpenClaw | **Running**, 12 GB, `onboot 1` |
| VM 101 Hermes | Running, **20 GB**, `onboot 1` |
| Host headroom | about 28 GB |
| Spare RAM | 2×16 GB DDR4-2666 (for t640-mon), 2×8 GB DDR4-2666 (old pve-node1 sticks) |

---

## 6. Lessons learned
- **Know where the slots are before you open the case.** The M720q's RAM and M.2 are on the bottom.
- **Beeps before video = memory not found.** Check seating before blaming the hardware or the BIOS.
- **Change one variable at a time.** One stick in the primary slot answered "is it the BIOS?" in one boot.
- **Don't do an upgrade you haven't proven you need.** The planned BIOS update wasn't necessary.
- **A stick's rated speed isn't the speed it runs at.** The CPU's memory controller sets the ceiling (2666 on the i5-8400T).
- **Check for a running backup before a planned shutdown.**
- **Read the warnings in a "successful" log.** The backup finished, but the freeze error means it isn't application-clean.
- **Capture a `dmidecode` baseline before the change**, so the "after" proves something.

## 7. Next
1. **memtest86+**: one full pass, 0 errors, before Wazuh goes on this box. The first boot failed, so verify the memory, not just the slot count. (Boot menu → Memory test. Takes 1–2 h, and both VMs are down during it.)
2. **LibreNMS:** re-enable docker-agents (disabled when it was parked).
3. **Backups:** add VM 100 back to the nightly vzdump job. Recheck the backup math: dump sizes × (keep-last + 1) must fit on `local`. Keep the R2 sync at one object, within the free tier.
4. **OpenClaw housekeeping:** pending kernel reboot (6.12.111 → .115), the 97.9% backup disk (vm-100-disk-1), UFW OpenSSH from Anywhere → tighten. Remove the Proxmox snapshot of VM 100 once it's healthy (a snapshot isn't a backup, and it overcommits the thin pool).
5. **Hermes:** the guest-agent freeze error on `/srv/storage` (§2.2).
6. **t640-mon:** install the 2×16 GB pair (4 → 32 GB).
7. **Phase 4:** Wazuh SIEM on pve-node1. It's no longer blocked on RAM.
