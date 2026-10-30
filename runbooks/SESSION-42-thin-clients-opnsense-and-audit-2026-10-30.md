# SESSION 42 — Two Thin Clients Built, OPNsense Installed, Lab Audit — 2026-10-29 → 2026-10-30

**Status: COMPLETE for this phase.** Two HP thin clients joined the lab:
`t640-mon` (Debian 13, future monitoring box) and `t655-fw` (OPNsense 26.7,
future firewall, WAN-only for now). The switch now syncs its clock over SNTP.
A full audit settled which records were stale, and the port map was corrected.

**Continuity:** follows `SESSION-41-reboot-test-and-vlan10-acl-2026-09-25.md`.
Closes SESSION-41's NTP follow-up. The `networking/` folder is still open.

---

## 0. WHY THESE TWO BOXES

The SG300's ACLs are **stateless**: each packet is judged on its own. That is
why SESSION-41 had to leave VLAN 10 → VLAN 1 open (Proxmox's replies to Mint
can't be told apart from new connections). A **stateful** firewall tracks
connections, so replies return automatically while new connections are
judged by rules. OPNsense on a thin client is that firewall.

The second box is the place for monitoring: SNMP graphs of the switch and
firewall, central syslog, later NetFlow/Zeek.

### Roles were swapped after inspecting the hardware

| | t640 | t655 |
|---|---|---|
| CPU | Ryzen R1505G, 2C/4T | Ryzen R2314, 4C |
| Storage | 128 GB **M.2 SATA SSD** (SSSTC) | 64 GB Kingston flash behind a **BayHub** bridge (eMMC-style) |
| BIOS | 2021 (updated this session) | 2024 |
| MAC | xx:xx:xx:xx:xx:xx | xx:xx:xx:xx:xx:xx |

Plan A was t640 = firewall. Reversed because **monitoring is the write-heavy
job** (graphs, logs, captures) and belongs on the real SSD, while a firewall
barely writes and benefits more from the newer BIOS and extra cores.

> **Lesson:** assign roles by what each workload actually needs from the
> hardware (writes vs. CPU vs. firmware age), not by which box arrived first.

---

## 1. t640 BIOS: M43 v01.09 → v01.24 without Windows

Full detail in `hardware/t640-mon.md`. Short version:

1. HP's flash tool must run **on the thin client itself** under Windows.
   Running it on the Windows laptop tried to load `amifldrv64.sys`, which
   **Windows' vulnerable-driver blocklist blocked. Left ON** — wrong machine,
   and disabling a security control to fix a different box is never the fix.
2. F10 → Flash System BIOS with hand-built USB layouts: "file missing or damaged."
3. Read HP's ToolLess procedure and reproduced each Windows step on Linux:
   copied `ToolLess\HP` to `/boot/efi/EFI/HP`, used F10 → **BIOS Config
   Utility** to save settings to the ESP (`EFI/HP/UefiBCU/hpisetup.txt`),
   edited `BIOS Update` from `*Disable` to `*Auto`, restored, rebooted.
   HP's own flasher ran (4096 blocks).
4. `hpisetup.txt` is **UTF-16LE with a BOM (`ff fe`)**. Plain `grep` returns
   nothing. `iconv -f UTF-16 -t UTF-8` to read/edit, convert back, re-add BOM.
5. Set `BIOS Update` back to `Disable` afterwards.

**Gotcha:** after the flash, neither the old Debian nor the Debian installer
booted — hang right after `EFI stub: Measured initrd data into PCR 9`.
`nomodeset` and removing `vga=788` did not help. **F10 → Apply Defaults and
Exit** fixed it.

---

## 2. t640-mon: Debian 13, hardened

The first Debian install was done **with no network cable**, and silently
skipped: internet apt sources (only the cdrom), the SSH server, and network
config (`enp1s0f0` left DOWN). It was wiped and reinstalled **with the cable
in gi6**, which fixed all three.

| Item | Value |
|---|---|
| Port / IP | SG300 **gi6**, VLAN 1, **192.168.8.24** (Beryl reservation) |
| OS | Debian 13.7 netinst, no desktop, SSH + standard utilities |
| Users | root login disabled (root password left blank), `scgiky` has sudo |
| Updates | `unattended-upgrades` enabled |
| SSH | key-only from Mint (`~/.ssh/homelab_mint`, Host `t640-mon`); `PasswordAuthentication no`, `PermitRootLogin no`; verified with a second session |
| Reboot test | passed — same IP, key login works |

Mint had only one key (`hetzner_mint`), so `ssh-copy-id` failed with
"No identities found." A **separate homelab key** was created rather than
reusing the Hetzner one, with `IdentitiesOnly yes` so SSH never offers the
Hetzner key to home hosts.

---

## 3. t655-fw: OPNsense 26.7

### Hardware check in live mode
- `re0` found: Realtek 8168/8111, MAC `xx:xx:xx:xx:xx:xx`, 1000baseT full duplex
- Disk `nda0`: BAYHUB Kingston-TY2964 64GB (58G usable)
- `rtw890` firmware errors = the Realtek **Wi-Fi** card. Unused; ignore.
- Booted with Secure Boot already off (FreeBSD/OPNsense doesn't support it).

### Near-miss 1: rogue DHCP
In live mode `re0` was assigned as **LAN**. OPNsense LAN defaults to
`192.168.1.1/24` **with a DHCP server**, and gi5 is on VLAN 1 — the same
network as the Beryl. That made the t655 a rogue DHCP server for a few
minutes. The guidance given was to plug into gi5 without warning that the
interface prompt would appear or what to answer.

Checked from Mint afterwards — no damage: Mint kept `.186`, the switch still
answered at `.165`, Proxmox reachable at `.10.2` with TTL 63.

### Near-miss 2: "WAN-only" still ran a DHCP listener
After installing with **WAN = re0, no LAN**, `sockstat -4l | grep ':67 '`
still showed `dnsmasq *:67`. The config still held the old LAN ranges
(`dhcp-range=tag:lan,192.168.1.100,192.168.1.199`). Stopped with
`configctl dnsmasq stop` before the cable went in.

### Near-miss 3: the wizard reassigned the port
After the setup wizard, a reboot showed **`LAN (re0)`** again. Config confirmed
`<lan><if>re0</if>`. Cable pulled, no address, nothing on :67 — reassigned to
**WAN = re0** from the console menu (option 1). Survived the next reboot.

### Getting into the GUI
WAN blocks all inbound by default, including the GUI. Sequence that worked:
1. Console shell: `pfctl -d` (filter off, briefly)
2. Browse to `https://192.168.8.195`, check the certificate fingerprint
   matches the one printed on the console
3. Firewall → Rules → WAN: **Pass TCP from 192.168.8.186/32 and
   192.168.8.142/32 → This Firewall : 443** (Mint Wi-Fi / Mint wired)
4. Apply, then `pfctl -e` and reload the GUI — still loads = rule proven

**Gotcha:** finishing the wizard reloads the ruleset, which re-enables the
filter and locks you out until the WAN rule exists.

| Item | Value |
|---|---|
| Hostname | `t655-fw.lan` |
| WAN | `re0`, DHCP, **192.168.8.195** (Beryl reservation) |
| LAN | none yet (VLAN 20 sub-interface comes in Phase 1) |
| GUI access | Mint `.186` and `.142` only, HTTPS |
| Wizard | Disable LAN ✓, no DHCP server, Multiwan/auto-DHCP-registration off, block private + bogons off on WAN |

---

## 4. Audit — what is actually true

Run from Mint against the switch, Proxmox and both VMs.

| Item | Result |
|---|---|
| `vlan20-policy` | deny 192.168.10.0/24, deny host .8.165, deny 192.168.8.0/24, permit any — **present** |
| `vlan10-policy` | deny 192.168.20.0/24, deny host .8.165, permit any — **present** |
| pve-node1 | gi2 Up, 1000 Full |
| VM 100 docker-agents | `192.168.10.10` |
| VM 101 hermes | `192.168.10.206` |
| SNTP | settings present, **no server** → fixed in §5 |

Several notes outside the repo still said "ACLs not yet written" and "VMs
staged." The repo was right; the notes were stale.

### Findings
- **gi5 at 100 Mbps** turned out to be **Mint**, not the t655. Mint's
  RTL8105e is Fast-Ethernet-only. Mint moved to gi4 (see §6).
- **OpenClaw OOM:** docker-agents console showed the kernel killing
  `openclaw-gateway` twice, days apart. Fixed by restart plus an automatic
  restart when it hits a memory limit. *Mechanism (Docker `--memory` +
  restart policy vs. systemd `MemoryMax` + `Restart=`) to be recorded in the
  VM doc.* Watch the restart count: frequent restarts = a leak being hidden.
- Hermes `Last login ... from 192.168.8.235` (Oct 23) — **source not yet
  identified.** Check the Beryl client list.
- Proxmox accepts **root password SSH**. Confirms the open "Proxmox SSH
  hardening" item.

---

## 5. SNTP on the switch — DONE

```
configure
sntp server 162.159.200.1 poll
sntp unicast client enable
sntp unicast client poll
clock source sntp
exit
show sntp status
show clock
```
Result: `Clock is synchronized, stratum 3, reference is 162.159.200.1`.
Raw IP used so the switch doesn't need DNS.

The first offset was **469,333,167 ms ≈ 5.4 days.** Every switch log before
this session carried timestamps days off — logs from different devices
couldn't be lined up. Switch stays on UTC on purpose.

---

## 6. Port map corrected + isolation retest

Mint's cable was on gi5 (the port reserved for the firewall trunk). It went
into gi3 first by mistake, which turned into an unplanned test:

```
nmcli radio wifi off
ip -br a      # enp1s0 UP, NO address  → PASS (VLAN 20: no DHCP, ACL'd)
nmcli radio wifi on
```
Then moved to gi4 → `enp1s0 192.168.8.142`, gi4 Up/100.

Switch log from the move, now with correct timestamps:
```
19:33:32 %LINK-I-Up: gi4
19:33:37 %STP-W-PORTSTATUS: gi4: STP status Forwarding
```
5 seconds to forwarding — **Rapid STP**, not classic STP's ~30 s.

---

## 7. State at end of session

```
WhiteSky (wired; WiFi repeater = failover)
        |
   Beryl AX 192.168.8.1
        |  reservations: switch .165, t640-mon .24, t655-fw .195
   SG300-10 sg300-apt 192.168.8.165  (Router mode, SNTP synced)
        |
        +-- gi1  VLAN 1   Beryl uplink
        +-- gi2  VLAN 10  pve-node1 + VMs .10.10 / .10.206 — vlan10-policy
        +-- gi3  VLAN 20  lab — vlan20-policy
        +-- gi4  VLAN 1   Mint (wired, 100 Mbps NIC)
        +-- gi5  VLAN 1   t655-fw WAN (future trunk)
        +-- gi6  VLAN 1   t640-mon
```

## Follow-ups

- [x] SNTP (from SESSION-41)
- [ ] `copy running-config startup-config` after the SNTP change — confirm saved
- [ ] t655-fw: confirm Services → Dnsmasq **Enable unticked** (stops it at boot)
- [ ] t655-fw: download a config backup
- [ ] t655-fw: 91 pending updates + `os-realtek-re` plugin + reboot test
- [ ] t640-mon: SNMP on the SG300, LibreNMS, syslog receiver (log rotation day one)
- [ ] OpenClaw: record the restart mechanism in the VM doc; watch restart count
- [ ] Identify `192.168.8.235` (Hermes login, Oct 23)
- [ ] Proxmox SSH hardening (key-only, no root password)
- [ ] OPNsense Phase 1: gi5 trunk + VLAN 20 behind the firewall
- [ ] Build the `networking/` folder (from SESSION-41)
