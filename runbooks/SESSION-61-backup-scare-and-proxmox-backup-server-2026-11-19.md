# SESSION 61: A second silent backup failure, and a backup server on its own machine
**Dates:** 2026-11-19 → 2026-11-21
**Machines touched:** pve-node1, VM 101 hermes, VM 102 wazuh, spare HP t640 (now `pbs`), SG300, t655-fw, t640-mon (LibreNMS)
**Identifiers:** passwords, tokens, MACs and serials are never written here. Credentials are in Apple Passwords.

**Continuity:** follows `SESSION-60`. SESSION-59 fixed a silent backup failure. A week later it happened again, for
the same reason. This session fixes the cause instead of the symptom: backups now go to a **second machine**.

---

## 1. Small fixes first (11-19)

### 1.1 LibreNMS: loopback false positive
"Port utilisation over threshold" fired on hermes `lo`, because net-snmp reports loopback as a 10 Mbps link.
**Alerts → Alert Rules → Port utilisation over threshold → + Add rule:** `ports.ifType` **not equal** `softwareLoopback` → Save.
Checked with Override SQL (read only, then Cancel): the query now contains `ports.ifType != "softwareLoopback"`.

### 1.2 OPNsense: the duplicate GUI rule
A screenshot showed **both** old GUI rules ("Mint wifi → GUI", "Mint wired → GUI") with source `ADMINS`: identical
duplicates, both enabled. One renamed **`Admins → GUI 443`**, the other disabled (delete after a few days).
Reading the rule list, not trusting memory, found it.

### 1.3 Kali couldn't scan the printer from "any network"
`ip -br a` showed Kali's Wi-Fi on a **100.x** address: the apartment's WhiteSky network, outside my network
entirely. On the Beryl's Wi-Fi (192.168.8.x) the scan worked. **Check where you are before blaming the target.**

---

## 2. Finding: the Nov 19 backup failed silently (again)

```
ls -lh /var/lib/vz/dump/
df -h /var/lib/vz
```
| | |
|---|---|
| Nov 18 archive | **31 GB**, up from 16–18 GB the week before |
| Free on `local` | **25 GB** |
| Nov 19 02:00 | Only a 6.7 KB log, no archive |

**Same mechanism as SESSION-59:** Hermes's root disk filled up (SESSION-60), so the dump grew, and `local` couldn't
hold the old archive plus the new one. Retention keeps the old dump until the new one finishes. **No alert.**

### 2.1 Why deleting files didn't shrink the backup
Hermes had cleaned up its own disk, but on a VM **deleted blocks still look used** to the host until the guest sends
TRIM. vzdump copies every block the host thinks is used.
```bash
# Hermes
sudo fstrim -av
# /boot: 2 GiB trimmed · /srv/storage: 20 GiB trimmed · /: 66.8 GiB trimmed
```
(XFS reports all of its free space each time, so the number is the free total, not what was just freed.)

### 2.2 Proving it
Proxmox → Datacenter → Backup → Hermes job → **Run now**:
- The log said **"92.0 GiB"**: that's the virtual disk size (72 + 20 GB), which vzdump reads in full. Empty blocks
  are skipped as sparse and cost almost nothing.
- **18 GB** archive in about 4 minutes, down from 31 GB. Retention removed the Nov 18 archive **after** the new one succeeded.
- I misread `df` mid-run and thought the old archive had been deleted early. `ls` showed the job had already finished.
  **Read the file list, not just the free-space number.**

Nov 20 02:00: ✅ **19 GB**. It fits, but with about 3 GB to spare. Fixing space twice in a week is a symptom. The
cause: **the only local backup lives on the same small partition, on the same disk, as the VMs.**

---

## 3. The decision: what to do with the spare t640

| Option | Gets | Problem |
|---|---|---|
| **Proxmox Backup Server (PBS) on the spare t640** ✅ | Backups on a **second machine**, incremental, deduplicated | Gives up the cold spare for t640-mon |
| Its SSD into the M720q's 2.5" bay | More room | Same host: dies with it |
| Keep it as a cold spare | Fast swap for t640-mon | Doesn't touch the real risk |

The cold spare isn't worth much: t640-mon can be rebuilt from these runbooks, and PBS can back it up.
PBS is **not a Proxmox node**: it doesn't join a cluster or run VMs. It only receives backups.

Target: **3-2-1**: 3 copies (the VM, PBS, R2), on 2 kinds of storage, 1 offsite.

---

## 4. Building PBS

### 4.1 Hardware and switch port
- Spare HP t640: 128 GB M.2 SATA SSD, **8 GB RAM** (two 4 GB DDR4 sticks from the spares box).
- **PBS 4.2** ISO, checked against the published SHA-256, written with
  `sudo dd if=… of=/dev/sdX bs=1M conv=fdatasync status=progress` after `lsblk` identified the USB by its size.
- SG300 **gi9 → VLAN 10** (next to the host it backs up):
```
configure
interface gi9
description pbs
switchport mode access
switchport access vlan 10
exit
exit
show interfaces switchport gi9
copy running-config startup-config
```
- `ping -c2 192.168.10.30` from Proxmox first: 100% loss = the address is free.

### 4.2 Installer
| Field | Value |
|---|---|
| Disk / filesystem | 128 GB SSD, **ext4** (single disk, 8 GB RAM: ZFS not worth it) |
| Hostname | `pbs.lan` |
| IP / gateway / DNS | **192.168.10.30/24** / 192.168.10.1 / 1.1.1.1 |
| Interface | `nic0` (r8169), **Pin network interface names** ☑ (same scheme as Proxmox) |

Checked from the console: `ip -br a`, then ping the gateway, Proxmox and 1.1.1.1.

### 4.3 Reaching the GUI
Port 8007 wasn't allowed. Instead of a new rule, one alias edit: **ADMIN_PORTS = 22, 8006, 8007**. WAN rule 5
(`ADMINS → SERVERS_NET : ADMIN_PORTS`) now covers it. https://192.168.10.30:8007 → login ✅.

### 4.4 Updates
Administration → Repositories: enterprise **disabled**, **No-Subscription** added. `apt update && apt full-upgrade -y`,
reboot for the new kernel. (`pvesm: command not found` on PBS: that's a Proxmox VE command. Check the prompt.)

### 4.5 Datastore
`df -h /` → about 87 GB available. **Datastore → Add:** `homelab` at `/mnt/datastore/homelab`, GC daily, prune daily,
**keep last 3 · daily 7 · weekly 4 · monthly 3**. Prune decides what to forget; garbage collection frees the space.
The datastore shares the root filesystem, so a full datastore = a full OS. LibreNMS's "Space on / ≥90%" alert covers it (§5).

### 4.6 A token that can't delete backups
If pve-node1 were compromised, a root login to PBS could delete every backup. So Proxmox uses a limited token:

| | |
|---|---|
| User | `pve-backup@pbs` |
| Token | `pve-backup@pbs!pve-node1` (secret shown once, saved immediately) |
| Permission | **DatastoreBackup** on `/datastore/homelab`, for the **user and the token** (a token gets the overlap of both) |

The permission dialog defaults to **NoAccess**: it has to be changed by hand. `DatastoreAdmin` / `PowerUser` would allow deleting.

### 4.7 Proxmox side
**Datacenter → Storage → Add → Proxmox Backup Server:** ID `pbs-homelab`, server .10.30, username = token ID,
password = token secret, datastore `homelab`, **fingerprint** from the PBS dashboard, retention **Keep all backups**
(PBS prunes; the token isn't allowed to anyway).
```
pvesm status
pbs-homelab   pbs   active   100334656   4319132   90872588   4.30%
```

### 4.8 Don't break the offsite copy
Before changing the backup job, I checked what feeds R2:
```
crontab -l      # 30 3 * * * /root/backup-to-r2.sh
```
The R2 upload runs at 03:30 from the local dumps. Moving the job to PBS would have **silently stopped offsite
backups**. So the old job stays and a **new** one is added:

| Time | Job | Copy |
|---|---|---|
| **00:30** | vzdump → **PBS**, VMs **101 + 102**, keep-all | Second machine, deduplicated history |
| 02:00 | vzdump → `local`, VM 101, keep-last 1 | Feeds R2 |
| 03:30 | `backup-to-r2.sh` | Offsite |

**Wazuh is backed up for the first time:** VM 102 → Hardware → disk → Advanced → **Backup ☑**. It had been off
because `local` couldn't hold it.

### 4.9 First run
Run now → one job, **each VM is its own backup** (`vm/101`, `vm/102`), visible from both ends: Proxmox's task log
and PBS's task list, where the user is the **token**, not root.
- read = write (~170 MiB/s): real data, every chunk new. read ≫ write: empty or already-seen chunks skipped (the TRIM paying off).
- "write" can exceed gigabit (~118 MB/s) because it counts data **before** compression.
- `TASK OK` for both.

**Verify job:** weekly, Sun 04:00, skip verified, re-verify after 30 days. It re-reads chunks against their checksums,
so corruption shows up before a restore is needed.

---

## 5. PBS is monitored like everything else
- **SNMP:** `snmpd` bound to `192.168.10.30:161`, unique read-only community, answers only `192.168.8.24` (t640-mon).
  Firewall already allowed it (WAN rule 7: MONITOR → SERVERS_NET UDP 161). `snmpget … sysName.0` → `pbs`.
  **LibreNMS: 8 devices.**
- **Wazuh agent `pbs`:** same VLAN as Wazuh, no rule needed. Proxmox lessons applied: root (no `sudo`),
  `apt install lsb-release` first, `<address>` checked before start. **Agents: hermes, pve-node1, t640-mon, pbs.**

---

## 6. State at end of session
| | |
|---|---|
| Backups | **PBS nightly (Hermes + Wazuh)** + local nightly (Hermes) + R2 offsite: **3-2-1** |
| PBS | `pbs`, 192.168.10.30, gi9 / VLAN 10, PBS 4.2, 8 GB RAM, datastore ~96 GB |
| ADMIN_PORTS | 22, 8006, **8007** |
| LibreNMS | 8 devices · loopback excluded from port utilisation |
| Wazuh | 4 agents |

## 7. Lessons learned
- **On a VM, deleting files doesn't shrink the backup until TRIM runs.** `fstrim -av` in the guest.
- **A backup of "92 GiB" can write 18 GB.** vzdump reads the whole virtual disk and skips empty blocks.
- **Read the file list, not just `df`,** before concluding something was deleted.
- **Fixing the same failure twice means fixing the wrong thing.** Space was the symptom; one disk was the cause.
- **A backup on the same disk isn't a backup.** A second machine is.
- **Check what depends on a job before changing it.** The R2 upload read the local dumps.
- **Give backup clients a token that can't delete.** If the host is compromised, the history survives.
- **A permission dialog's default (NoAccess) is not a choice.** Change it on purpose.
- **Check which machine your prompt is on.** `pvesm` exists on Proxmox VE, not PBS.

## 8. Next
1. **Backup-failure alerts to Telegram** (Proxmox webhook + matcher on `vzdump`, then the same on PBS). The 02:00 job still feeds R2 with a thin margin.
2. **Restore test:** file-level now (vm/101 → File Restore → etc/hostname). Full VM restore after the 500 GB SSD is installed, to a new VM ID with the network disconnected.
3. Wazuh: journald on Proxmox and PBS so SSH logins show up.
4. 500 GB SSD → second VM storage on pve-node1; move Wazuh's disk there.
5. A UPS for the six core devices.
6. Decide on encrypting R2 backups.
