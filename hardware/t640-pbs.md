# pbs — HP t640 Thin Client (Proxmox Backup Server)

Status: **Built 2026-11-21 (SESSION-61).** Receives nightly backups of Hermes (VM 101) and Wazuh (VM 102) from pve-node1.
Was the cold spare for t640-mon; repurposed because a backup on a second machine is worth more than a spare.

## Role
Backup target only. Not a Proxmox VE node (no cluster, no VMs). Copy 2 of 3 in the 3-2-1 plan:
the VMs on pve-node1 · **this box** · Cloudflare R2 offsite.

## Hardware
| Item | Value |
| --- | --- |
| Model | HP t640 Thin Client |
| Serial | xxxx |
| CPU | AMD Ryzen Embedded R1505G — 2 cores / 4 threads |
| RAM | **8 GB** DDR4 (2 × 4 GB from the spares box) |
| Storage | 128 GB M.2 2280 **SATA** SSD — OS + datastore on one disk |
| Wired NIC | Realtek r8169 → `nic0` (pinned interface names) |
| LAN MAC | xx:xx:xx:xx:xx:xx |
| Power | HP 19.5 V / 2.31 A barrel brick (4.5 mm blue tip). On 24/7 |

## Network
| Item | Value |
| --- | --- |
| Switch port | SG300 **gi9**, VLAN 10 access (description `pbs`) |
| Address | **192.168.10.30/24** static, gw 192.168.10.1 (OPNsense), DNS 1.1.1.1 |
| Hostname | `pbs.lan` |
| GUI | https://192.168.10.30:8007 — reachable from `ADMINS` via WAN rule 5 (`ADMIN_PORTS` = 22, 8006, 8007) |

## Software
| Item | Value |
| --- | --- |
| OS | Proxmox Backup Server **4.2**, ext4, No-Subscription repo (enterprise disabled) |
| Datastore | `homelab` at `/mnt/datastore/homelab`, ~96 GB, **shares the root filesystem** |
| Retention (prune, daily) | keep last 3 · daily 7 · weekly 4 · monthly 3 |
| Garbage collection | daily |
| Verify job | Sun 04:00, skip verified, re-verify after 30 days |
| Backup client | user `pve-backup@pbs`, token `pve-backup@pbs!pve-node1`, role **DatastoreBackup** on `/datastore/homelab` (user + token). Cannot delete or prune. Secret in Apple Passwords |
| On pve-node1 | storage `pbs-homelab` (fingerprint pinned, Keep all backups); job **00:30**, VMs 101 + 102, keep-all |
| SNMP | snmpd on 192.168.10.30:161, v2c read-only, unique community, answers only t640-mon. LibreNMS device |
| Wazuh | agent `pbs` (manager 192.168.10.20) |

## Known limits
- One SSD: no redundancy. R2 is the other copy.
- Datastore on the root filesystem: a full datastore fills the OS. Watched by LibreNMS "Space on / ≥90%".
- Logs to journald (no /var/log/auth.log): SSH logins don't reach Wazuh yet (same open item as Proxmox).

## Next
- [ ] File-level restore test, then a full VM restore once pve-node1 has the 500 GB SSD
- [ ] Notifications (GC / prune / verify failures) → Telegram
- [ ] UPS
