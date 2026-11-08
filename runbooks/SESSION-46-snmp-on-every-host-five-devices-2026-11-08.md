# SESSION 46 — SNMP on Every Host: Five Devices in LibreNMS — 2026-11-08

**Status: COMPLETE.** LibreNMS now monitors five devices across two VLANs and three
operating systems. The Beryl and 2FA are deliberately left for their own sessions.

**Continuity:** continues `SESSION-45` (LibreNMS live, SG300 added). Same night,
past midnight — hence the new date.

| ID | Device | IP | VLAN | OS | Firewall opened |
|---|---|---|---|---|---|
| 1 | sg300-apt | 192.168.8.165 | 1 | Cisco Small Business 1.1.2.0 | — (built-in agent) |
| 2 | pve-node1 | 192.168.10.2 | 10 | Proxmox VE (kernel 7.0.14-5-pve) | none needed |
| 3 | t640-mon | 192.168.8.24 | 1 | Debian 13 (6.12.113) | none (loopback path) |
| 4 | docker-agents (VM 100) | 192.168.10.10 | 10 | Debian 13 | **UFW** |
| 5 | hermes (VM 101) | 192.168.10.206 | 10 | RHEL 10.2 | **firewalld** |

---

## 1. The pattern used on every host

1. Install the agent (`snmpd` / `net-snmp`).
2. **Back up, then replace** the whole `snmpd.conf` — distro defaults ship with a
   `public` community.
3. Listen on **one address** only; answer **one source** only (192.168.8.24);
   read-only; a **unique community string per device** (kept in Apple Passwords and
   LibreNMS's credential store — never in this repo).
4. Test with `snmpget … 1.3.6.1.2.1.1.5.0` (sysName) from t640-mon **before** adding
   to LibreNMS.
5. Add the community to LibreNMS's credential store, then Add Device by IP.

Template:
```
agentaddress udp:<this-host-ip>:161
rocommunity <UNIQUE_STRING> 192.168.8.24
sysLocation "Apartment media shelf"
sysContact Rich
```

---

## 2. pve-node1 (Proxmox)

`apt install snmpd`, template with `192.168.10.2`. Worked first try — no Proxmox
firewall change needed. The poll crosses VLAN 1 → VLAN 10 through the SG300; replies
are permitted by `vlan10-policy`.

---

## 3. t640-mon itself — the "duplicate IP" trap

First tried loopback only (`agentaddress udp:127.0.0.1:161`) and Add Device
`127.0.0.1`. LibreNMS refused:
```
Validation Failed: Cannot add 127.0.0.1, already have device 192.168.10.2 with this IP 127.0.0.1
```
**Why:** discovery records *every* IP on a device — including Proxmox's loopback.
Every Linux host has 127.0.0.1, so LibreNMS saw a duplicate.

**Fix (not "Add Anyway"):** listen on loopback **and** the real address, add by real IP:
```
agentaddress udp:127.0.0.1:161,udp:192.168.8.24:161
rocommunity <STRING> 127.0.0.1
rocommunity <STRING> 192.168.8.24
```
No nftables change: traffic from a host to its own IP travels over the loopback
interface, so `iif lo accept` already covers it, and nothing else may reach UDP 161.

---

## 4. VM 100 docker-agents — UFW was silently dropping SNMP

- Install showed **needrestart** dialogs:
  - **Pending kernel upgrade:** running 6.12.111, installed 6.12.115. Unattended
    upgrades *install* kernels; only a **reboot** loads them. Deferred to a planned window.
  - **Daemons using outdated libraries:** accepted defaults. Docker/containerd were
    **not** listed, so the OpenClaw agents were untouched. Restarting `ssh.service`
    does not drop the current session.
- `ss -ulnp | grep 161` → `UNCONN … 192.168.10.10:161` — **UNCONN is normal for UDP**
  (connectionless); the agent was listening correctly.
- `snmpget` from t640-mon → `Timeout: No Response`.
- Diagnosis, hop by hop: `tcpdump -ni any udp port 161` on the VM, then the local firewall:
  ```
  sudo iptables -S INPUT   → -P INPUT DROP, ufw-* chains, ts-input (Tailscale)
  sudo ufw status          → active; only OpenSSH allowed
  ```
- **Fix — one narrow rule:**
  ```
  sudo ufw allow from 192.168.8.24 to any port 161 proto udp comment 'SNMP from t640-mon'
  ```
- Finding: VM 100 was already default-deny. Good. But `OpenSSH ALLOW Anywhere` →
  follow-up to restrict SSH sources.

---

## 5. VM 101 hermes (RHEL 10.2) — same job, different tools

| Debian | RHEL |
|---|---|
| `apt install snmpd` | `dnf install net-snmp net-snmp-utils` |
| nano preinstalled | **vi** only → `dnf install nano` |
| UFW | **firewalld**, on by default |

- **Typo caught:** `vi /etc/snmp/snmp.conf` (missing `d`) → `[New] 0L, 0B written` =
  vi created an empty, wrongly-named file. `snmp.conf` is the *client* config;
  `snmpd.conf` is the *agent*. Removed the stray file, edited the right one.
- Firewall — a **rich rule** = "this port, only from this source":
  ```
  sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="192.168.8.24" port port="161" protocol="udp" accept'
  sudo firewall-cmd --reload
  ```
- `systemctl enable --now snmpd`; snmpget OK; added.
- If it had failed: `ausearch -m avc -ts recent` checks **SELinux** denials.

---

## 6. Locations, and discovery vs. polling

- Locations didn't match ("Apartment media shelf", "Apt media shelf", blank).
  LibreNMS groups by the exact string.
- SG300 errors along the way:
  - `% Unrecognized command` — typed at `#`, not `(config)#`.
  - `% Wrong number of parameters` — spaces split the value. **Quote it:**
    `snmp-server location "Apartment media shelf"`.
- Waited >5 min, no change. **Correction to my earlier advice:**

  | Job | Default | Collects |
  |---|---|---|
  | Polling | every 5 min | numbers → graphs (traffic, CPU, uptime) |
  | Discovery | every 6 h | what the device *is*: name, **location**, ports, sensors |

  **Rediscover** (device page) updated it immediately.
- "Last polled" isn't on the new overview page; evidence polling works: uptime
  advancing, Device Traffic sparkline populated, gear → Pollers lists last-polled times.

---

## 7. Reading the graphs

- The switch's CPU/memory graphs are **near-flat by design**: forwarding happens in the
  **ASIC**, not the CPU. The switch's real work is in **Ports** (gi1 uplink busiest,
  gi2 Proxmox/VMs, gi4 Mint, gi6 t640-mon).
- Servers do their work on the CPU — Proxmox's CPU graph is where to look there.

---

## 8. What monitoring surfaced on night one

- **pve-node1 uptime ~1 month 1 week** and **VM 100 running an older kernel than
  installed** → security kernels not yet loaded. Planned reboot needed.
- **VM 100 SSH open to "Anywhere"** (UFW).

---

## 9. Decisions

- **Beryl deferred:** it's the internet gateway; OpenWrt's `snmpd` package ships a
  default `public` community answering the whole LAN. Needs a config backup first and
  a fresh head.
- **2FA deferred** (SESSION-45 §6) — next focused session. Accepted risk unchanged:
  UI reachable only from Mint, LibreNMS 26.10.0, non-default admin name.

## Follow-ups

- [ ] **2FA** on the LibreNMS admin
- [ ] **Beryl:** config backup → `opkg install snmpd` → replace default `public`, restrict to 192.168.8.24
- [ ] **Planned reboot window:** confirm VMs `onboot`, reboot pve-node1 (VMs restart on new
      kernels); verify via LibreNMS uptime reset + public sites back
- [ ] Restrict SSH on VM 100 (UFW `OpenSSH ALLOW Anywhere`) — Mint + Tailscale range
- [ ] Proxmox SSH hardening (root password login still allowed — SESSION-42 audit)
- [ ] t655-fw: replace power supply; then `os-net-snmp` plugin + add as device 6
- [ ] Add OPNsense + Proxmox as syslog sources to t640-mon
