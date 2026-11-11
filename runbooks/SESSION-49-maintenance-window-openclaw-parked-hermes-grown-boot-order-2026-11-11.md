# SESSION 49 — Maintenance Window: OpenClaw Parked, Hermes Grown, Proxmox Updated, Boot Order Fixed — 2026-11-10 → 2026-11-11

**Status: COMPLETE, one item left (BIOS change + watched reboot).** pve-node1 now
belongs to Hermes. OpenClaw (VM 100) is parked with a clean shutdown. Hermes (VM 101)
got a bigger disk and three times the RAM. Proxmox got its first host update and
reboot in more than a month. The reboot hit the PXE network-boot screen for the
second time; the cause was found and the boot order was fixed from Linux.

**Continuity:** follows `SESSION-48`, which planned the parking. That plan said
"later this week"; it happened the same night.

---

## 1. Backups before touching anything

| Copy | Where | Notes |
|---|---|---|
| VM snapshot | Copied to the Windows PC | Off the node, so it survives a disk failure |
| OpenClaw backup | On Hermes | Hermes took this as part of its inventory |
| Proxmox snapshot of VM 100 | On the thin pool | See §3 |

Rule followed: **two copies, one of them off the machine**, before any shutdown or resize.

---

## 2. OpenClaw (VM 100) parked

```
qm shutdown 100          # clean shutdown: the guest OS stops itself
qm set 100 --onboot 0    # don't start it again when the host boots
```
- LibreNMS: docker-agents **disabled** (not deleted). A switched-off device would
  otherwise alert "down" forever. Disabling keeps its graphs and history.
- **Clean shutdown, not Hibernate** (SESSION-48 §2): saved RAM state is tied to the
  QEMU/Proxmox version, and Proxmox was about to be updated.
- Both disks are intact. Bringing it back = `qm start 100`, and **lower Hermes's RAM first** (§5).

---

## 3. The thin pool warning, explained by a snapshot

A Proxmox snapshot of VM 100 shows up in `lvs` as extra thin volumes
(`snap_vm-100-disk-0`, `snap_vm-100-disk-1`, about **48 GB** promised).

| Fact | Meaning |
|---|---|
| A thin volume **promises** space; it only **uses** blocks that were written | The "48 GB" is a promise, not 48 GB used |
| A snapshot only grows when the original disk **changes** | VM 100 is off → nothing changes → the snapshot stays nearly free |
| Proxmox warns when promises add up to more than the pool | That's the warning. It's **overcommit**, not a full pool |

**Decision: keep it** while VM 100 is parked. It's a quick rollback point and costs
almost nothing. **But a snapshot is not a backup:** it lives on the same NVMe, so if
the disk dies, the snapshot dies with it. The off-node copy is the backup, and it
should be a **vzdump file** (`.vma.zst`) that Proxmox can restore directly.

---

## 4. Hermes (VM 101): disk grown 32 → 52 GB

Why: the root disk alerted at ≥ 90% (SESSION-47). Hermes's cleanup bought time
(87%), but the disk keeps growing about 7–8 GB a month.

Four layers, grown from the **outside in**:

```
Proxmox virtual disk   → qm disk resize 101 scsi0 +20G      (on the host)
partition              → growpart  <disk> <partition-number> (in the VM)
LVM physical volume    → pvresize  <partition>
LVM logical volume +   → lvextend -r -l +100%FREE <root LV>
XFS filesystem           (-r grows the filesystem in the same step)
```

| Layer | Analogy |
|---|---|
| Virtual disk | A bigger lot |
| Partition | Moving the fence out to the new edge |
| PV / LV | Telling the building plan it has more room |
| Filesystem | Actually building on it |

Skip any layer and the extra space stays invisible to the one above it.
**XFS can grow but never shrink**, so size up in steps you can live with.

**Result:** root **47 GB, 54% used** (was 85%+ of 32 GB).

---

## 5. Hermes RAM: 4 GB → 12 GB

```
qm set 101 --memory 12288
```
Takes effect on the VM's next full stop/start (the host reboot did that).

**Watch out:** the host has 16 GB. Hermes 12 + OpenClaw 6 = 18 GB > 16 GB.
**Before OpenClaw ever starts again, lower Hermes first**, or give the node
the planned 2×16 GB upgrade.

---

## 6. Proxmox host update + reboot

- `apt update && apt full-upgrade`: about **651 MB**, including an **rsync** security fix.
  On Proxmox, always `full-upgrade` (plain `upgrade` can hold back kernel and
  Proxmox packages whose dependencies change).
- Host uptime was more than a month (SESSION-46 §8), so newer kernels were installed but not running.
- **Kernel after reboot: `7.0.14-28-pve`** (was `7.0.14-5-pve` in SESSION-46).
- Hermes came back by itself (`onboot 1`); Rich verified its services and sites.
  OpenClaw stayed off, as intended (`onboot 0`).

---

## 7. PXE on reboot — second time — cause found

### Symptom
The host stopped on a **PXE network-boot** screen (asking the network for an
operating system). Pressing a key → a GRUB prompt → `exit` → "continue boot" → Proxmox.
The same thing happened on the 2026-09-20 reboot. **An unattended reboot (power
outage) would have left everything down** (Hermes, the public sites, the blog)
until someone walked over.

### Investigation (read-only)
```
efibootmgr
mokutil --sb-state
findmnt /boot/efi
ls -l /boot/efi/EFI/proxmox /boot/efi/EFI/BOOT
```

| Finding | Meaning |
|---|---|
| `BootCurrent: 000C` (`\EFI\BOOT\BOOTX64.EFI`) | This boot came through the **fallback** path, not the Proxmox entry |
| Two `proxmox` entries (0001, 0002) | Duplicates: one was re-created automatically |
| Secure Boot **disabled** | Not a signature problem |
| `EFI/proxmox` and `EFI/BOOT` hold **identical** files (same sizes, same time) | The boot files are fine |
| `fbx64.efi` present in `EFI/BOOT` | The **fallback** program: when booted through BOOTX64, it reads `BOOTX64.CSV`, **re-creates the proxmox entry and puts it first** |

**Conclusion:** the files were never broken. The **boot order** was. PXE sat ahead
of Proxmox. The order seen afterwards (`0002` first) was the order fallback had just
**repaired**. Lenovo firmware keeps its own boot list in the BIOS and can push
network boot back up, so the lasting fix is in the BIOS.

### Fix, part 1 — from Linux (done)
```
efibootmgr -o 0002,000C,0001,0008,0009,0006,0007
```
Proxmox → fallback copy → duplicate → USB → CD → **PXE IPv4/IPv6 last**.
The leftover factory "Windows Boot Manager" entry is out of the order (no Windows on this box).

### Fix, part 2 — in the BIOS (open)
F1 at power-on → **Startup**: proxmox/NVMe first, **PXE/Network → Excluded**,
**Boot Order Lock** off, save with F10. Then a **watched reboot**: it should go straight to
Proxmox, and `efibootmgr` should show **`BootCurrent: 0002`**.

*"Platform is in Setup Mode"* (from `mokutil`) is normal with Secure Boot off.
No Secure Boot keys are enrolled. That's a possible future hardening topic.

---

## 8. RHEL ISO removed

The RHEL install ISO (~10 GB) was still attached to VM 101 as a virtual CD (`ide2`).
Detached first, then deleted. **Order matters:** delete an ISO a VM still points to,
and that VM **refuses to start** ("volume does not exist") on its next boot.
`pve-root`: **46 → 36 GB used (72% → 56%).**

---

## State at end of session

| VM | State | vCPU / RAM | Disk | Boot |
|---|---|---|---|---|
| 100 docker-agents (OpenClaw) | **Parked** (stopped) | 3 / 6 GB | 32 GB + 16 GB, both kept | `onboot 0` |
| 101 hermes | Running | 4 / **12 GB** | root **52 GB** (47 GB fs, 54%) + 20 GB scsi1 | `onboot 1` |

LibreNMS: docker-agents disabled; 5 active devices.
Host: kernel 7.0.14-28-pve; boot order fixed in software; BIOS step pending.

## Follow-ups

- [ ] **BIOS: PXE excluded, Proxmox first, then a watched reboot** (`BootCurrent: 0002`)
- [ ] Confirm the Windows copy of OpenClaw is a **vzdump** file (restorable by Proxmox)
- [ ] Confirm VM 100 is excluded from the nightly vzdump job (Datacenter → Backup)
- [ ] Hermes: what's growing `/` (~7–8 GB/month); purpose of the 20 GB scsi1 (7% used)
- [ ] Move blog Cloudflare Pages deployment to Hermes with a new scoped token; revoke the old one
- [ ] Monitor the thin pool + non-root mounts (not visible to SNMP today)
- [ ] Carried: alert transport (Telegram) · 2FA on LibreNMS · Beryl as device 7 ·
      Proxmox SSH hardening · t655 "Defunct rules" · Phase 1 (USB Ethernet adapter) ·
      revoke the old honeypot Telegram token
