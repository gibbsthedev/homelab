# SESSION 59: Suricata on the servers, OpenClaw retired, a silent backup failure, Wazuh live
**Date:** 2026-11-15 (about 11:00 → 18:45 CST)
**Machines touched:** t655-fw, t640-mon, pve-node1, VM 101 hermes, new VM 102 wazuh, Mint
**Identifiers:** passwords, tokens, MACs and serials are never written here. Credentials are in Apple Passwords.

**Continuity:** follows `SESSION-58`. Starts **Phase 4 (Wazuh SIEM)**. Decisions this session: SSH confirmed working;
LibreNMS 2FA waits until the new admin laptop ("command") takes over.

---

## 1. Suricata now watches the servers

### 1.1 EVE check deferred, with a rule
The phone watcher (`ids-telegram.service`) only matches the **old** syslog format (`Priority: [12]`). EVE output sends
JSON instead. I deferred updating the watcher. **Rule until it's done: keep "Enable syslog alerts" ON.** Turning it off,
or Suricata 9.0 removing it, would silently stop phone alerts.

### 1.2 SERVERS added
**Services → Intrusion Detection → Administration → Settings → Interfaces: LAB + SERVERS** → Apply.
Still IDS / PCAP live mode (alerts only, never blocks).

**What Suricata can and can't see on SERVERS:** the public sites arrive through **Cloudflare's encrypted tunnel**, so
attacks *against* the websites are invisible to it. What it can catch is a server that's **already compromised**:
outbound malware traffic, botnet command servers, crypto-mining, known-bad TLS fingerprints. Rulesets offered for that:
ET open `emerging-malware`, `emerging-coinminer`, abuse.ch Feodo Tracker, abuse.ch SSL Fingerprint Blacklist.

### 1.3 Proof: from Hermes
```bash
# Hermes
curl -d 'uid=0(root) gid=0(root)' http://example.com
```
Phone, within seconds:
```
IDS alert: [1:2100498:7] GPL ATTACK_RESPONSE id check returned root
  [Classification: Potentially Bad Traffic] [Priority: 2] {TCP} 192.168.10.206:40202 -> 104.20.23.154:80
```
Same signature as the lab test in SESSION-53, now from the internet-facing box: **detection → central log → phone**.

### 1.4 Reading alerts like an analyst
The same Telegram chat also received **LibreNMS** messages ("Device 192.168.8.195 recovered from Device rebooted",
then the same for 192.168.8.24). Different source, different meaning:

| Message starts with | From | Based on |
|---|---|---|
| "Device … recovered from Device rebooted" | LibreNMS | SNMP: uptime reset, device back |
| "IDS alert: …" | Suricata → firewall log → watcher | Traffic content |

Both reboot alerts matched my own maintenance (the t655 RAM swap at ~08:19, the t640 snmpd test reboot at 07:57).
Each one was matched to a known cause and closed. The t640 one also proves the snmpd retry fix from SESSION-58:
LibreNMS could only see the reboot because snmpd came back by itself.

---

## 2. Storage: why Wazuh didn't fit

Wazuh 4.x all-in-one, sized for 1–25 agents: **4 vCPU, 8 GiB RAM, 50 GB** (Wazuh's quickstart table), and it grows
with log retention. RAM was fine (~28 GB free). Disk wasn't:

```bash
pvesm status
lvs
```
| Storage | Size | Used | Free |
|---|---|---|---|
| local-lvm (thin pool, VM disks) | 141 GB | **70.5%** | ~41 GB |
| local (backups, ISOs) | 69 GB | 74% | ~14 GB |

| Volume | Size | Data% | What |
|---|---|---|---|
| vm-101-disk-0 / disk-1 | 52 / 20 GB | 62 / 27 | Hermes |
| vm-100-disk-0 / disk-1 | 32 / 16 GB | 86 / **99** | OpenClaw |
| **vm-100-state-hibernation** | ~10.5 GB | full | RAM saved with OpenClaw's snapshot |
| snap_vm-100-…_hibernation | 32 + 16 GB | grows with every change | OpenClaw's old snapshot |

**A thin pool that fills up freezes every VM on it** (and can corrupt their disks). Keep at least ~20% free. Adding
Wazuh to 41 GB free would have pushed the pool past 90%.

Options: A) retire OpenClaw, B) delete only the snapshot (~10–20 GB, too tight), C) add a disk (the empty SSD2 M.2
slot or the 2.5" bay). **Chosen: A now, C later.**

---

## 3. OpenClaw retired

**Decision:** Hermes is the better product and had already taken over everything important. No final backup: if
OpenClaw is ever needed again, it gets rebuilt from scratch. Hermes handles the shared secrets.

```bash
qm list                                              # 100 = docker-agents, 101 = hermes (check before every step)
qm shutdown 100 ; qm list                            # 100 stopped, 101 still running
qm listsnapshot 100                                  # "hibernation"
qm delsnapshot 100 hibernation                       # removed vm-100-state-hibernation + both snap volumes
qm destroy 100 --purge --destroy-unreferenced-disks 1   # removed vm-100-disk-0 + disk-1, purged from configs
```
| | Before | After |
|---|---|---|
| local-lvm | 70.5% used, ~41 GB free | **26.8% used, ~103 GB free** |
| VMs | OpenClaw + Hermes | Hermes |
| Free RAM | ~28 GB | ~40 GB |

Also: LibreNMS device deleted. **Deleting a VM does not delete its backups**, so the old OpenClaw dumps on `local`
were removed by hand (`rm -i /var/lib/vz/dump/vzdump-qemu-100*`, confirming each file): **local 74% → 59%**.
Old OpenClaw copies elsewhere (the Windows PC, the one Hermes made) and its Tailscale machine entry are on the cleanup list.

---

## 4. Finding: Hermes's backup had failed two nights in a row

The same `ls -lh /var/lib/vz/dump/` showed:

| Night | Archive (`.vma.zst`) | Log |
|---|---|---|
| Nov 12 | ✅ 16 GB | ✅ |
| Nov 13 | ✅ 18 GB | ✅ |
| **Nov 14** | ❌ missing | 6.9 KB |
| **Nov 15** | ❌ missing | 6.6 KB |

The log ended in **`write error: broken pipe`**. The backup pipes the VM image into a compressor that writes the file.
When `local` ran out of space (**~14 GB free vs an ~18 GB dump**), the compressor's write failed and quit, leaving the
backup writing into a closed pipe. Retention keeps old dumps until the new one finishes, so the job needed room for
**one more dump than it keeps**. Nothing alerted on it.

**Fix (this session):**
1. Deleted the OpenClaw dumps (above): ~25 GB free on `local`.
2. Datacenter → Backup → job → Retention → **Keep Last: 1**. The off-site copy in R2 is the second copy.
3. Deleted the oldest Hermes archive (Nov 12) with `rm -i`.

**Still open:** confirm tonight's 02:00 run produced a `.vma.zst`, alert on backup failure (Proxmox notifications →
the same Telegram bot), and a separate backup disk long-term (same drive = not a real backup if the drive dies).

---

## 5. Wazuh (Phase 4)

### 5.1 Version and OS (checked, not assumed)
- **Wazuh 4.14** is the stable line. 5.0 is still a release candidate from a staging repo, so it isn't used for the security system.
- Wazuh 4.14's central components support **Ubuntu up to 24.04** (26.04 only appears for 5.0). → **Ubuntu Server 24.04.5 LTS**.

### 5.2 ISO
Proxmox → local → ISO Images → **Download from URL** with **SHA-256 checksum** from Ubuntu's `SHA256SUMS` → `TASK OK`.
Gotcha: the first link copied was `ubuntu-24.04.4-desktop-amd64.iso.torrent` (desktop, older, a 496 KB torrent file).
The right one: **`ubuntu-24.04.5-live-server-amd64.iso`** (3.8 GB). Each file has its own checksum.

### 5.3 VM 102 `wazuh`
| Setting | Value | Why |
|---|---|---|
| VM ID | **102** | Not reusing 100, so old logs never get confused |
| Machine / BIOS | q35 / SeaBIOS | |
| Disk | 60 GB, local-lvm, Discard ✅, IO thread, SSD emulation | Discard = TRIM can free pool space |
| **Backup** | ☐ off | A 60 GB dump would break Hermes's backups on `local`. Wazuh can be rebuilt |
| CPU / RAM | 4 cores (host) / 8 GB | Wazuh's sizing for 1–25 agents |
| Network | vmbr0, no VLAN tag | Same as Hermes: the switch port puts it on VLAN 10 |
| Start at boot | ✅ | |

Pre-check: `ping -c2 192.168.10.20` from Proxmox → 100% loss (address free).

### 5.4 Ubuntu install: two traps
1. **Network skipped by accident.** VLAN 10 has no DHCP, and the installer couldn't go back from the storage screen.
   Fix: Stop/Start the VM and start over (nothing had been written yet). Static: `192.168.10.20/24`, gw `192.168.10.1`, DNS `1.1.1.1, 8.8.8.8`.
2. **The installer's LVM uses only half the disk** (`ubuntu-lv` 28.996G + 29G free). Edit `ubuntu-lv` → size = max (57.996G).

Also: the noVNC console cut off the bottom buttons. Fix: console side tab → Settings → **Scaling mode: Local scaling**.
No snaps installed (only what I choose goes on the SIEM box). OpenSSH server installed.

Verified after boot:
```
ip -4 addr show enp6s18     # 192.168.10.20/24   (q35 names the NIC enp6s18, not ens18)
df -h /                     # 57G, 48G free
```

### 5.5 Prep
```bash
sudo apt update && sudo apt full-upgrade -y
sudo apt install -y qemu-guest-agent && sudo systemctl start qemu-guest-agent   # Proxmox shows the IP, clean shutdowns
sudo timedatectl set-timezone America/Chicago ; timedatectl                    # "System clock synchronized: yes"
sudo reboot
ssh <user>@192.168.10.20      # from Mint: already allowed by WAN rule 5 (ADMINS → SERVERS_NET 22)
```

### 5.6 Install (Wazuh's official quickstart command)
```bash
curl -sO https://packages.wazuh.com/4.14/wazuh-install.sh && sudo bash ./wazuh-install.sh -a
```
`-a` = all-in-one (indexer + server + dashboard). Ended with `INFO: Installation finished.` at 18:00.

```bash
sudo mv ~/wazuh-install-files.tar /root/ && sudo chmod 600 /root/wazuh-install-files.tar   # holds EVERY Wazuh password
sudo sed -i "s/^deb /#deb /" /etc/apt/sources.list.d/wazuh.list                           # no accidental upgrades
```
(First `sed` attempt had an unfinished quote → `>` prompts → Ctrl+C.)

**Finding: the dashboard admin password was captured in a photo of the terminal.** Treated as burned. Rotation:
`wazuh-passwords-tool.sh -u admin -p '…'`, then update Filebeat's keystore (it logs in as admin in an all-in-one),
`filebeat test output` → OK, clear shell history, delete the photo. **Status: to confirm.**

### 5.7 Dashboard through the firewall
The first browser attempt failed. **Expected:** 443 wasn't allowed. New WAN rule:

| Action | Proto | Source | Destination | Port | Description |
|---|---|---|---|---|---|
| Pass | TCP | ADMINS | 192.168.10.20 | 443 | Admins → Wazuh dashboard |

https://192.168.10.20 → self-signed certificate warning (expected) → dashboard ✅. 123 medium and 190 low alerts
already showing: the Wazuh server watching **itself** (agent 000: package installs, sudo, its own config checks).

### 5.8 Agents
| Agent | OS | Status | Notes |
|---|---|---|---|
| **hermes** | RHEL 10 | ✅ active | Dashboard wizard → RPM amd64, manager 192.168.10.20. Same VLAN, no rule needed |
| **pve-node1** | Proxmox 9 (Debian) | ✅ active, after 3 fixes ⬇ | |
| t640-mon | Debian 13 | next | VLAN 1 → needs a firewall rule |

**Proxmox, three lessons in one:**
1. **No `sudo` on Proxmox** (you're already root) → drop `sudo` from the wizard's commands.
2. **No `lsb-release`** → `dpkg: dependency problems … leaving unconfigured`. `dpkg -i` doesn't fetch dependencies. Fix: `apt install -y lsb-release`, then rerun the install with the variables.
3. **The agent wouldn't start**: the server address is written during the setup step, which had failed, so `ossec.conf` still said `MANAGER_IP`.
```bash
grep -A1 "<address>" /var/ossec/etc/ossec.conf
sed -i 's|<address>MANAGER_IP</address>|<address>192.168.10.20</address>|' /var/ossec/etc/ossec.conf
systemctl restart wazuh-agent ; systemctl status wazuh-agent --no-pager     # active (running)
```

---

## 6. State at end of session
| | |
|---|---|
| pve-node1 | Hermes (VM 101, 20 GB) + **Wazuh (VM 102, 8 GB, 60 GB disk)**. local-lvm ~27% + Wazuh's growth; local ~59% |
| Wazuh | 4.14.x all-in-one at https://192.168.10.20, agents: hermes, pve-node1 |
| Suricata | IDS on **LAB + SERVERS**, old syslog output kept ON for the phone watcher |
| Backups | Hermes only, **Keep Last 1** + R2; Wazuh excluded on purpose |
| OpenClaw | **Retired**: VM, snapshot, disks, local dumps, LibreNMS device gone |

## 7. Lessons learned
- **Check what a ruleset can see.** Encrypted tunnel traffic hides web attacks; watch for a compromised server's behavior instead.
- **Match every alert to a cause.** Your own maintenance explains most of them; the ones that don't fit are the ones to dig into.
- **A thin pool that fills freezes every VM.** Check `lvs` Data% before adding a big disk.
- **Deleting a VM leaves its backups behind.** Old dumps keep the secrets and the space.
- **A backup job that fails silently isn't a backup.** List the archives, not just the logs. Retention needs room for one more dump than it keeps.
- **`broken pipe` after a write error** usually means the disk filled up mid-write.
- **Read the file name before downloading.** desktop vs server, .4 vs .5, .iso vs .torrent. A checksum proves you got the right file.
- **Ubuntu's LVM default uses half the disk.** Resize `ubuntu-lv` in the installer.
- **Never photograph a password.** If it happens, rotate it.
- **Generated install commands assume a full distro.** Proxmox has no `sudo` or `lsb-release`.
- **A failed package setup can leave placeholders in the config.** Check `ossec.conf` before blaming the network.

## 8. Next
1. **Confirm the Wazuh admin password was rotated** (and Filebeat test OK).
2. t640-mon agent: alias `WAZUH_AGENT_PORTS` (1514, 1515) + WAN rule MONITOR → 192.168.10.20; `apt install lsb-release` first.
3. Confirm tonight's Hermes backup produced a `.vma.zst`. Then backup-failure alerts → Telegram.
4. Wazuh: test that an SSH login on Proxmox shows up (journald vs `/var/log/auth.log`).
5. Wazuh: OPNsense + Suricata logs in; Windows and "command" agents later.
6. OpenClaw cleanup: Windows PC copy, Hermes's copy, Tailscale machine entry.
7. Suricata: confirm which extra rulesets were enabled on SERVERS; EVE watcher update.
8. Long-term: a second disk for backups (and Wazuh's growth).
