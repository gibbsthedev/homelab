# SESSION 60: A deliberate foothold, a new admin machine, the t640 Wazuh agent, and Hermes's disk full
**Dates:** 2026-11-16 → 2026-11-19 (several evenings and one day off)
**Machines touched:** HP OfficeJet Pro 8025e (new), "command" (new admin laptop), t655-fw, t640-mon, SG300, pve-node1, VM 101 hermes, VM 102 wazuh
**Identifiers:** passwords, tokens, MACs and serials are never written here. Credentials are in Apple Passwords.

**Continuity:** follows `SESSION-59` (Wazuh live with two agents). This session: a networked printer joins the home
network on purpose as an attack target, "command" becomes an admin machine before its SSD arrives, t640-mon becomes the
third Wazuh agent, and Hermes's root disk fills up and gets grown live.

---

## 1. Design question: "is the router behind the firewall?"
**No, by design.** The Beryl is the **edge**: it faces the internet, does NAT, and has the wired WAN with automatic
WhiteSky Wi-Fi failover (SESSION-39). What isn't behind OPNsense is the **home network (VLAN 1)**. The servers (VLAN 10)
and the lab (VLAN 20) are.

| Option | What changes | Cost |
|---|---|---|
| A) Management VLAN | Admin machines + device GUIs move to their own VLAN behind OPNsense | Least disruption. **Recommended first** |
| B) Home VLAN behind OPNsense | All home devices behind the firewall, Beryl only as uplink | Beryl Wi-Fi clients are the hard part |
| C) OPNsense as the true edge | Beryl becomes an access point only | Loses the tested Wi-Fi failover unless replaced |

Decision: **none built yet**. Decide after Wazuh is finished.

---

## 2. HP OfficeJet Pro 8025e: a deliberate foothold on VLAN 1

### 2.1 Why it stays on the home network
The Canon MG3620 was taken **off** the network in SESSION-55 (USB + CUPS on t640-mon). The HP is the opposite
experiment: **networked and left as it is, on purpose**, as a practice target.

**Decision: it stays on VLAN 1, not its own VLAN.** A foothold is only realistic where a real one would be. It answers
three questions:

| Question | Expected answer |
|---|---|
| Can it reach the servers? | **No.** It isn't in `ADMINS` or `MONITOR`, so OPNsense drops it |
| What can it reach on VLAN 1? | **Everything:** the admin laptops, Windows, t640-mon, the Beryl and switch GUIs. That's the case for the management VLAN |
| What can see it? | VLAN 1 traffic never crosses OPNsense, so **Suricata is blind there**. Only LibreNMS (SNMP) and endpoint agents see it |

Guardrails: no port forwards to it, UPnP off on the Beryl, nothing sensitive printed during the experiment.

| | |
|---|---|
| Switch port | **gi7** (description `hp8025e-foothold`) |
| Address | **192.168.8.233** |
| In LibreNMS | Yes, SNMP v2c |

### 2.2 Baseline scan
From testkit (Kali). **From the lab it showed nothing**: Phase 1 blocks LAB → home, so the first scan proved the
firewall, not the printer. For the baseline, testkit was put on VLAN 1 on purpose:
```bash
sudo nmap -sV -p- 192.168.8.233 -oN hp8025e-baseline-tcp.txt
sudo nmap -sU --top-ports 50 192.168.8.233 -oN hp8025e-baseline-udp.txt
```
| Port | Service | Note |
|---|---|---|
| 80, 443, 8080 | nginx (embedded web server) | Admin page |
| 631 | IPP | Network printing |
| 9100 | JetDirect (raw printing) | **No authentication**: anything sent is printed or interpreted |
| 3910, 3911, 53048 | HP SOAP (gSOAP 2.7) | An old library |
| 5355 | LLMNR | Name resolution by broadcast; a classic credential-capture target |

Scans saved off-repo as the "before" picture.

### 2.3 Finding #1: SNMP can only be read-only with `public`
The printer's web page offers **Read-Only only with the default community `public`**. A custom community name exists
only for **Read-Write**. Read-Write was rejected: it allows remote reconfiguration over a plaintext community.
**Kept: read-only `public`, documented as a deliberate, known weakness.**

### 2.4 First alert from it
**2026-11-17 01:30:11** · *State Sensor Critical* · `Printer Device Status = Down` (`hrDeviceStatus`). Recovered at
01:35:11. **Normal printer behavior** (deep sleep / auto-off overnight), not a fault. Kept on during the experiment: it
logs sleep/wake and would show a crash mid-attack. It's already a comparison point: LibreNMS sees the HP's state, while
the Canon is invisible to monitoring.

---

## 3. "command" becomes an admin machine (before its SSD)

### 3.1 Why now
The old Mint laptop (4 GB DDR3) runs out of RAM with the Wazuh dashboard open (§5). "command" (HP, 32 GB, memtest
passed in SESSION-58) was ready, so it became an admin machine now instead of waiting for the SSD.

| | |
|---|---|
| Wired | **192.168.8.118**, switch port **gi8** (description `command`) |
| Wi-Fi | 192.168.8.213 (reserved on the Beryl) |

### 3.2 One alias, then three more locks
Adding .118 to the OPNsense alias **`ADMINS`** worked immediately for everything built on it: the Wazuh dashboard, SSH,
the Proxmox GUI, ping. But command still couldn't open OPNsense or LibreNMS. There were **three separate allow-lists**,
each written before command existed:

| # | Where | Was | Fix |
|---|---|---|---|
| 1 | **OPNsense GUI** (WAN rules 1–2) | Two single-host rules: "Mint wifi → GUI", "Mint wired → GUI" | One rule's source changed to **`ADMINS`**, renamed `Admins → GUI 443`. The other disabled (2026-11-19), to delete after a few days |
| 2 | **t640 nftables** | `ip saddr { .186, .142 } tcp dport { 22, 80 }` | Now `{ .186, .142, .118, .213 }` |
| 3 | **nginx (LibreNMS)** | `allow .186; allow .142; deny all;` | `allow 192.168.8.118;` added above `deny all;` |

nginx, the careful way:
```bash
sudo cp /etc/nginx/sites-enabled/librenms.conf /root/librenms.conf.bak   # backup OUTSIDE sites-enabled
sudo nano /etc/nginx/sites-enabled/librenms.conf
sudo nginx -t          # "syntax is ok" / "test is successful"
sudo systemctl reload nginx
```
A backup left inside `sites-enabled/` would be loaded as a second site.

**How to read the errors:**

| What you see | What it means |
|---|---|
| Timeout | The firewall dropped it (OPNsense or nftables) |
| **403 Forbidden** | The network path is fine. **nginx's own list** said no |
| Page loads | All three layers agree |

### 3.3 Naming .213
The first notes said .213 was the iPhone (SESSION-48 had identified it that way). On 2026-11-19 `ip -br a` on command
showed **.118 wired and .213 Wi-Fi**: .213 is command now. A DHCP address belongs to a device **only at a point in
time**, which is why it stays in the t640 rule and is now **reserved to command** on the Beryl.

### 3.4 SSH to t640 from command
`Permission denied (publickey)`: t640 is key-only and only knows Mint's key. Mint also uses a `~/.ssh/config` shortcut.
command already has its own `~/.ssh/id_ed25519` (**don't overwrite it**). Decision: keep doing t640 work from Mint for
now; add command's public key later.

---

## 4. t640-mon: the third Wazuh agent
t640 is on VLAN 1, Wazuh is on VLAN 10, so the firewall needed a rule first.

| Alias | Content |
|---|---|
| `WAZUH_AGENT_PORTS` | 1514 (events), 1515 (enrollment) |

| Action | Proto | Source | Destination | Port | Description |
|---|---|---|---|---|---|
| Pass | TCP | MONITOR | 192.168.10.20 | WAZUH_AGENT_PORTS | t640 → Wazuh agent |

```bash
nc -zv 192.168.10.20 1515      # "open": the path works before installing anything
sudo apt install -y lsb-release   # the lesson from Proxmox in SESSION-59
```
Installed agent 4.14.10. Then:
```
ERROR (1208): Unable to connect to enrollment service at '[192.168.8.24]:1515'
```
**The agent was asking itself.** `WAZUH_MANAGER` had been given t640's own IP, so `ossec.conf` said
`<address>192.168.8.24</address>`.
```bash
grep -A1 "<address>" /var/ossec/etc/ossec.conf
sudo sed -i 's|<address>192.168.8.24</address>|<address>192.168.10.20</address>|' /var/ossec/etc/ossec.conf
sudo systemctl restart wazuh-agent      # the agent only reads its config at start
```
✅ t640-mon shows as the **third active agent** (hermes, pve-node1, t640-mon).

---

## 5. "The Wazuh dashboard isn't loading" (it was the laptop)
On Mint, the dashboard hung. Triage by layer before blaming Wazuh:

| Symptom | Layer |
|---|---|
| Timeout | Network / firewall |
| Connection refused | Service down |
| "Wazuh dashboard server is not ready yet" | Indexer still starting |
| Loads on another machine, hangs here | **The client** |

It loaded on command. Mint's RAM was nearly exhausted. This is what moved command forward (§3).

---

## 6. Hermes's root disk hit 100%, grown live
```
df -h /     →   /dev/mapper/rhel-root   47G   47G   268K   100%
```
Layout: `sda` → `sda3` (LVM physical volume) → `rhel-root` (XFS) + `rhel-swap`. Separate `sdb` (20 GB) at `/srv/storage`.

```bash
qm resize 101 scsi0 +20G                      # Proxmox: virtual disk 52 → 72 GB, VM still running
sudo growpart /dev/sda 3                      # FAILED: "No space left on device"
sudo TMPDIR=/dev/shm growpart /dev/sda 3      # "NOCHANGE: partition 3 is size 146796511"
sudo partx -u /dev/sda                        # kernel re-reads the partition table: sda3 ~70 GB
sudo pvresize /dev/sda3
sudo lvextend -r -l +100%FREE /dev/rhel/root  # -r grows XFS online
```
Two traps:
1. **growpart writes a temp file to `/tmp`**, which was on the full disk. `TMPDIR=/dev/shm` points it at RAM.
2. **"NOCHANGE" didn't mean nothing happened.** The first, "failed" attempt had already rewritten the partition table.
   The kernel just hadn't noticed (the disk is in use). `partx -u` made it look again.

No restart. Hermes then cleaned up its own disk; verified healthy.

**The real question:** root was grown twice in a week (32 → 52 GB on 11-11, disk → 72 GB on 11-17). Growing is a
bandage; next is finding what grows: `du -xh / --max-depth=2 | sort -rh | head`.

---

## 7. Reading this week's alerts
| Alert | Cause | Verdict |
|---|---|---|
| **Port utilisation over threshold**, hermes `lo` (11-12, 4m59s) | net-snmp reports loopback as a **10 Mbps** link. Local traffic can top that for one poll | **False positive.** Fix: exclude `ports.ifType = softwareLoopback` in the rule (next) |
| **State Sensor Critical**, HP printer (11-17, 5 min) | Printer sleep | Expected. Kept on during the experiment |
| Telegram recovery messages showing `=&gt;` | HTML-escaped text sent as plain text | Cosmetic. Looked fixed by 11-16; confirm transport format = HTML |

Better to tune a rule to exclude an interface **type** it should never match than to silence hosts one by one.

---

## 8. Confirmed
- Hermes's nightly backup on 11-16 produced a `.vma.zst`: the **Keep Last 1** fix from SESSION-59 works.
- t640-mon agent active in Wazuh.
- nginx `allow .118`; command reaches LibreNMS.
- OPNsense GUI rule on `ADMINS`, renamed; old single-host Mint rule disabled.
- Printer in LibreNMS; SG300 gi7/gi8 descriptions saved (`copy running-config startup-config`).
- Printer baseline scans saved; printer admin password changed from the default.
- **Wazuh admin password rotated** (it had been photographed in SESSION-59).
- command's Wi-Fi address .213 reserved on the Beryl.
- 2026-11-19: cables tidied with Velcro, no ports moved.

---

## 9. State at end of session
| | |
|---|---|
| VLAN 1 | + HP printer (.233, gi7, deliberate foothold) · + command (.118 wired on gi8, .213 Wi-Fi) |
| `ADMINS` | .142, .186 (old Mint), .118 (command), .166 (Windows) |
| Wazuh | 3 agents: hermes, pve-node1, **t640-mon** |
| LibreNMS | 7 devices, incl. the printer |
| Hermes | root disk 72 GB, healthy after cleanup |

## 10. Lessons learned
- **A foothold is realistic only where a real one would be.** Then measure what it can reach and what can see it.
- **Some devices can't do private read-only SNMP.** Document the risk instead of turning on write access.
- **An alias is one lock.** Device-level allow-lists (nftables, nginx) are separate locks and need the new address too.
- **403 from a web server means the network is fine.** The web server's own list refused you.
- **Check `<address>` in `ossec.conf` before blaming the network.** Restart the agent after changing it.
- **"The dashboard is broken" can be the laptop.** Try a second client before debugging the server.
- **growpart needs `/tmp`.** On a full root, `TMPDIR=/dev/shm`. "NOCHANGE" can mean an earlier try worked; `partx -u` makes the kernel see it.
- **A DHCP address names a device only at a point in time.** Re-check before writing it into a rule; reserve it after.

## 11. Next
1. When command takes over fully: drop the old Mint (.142/.186) from `ADMINS`, nftables and nginx; delete the disabled GUI rule.
2. LibreNMS: loopback exclusion on the port-utilisation rule; confirm Telegram format = HTML.
3. Find what keeps filling Hermes's root.
4. Wazuh: SSH-login test on Proxmox; OPNsense + Suricata logs; Windows and command agents.
5. Backup-failure alerts → Telegram; Hermes `/srv/storage` freeze error.
6. Printer experiment: make it reachable from the lab with a narrow, temporary rule, attack it, then harden one control at a time.
7. Decide the management VLAN design (§1).
