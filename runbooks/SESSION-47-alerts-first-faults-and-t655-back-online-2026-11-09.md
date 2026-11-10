# SESSION 47 — First Alerts, First Faults, and t655-fw Back Online — 2026-11-08 → 2026-11-09

**Status: COMPLETE for this phase.** LibreNMS produced its first real alerts and
caught three real problems on day one. The t655 got a new power cord, its stalled
update finally landed, a security gap was found and closed, and it became
monitored device 6.

**Continuity:** follows `SESSION-46` (five devices). Two sittings: Sunday afternoon
(alerts, disks) and Monday morning (t655, done hands-on, one step at a time).

---

## Part A — First morning with monitoring (2026-11-08)

### A1. VM 100 rebooted — planned, and monitoring confirmed it
- Header showed `Linux 6.12.115+deb13.1-amd64`. At 02:40 needrestart had said
  *running 6.12.111*. A kernel only loads on reboot → the VM had rebooted.
- **Confirmed planned** (Rich rebooted it).
- Graphs: CPU boot spike (~65%) then near-idle; memory **flat ~3 GB, no sawtooth**
  (no leak overnight); ~0.5–0.8 GB swap in use (watch: steady = fine, growing = needs RAM);
  root disk ~70%.

### A2. First alerts — mostly noise, one real finding
Rule "Port status" fired for: gi3, gi7–gi10, the VLAN 20 SVI ("Port 20"), docker0
on VM 100 — all **expected down** (nothing plugged in / VLAN has no up members /
Docker's default bridge unused) — and **gi6**.

**gi6 was the real finding:** docs said gi6 = t640-mon, but t640-mon was up and
gi5 wasn't alerting. **t640-mon was actually on gi5.** Cables swapped: t640-mon → gi6,
gi5 kept for the firewall. Monitoring caught documentation drift without anyone looking.

Noise handling: expected-down ports set to **Ignore** (alerting off, polling on) —
not Disable. Reason: alerts that are always red stop being read (alert fatigue).

### A3. Alert rules — read the logic, not the name
- Collection search "disk usage" offered **"Windows Disk Usage"** rules —
  `… AND devices.os = "windows"` → would **never fire** on these Linux/Cisco devices.
- Selected instead: **"Space on / is >= 90% and < 95%"** (warning) and
  **">= 95%"** (critical): match `storage_descr = "/"` on any device. 5-min delay,
  recovery alerts on.
- Known gaps: only `/` is covered; Proxmox's **`local-lvm` thin pool isn't SNMP
  storage at all**. Notifications are web-UI only ("no operation") — no transport yet.

### A4. First real alert — hermes root disk ≥ 90%
- Fault: *Space on / is >= 90% and < 95%* on 192.168.10.206, 13:20.
- Context: the 30 GB disk used ~5.6 GB after setup in August → >27 GB now
  (~7–8 GB/month).
- Hermes (the agent) cleaned up → **87%**, alert recovered. First full cycle:
  fired → investigated → fixed → recovered.
- Open: what exactly was deleted, and the cause of the growth.

### A5. Proxmox storage check (`pvesm status`, `qm config 101`, `lvs`)
| Item | Value | Reading |
|---|---|---|
| Thin pool `data` (local-lvm) | 141 GB, **51.4%** used | Disks promise 100 GB < 141 GB → **not overcommitted** |
| `local` (dumps + ISOs) | 64% | RHEL ISO (~10 GB) still attached to VM 101 as ide2 |
| vm-101-disk-0 (root, 32 GB) | 85% | the alerting disk |
| **vm-101-disk-1 (20 GB)** | **7%** | **barely used — purpose unknown** |
| **vm-100-disk-1 (16 GB, OpenClaw backups)** | **97.9%** | **not covered by the `/` rule** |

---

## Part B — t655-fw back online (2026-11-09, hands-on)

### B1. Safe bring-up (the same routine as 10-30, now understood)
1. Cable **unplugged**, console login.
2. Header shows `WAN (re0)` and **no LAN line** → guest, not receptionist.
3. Shell: `sockstat -4l | grep ':67 '` → **nothing** → no DHCP server running.
4. Plug into **gi5** → `ifconfig re0 | grep "inet "` → **192.168.8.195**.
5. GUI from Mint.

### B2. The update that never finished
Dashboard showed **26.7** — the install image's version — and **91 updates**,
the same count as 10-30. The earlier update had **silently not completed**.
Ran it (294 MB, reboot required).
**Now: OPNsense 26.7.8_2, FreeBSD 15.1-RELEASE-p6, OpenSSL 3.5.9.**
Lesson: confirm an update by the version number afterwards, not by having clicked it.

### B3. `os-realtek-re` driver
- Lives under **"Click to view the community plugins"** (community tier is hidden by default).
- Installed + reboot. `ifconfig re0 | grep media` → `1000baseT <full-duplex>` ✅.

### B4. Security gap: anti-lockout on WAN
- Dashboard's Firewall widget listed `anti-lockout rule (WAN)`. OPNsense attaches it
  to LAN; with no LAN, it landed on WAN.
- **Tested:** phone on the Beryl Wi-Fi (VLAN 1) **reached the login page**; phone on
  the building's Wi-Fi did not (different network entirely). The "Mint only" rules
  were being bypassed for all of VLAN 1.
- Risk today is low (only Rich's devices on that Wi-Fi), but the phone/laptop are the
  likeliest to be compromised, and in Phase 1 anti-lockout would move to the
  **untrusted lab VLAN**.
- **Fix:** Firewall → Settings → Advanced → Miscellaneous → **Disable anti-lockout**.
  (Not under System → Settings → Administration in 26.x — that path was wrong.)
- Verified: Mint still loads the GUI; phone no longer does.

### B5. Config backup
System → Configuration → Backups → Download (encrypted option available).
**Never committed to the repo** — config.xml holds the root password hash and,
now, the SNMP community.

### B6. SNMP + ping, only from t640-mon → device 6
- Plugin **`os-net-snmp`** (official tier) → Services → Net-SNMP: enabled, unique
  community (generated, not a keyboard walk; never photographed), location
  "Apartment media shelf", contact Rich, **listen 192.168.8.195**. AgentX, Observium,
  Layer 3 Visibility, Display Version in OID: **off** (the last would leak the version).
- Difference from Linux: OPNsense's SNMP page has **no allowed-source field** — the
  **firewall rule** does the restricting.
- Rule form (26.x layout): Interface (rule) **WAN** (not `any`), Interface (origin)
  any, Quick on, UDP, source `192.168.8.24/32`, dest This Firewall, port 161,
  Log off (routine traffic), **State type: keep state** (= stateful).
- **First test failed** → Firewall → Log Files → **Live View** showed the request
  **blocked**. Cause: **rule saved but not Applied.** Apply → `snmpget` returned
  `t655-fw.lan`.
- Add Device then failed: `Could not connect to 192.168.8.195 using ICMP … 3/0/100%`.
  LibreNMS uses ping for up/down. Added rule: **ICMP echo request** from
  192.168.8.24 only → added. Replies go out via keep-state; nothing else can ping it.
- gi5 un-ignored in LibreNMS (firewall now on it).

---

## State at end of session

| ID | Device | IP | Notes |
|---|---|---|---|
| 1 | sg300-apt | 192.168.8.165 | gi5 = t655-fw, gi6 = t640-mon (corrected) |
| 2 | pve-node1 | 192.168.10.2 | reboot pending (kernel) |
| 3 | t640-mon | 192.168.8.24 | now on gi6 |
| 4 | docker-agents | 192.168.10.10 | rebooted 11-08 → 6.12.115 |
| 5 | hermes | 192.168.10.206 | root 87% after cleanup |
| 6 | **t655-fw** | 192.168.8.195 | **new** — OPNsense 26.7.8_2 |

t655-fw WAN rules: Mint wifi → GUI 443; Mint wired → GUI 443; t640-mon → SNMP UDP 161;
t640-mon → ICMP echo. Anti-lockout disabled. Default deny for everything else.

## Follow-ups

- [ ] **VM 100 backup disk (vm-100-disk-1) at 97.9%** — check rotation in `~/.openclaw/backups/local` (most urgent)
- [ ] hermes: what was deleted, cause of growth; purpose of the 20 GB disk-1 (7% used); grow root or move data
- [ ] Monitor Proxmox thin pool + non-root mounts (SNMP extend)
- [ ] Optional: detach/delete RHEL ISO from VM 101 (`local` 64%)
- [ ] **t655 syslog → t640-mon** (add `ip saddr 192.168.8.195 udp dport 514 accept` to t640-mon nftables)
- [ ] Inspect "Defunct rules" on t655 (likely leftover LAN rules — delete before Phase 1)
- [ ] Alert transport (Telegram) so alerts leave the web UI
- [ ] 2FA on LibreNMS · Beryl as device 7 · planned Proxmox reboot · SSH hardening
- [ ] Phase 1 (needs fieldkit USB Ethernet adapter)
