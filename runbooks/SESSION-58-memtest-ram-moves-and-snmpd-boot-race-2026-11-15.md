# SESSION 58: memtest on the new RAM, the RAM moved around the lab, and an snmpd boot race on t640-mon
**Date:** 2026-11-15 (about 03:15 → 08:20 CST)
**Machines touched:** pve-node1, "command" (new admin laptop), t640-mon, t655-fw, Beryl
**Identifiers:** serials, MACs and the SNMP community are written as `xxxx`.

**Continuity:** follows `SESSION-57`. Closes the memtest item from `SESSION-56` and the "monitoring box RAM" item
from the README.

---

## 1. Where the RAM went

The `SESSION-55` lot (2×32 GB + 2×16 GB) plus the parts freed up along the way:

| Machine | Before | After | How |
|---|---|---|---|
| pve-node1 (M720q) | 2×8 GB | **2×32 GB = 64 GB** | SESSION-56 |
| "command" (HP laptop, i5-8250U, the new admin machine) | — | **2×16 GB = 32 GB** | this session |
| t640-mon | 4 GB (one stick, Channel B) | **8 + 4 = 12 GB** (both channels) | this session (§4) |
| t655-fw | 4 GB | **2×8 GB = 16 GB** (the old M720q pair: 8 GB 2666 + 8 GB 3200, both run at the board's speed) | this session (§5) |
| Spare | | **2×4 GB** (Samsung 2400 + the t655's original) | |

The old Mint laptop was checked first: `dmidecode` said **DDR3**, so none of these DDR4 sticks fit it.

---

## 2. memtest86+ on pve-node1 (64 GB)

The first boot after the 64 GB swap had beeped (a seating problem, `SESSION-56`), so the RAM got a full test before
Wazuh goes on this box.

```bash
# pve-node1: before rebooting
pgrep -a vzdump          # nightly backup (02:00) not running
pgrep -a rclone          # R2 upload (03:30) not running: don't cut an upload in half
ls /boot | grep -i memtest
qm shutdown 101 ; qm shutdown 100 ; qm list   # both stopped
reboot                   # boot menu → "Memory test (memtest86+)" — NOT the "serial console" entry
```
| | |
|---|---|
| Version | memtest86+ v7.20 |
| Memory | 63.8 GB, Slot 0 and Slot 2 = 32 GB each, DDR4-2666 (1336 MHz), CAS 19-19-19-43 |
| Time | 1:09:14 for one full pass |
| Result | ✅ **Pass 1, Errors 0** |

Gotchas:
- **memtest never stops on its own.** After a pass it starts the next one (`Pass 0%` again). One clean pass is the goal; Esc to exit.
- **memtest only looks for a keyboard when it starts.** I unplugged the keyboard mid-test to use it on the t640, and Esc stopped working afterwards. A hard power-off is safe here: memtest only touches RAM, and Proxmox isn't running.
- The "serial console" entry is for headless servers watched over a serial cable. With a monitor attached, use the plain one.
- The reboot also fixed Proxmox's console resolution (it had been sized for the old, smaller monitor). The display is detected at boot.

---

## 3. memtest on "command" (32 GB)

| Attempt | What happened | Why |
|---|---|---|
| Esc at boot | HP **Startup Menu**, not the Linux boot menu | On HP, Esc opens HP's own menu |
| memtest86+ from the boot menu | **`bad shim signature`** | **Secure Boot** only runs signed programs, and Mint's memtest86+ isn't signed |
| HP diagnostics (F2) | Older version 1.24: Memory Test only, no "Extensive" option | |
| Secure Boot temporarily off → memtest86+ v7.00 | ✅ **Pass 1, Errors 0**: 31.8 GB, 2×16 GB DDR4-2666, 55 min | |

- Turning Secure Boot off on an HP asks for confirmation (a physical-presence code, or a BIOS admin password if one is set).
- **Secure Boot must go back on.** It blocks malware that loads before the OS. ✅ Turned back on after the test.

---

## 4. t640-mon: 4 → 12 GB

### 4.1 Check before opening
```bash
sudo dmidecode -t memory | grep -E "Size:|Locator:|Speed:"
```
| Slot | Before |
|---|---|
| Top-Slot 1 (top), P0 CHANNEL A | **No Module Installed** |
| Top-Slot 2 (under), P0 CHANNEL B | 4 GB, 2400 MT/s |

So a stick could simply be **added** in the top slot. The Samsung 8 GB is rated 2400, an exact match.

### 4.2 Opening a t640 (looked up, not guessed)
1. `sudo shutdown -h now`, unplug, wait ~30 s (the slots stay powered while plugged in).
2. Rear port cover off. **Press and hold the green tab** on the back, then **lift** the side panel up and off. It doesn't slide.
3. **The RAM is under the large perforated metal cover** (the one with the hot-surface sticker and a thermal pad on top). It's a **friction fit with no screws: pull it straight up.**
4. Two stacked SO-DIMM slots underneath. Insert at ~30°, press flat until the clips lock.

I first pointed at the wrong cover (the small one in the middle). The fix was to look up HP's guide and a teardown
instead of reading the board from a photo.

### 4.3 After
| Slot | After |
|---|---|
| Top-Slot 1 (top), Channel A | **8 GB**, 2400 MT/s |
| Top-Slot 2 (under), Channel B | 4 GB, 2400 MT/s |

`free -h` shows **9.6 Gi**, not 12. The R1505G's **integrated graphics borrow system RAM** (UMA frame buffer, about 2 GB
here). The box runs headless, so this can be lowered in the BIOS later (F10 → UMA frame buffer / video memory).

---

## 5. t655-fw: 4 → 16 GB

**OPNsense (FreeBSD) has no `dmidecode`.** I didn't install it on the firewall just for this: every extra package on the box at the edge is something more to patch. FreeBSD's own tools:
```
# OPNsense console → 8 (Shell)
grep -i "memory" /var/run/dmesg.boot    # real memory / avail memory
sysctl -n hw.physmem                    # bytes
```
Before: ~4 GB (one stick, one slot free, matching the hardware doc). Swapped in the matched pair from the M720q (opens like
the t640: rear port cover, green tab, lift the panel). After: **~16 GB**.

The firewall was down during the swap: servers and lab had no internet; home Wi-Fi (Beryl) was unaffected.

**Cosmetic:** the console text came back tiny (FreeBSD uses the monitor's full resolution). Cap it at the next reboot:
```
echo 'efi_max_resolution="1280x720"' >> /boot/loader.conf.local    # .local survives OPNsense updates
```

**Why the firewall got the matched pair:** everything depends on it, and Suricata (soon on two interfaces) is the
memory-hungry part. 4 GB was tight for that.

---

## 6. Finding: snmpd didn't come back after the reboot

### 6.1 Symptom → cause
After the RAM swap, **t640-mon was missing from LibreNMS**. `snmpd` wasn't running. A manual restart + enable fixed it,
but the boot log showed the real story:

```bash
journalctl -u snmpd -b --no-pager | head -20
```
```
05:02:59 snmpd: Error opening specified endpoint "udp:192.168.8.24:161"
05:02:59 snmpd: Server Exiting with code 1
05:02:59 systemd: Failed to start snmpd.service
07:00:04 systemd: Started snmpd.service        ← my manual restart
```
snmpd **was** enabled. It listens only on `192.168.8.24` (least exposure). At boot it started **before DHCP had given
the box that address**, failed to bind, and **never tried again**. It had probably been down since the last kernel
update reboot (LibreNMS showed 6.12.113 before, 6.12.117 now).

### 6.2 Fix: wait for the network, and retry if that isn't enough
```bash
sudo systemctl edit snmpd
```
```ini
[Unit]
Wants=network-online.target
After=network-online.target

[Service]
Restart=on-failure
RestartSec=10
```
```bash
systemctl cat snmpd | tail -8      # override in /etc/systemd/system/snmpd.service.d/override.conf
sudo reboot
journalctl -u snmpd -b --no-pager | head -20
```

### 6.3 Proof (reboot at 07:57)
| Time | Event |
|---|---|
| 07:57:17 | starts → `Error opening … 192.168.8.24:161` → fails |
| 07:57:27 | automatic retry #1 → fails |
| 07:57:37 | automatic retry #2 → **Started** ✅ |

It came back about **20 seconds after boot with nobody touching it**. `After=network-online.target` alone did **not**
make it wait: the DHCP lease still arrived later. The **retry is what fixed it**. The likely cause is that the interface
is `allow-hotplug` in `/etc/network/interfaces`, so the "network online" signal doesn't wait for the lease.

### 6.4 Everything else on the box
```bash
systemctl is-enabled snmpd rsyslog nftables ids-telegram cups   # enabled ×5
systemctl --failed                                                # 0 loaded units listed
```

---

## 7. Address housekeeping (from SESSION-57's follow-ups)
- Windows (.166) reserved on the Beryl. The old Mint (.142 / .186) and "command" reservations wait until the SSD arrives
  and "command" replaces the old Mint. Then: reserve it, swap it into OPNsense's `ADMINS`, drop .142/.186.
- **An IP isn't an identity:** `.213` was the phone in SESSION-48 and is "command" now. Old log lines need a timestamp (or MAC) to say who they were.

---

## 8. Lessons learned
- **One full memtest pass with 0 errors** is the bar. The test loops forever, so stop it yourself.
- memtest's keyboard is detected once, at start. Don't swap it mid-test.
- **Secure Boot blocks unsigned tools** like memtest86+. Turn it off for the test, then back on.
- **Look up the hardware before opening it.** On the t640 the RAM is under the big perforated cover, held by friction.
- **Integrated graphics borrow RAM:** 12 GB installed, 9.6 GiB visible.
- **"Enabled" isn't "comes back after a reboot."** A service bound to a specific IP can lose a race with DHCP at boot.
- **`Restart=on-failure`** is the safety net. **Reboot and read the journal** is the proof.
- Monitoring caught it: the box went missing from LibreNMS, and the journal said why.
- **FreeBSD isn't Linux:** no `dmidecode` on OPNsense. Use `dmesg.boot` / `sysctl hw.physmem`, and don't add packages to the firewall for a one-off check.

## 9. Next
1. ~~"command": turn Secure Boot back on~~ — done 2026-11-15.
2. After the t655 swap: confirm Suricata running, t655-fw green in LibreNMS, `ping 1.1.1.1` from Proxmox, a site loads on cellular.
3. Optional: `grep -E "auto|allow-hotplug" /etc/network/interfaces` on t640-mon. `auto` makes network-online meaningful. Also lower the UMA frame buffer.
4. Check the same boot behavior on the other boxes: `systemctl is-enabled snmpd` + `systemctl --failed` on Proxmox and Hermes.
5. Carried: retire OpenClaw properly · Hermes guest-agent freeze error · LibreNMS 2FA · SSH check · Suricata EVE + SERVERS · Wazuh (Phase 4).
