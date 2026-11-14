# SESSION 55 — Printer assessed, taken off the network, shared through CUPS + quick wins
**Dates:** 2026-11-12 → 2026-11-13
**Machines touched:** t640-mon, Mint (mintstation), Windows laptop, Kali (testkit), pve-node1, SG300
**Identifiers:** serial numbers, MAC addresses, the WPS PIN and the USB serial are written as `xxxx`. Real values are kept privately.

---

## 1. Hardware decisions

| Item | Decision | Why |
|---|---|---|
| Old parts (Pentium, Duron, micro-ATX board, ATI All-in-Wonder) | Retire / recycle | No VT-x, likely no 64-bit; can't run Proxmox or a modern stack |
| i5 tower (SLI card) | Keep as a possible future node or firewall | Any Core i5 has VT-x; remove the SLI card (power and heat) |
| 96 GB DDR4 SO-DIMM lot (2×32 GB + 2×16 GB, PC4-21300 / 2666) | **Bought for $140** | Fits pve-node1 (M720q) and t640-mon |
| Supermicro CSE-512 1U (X8SIL, 1 GB DDR3, CPU not listed) | Passed | ~2010 platform, loud 1U fans, no IPMI on the non-F board, firewall role already filled by t655 |

**RAM plan (not installed yet):**
```
2×32 GB  →  pve-node1 (M720q)   16 → 64 GB   (BIOS update first; fallback: the 2×16 pair)
2×16 GB  →  t640-mon            4 → 32 GB    (runs at the t640's lower speed — normal)
old 2×8 GB from pve-node1 → spare
```
Verify after each install: `free -h`, `dmidecode -t memory | grep -E "Locator|Size|Speed"`, then memtest86+ overnight.

---

## 2. Printer assessment — Canon PIXMA MG3620

The goal: assess a real IoT device on my own network, then fix it and prove the fix.

### 2.1 Getting it on Wi-Fi (baseline)
- The MG3620 has **USB-B and 2.4 GHz Wi-Fi only — no Ethernet**.
- Cableless setup: Stop (if the Wi-Fi lamp flashes) → **hold Wi-Fi until the ON lamp flashes** → Color → Wi-Fi → Wi-Fi lamp flashes fast, ON lamp solid → finish in the app.
- **Gotcha:** if the printer isn't in setup mode yet, **Color = start a color copy**, so it just prints a blank page.
- Joined FBI #4 (WPA2-PSK/AES) and got the IP `192.168.8.158`.

### 2.2 Scans from Kali (on the home Wi-Fi, before moving it to the lab)
```bash
sudo nmap -sV -p- 192.168.8.158 -oN printer-before-tcp.txt
sudo nmap -sU --top-ports 50 192.168.8.158 -oN printer-before-udp.txt
sudo nmap -p 443 --script ssl-enum-ciphers 192.168.8.158
```

| Port | Service | Finding |
|---|---|---|
| 80/tcp | Admin web page (KS_HTTP 1.0), **unencrypted** | 🔴 Admin traffic in plain text |
| 443/tcp | Same page over TLS | 🟡 Grade **C**: see below |
| 515/tcp | LPD (1980s, no authentication) | 🔴 Anyone on the LAN can send it jobs |
| 631/tcp | IPP (the printing protocol it actually uses) | 🟢 Needed |
| 1900/udp | UPnP | 🔴 Announces itself; history of abuse |
| 5353/udp | mDNS / Bonjour | 🟡 Leaks model/name; convenient for discovery |
| 500/udp | IPsec (open\|filtered) | 🟡 Inactive per its config page |
| 68/udp | DHCP client | 🟢 Normal |
| 161/udp | SNMP | 🟢 Closed |

**TLS (443):**
- **Old protocols:** TLS 1.0 and 1.1 still enabled; no TLS 1.3.
- **Weak ciphers:** RC4 (broken, RFC 7465), 3DES (SWEET32), and an MD5-based suite.
- **No forward secrecy:** plain RSA key exchange throughout.
- **`cipher preference: client`:** a man-in-the-middle can pick the weakest cipher.
- **Least strength: C.**
- **Can't be fixed on the device** (firmware). The remediation is a compensating control: limit who can reach it.

### 2.3 The printer's own Network Configuration Page (printed from the front panel)
- Firmware (ROM) **2.110**.
- It prints the **serial number** and the **WPS PIN** (`xxxx`), and on PIXMAs this age the serial is commonly the default admin password.
- **The finding:** anyone with physical access can print the slip that holds the admin password.
- Services enabled: WSD, **LPD**, Bonjour, PictBridge, and IPv6 (link-local). This matches the scans.
- The admin web page **asked for a password** (better than many IoT devices).

### 2.4 Fix: take it off the network entirely
1. Disabled the printer's **wireless LAN** (from Mint, because Kali on gi3 is in the lab and **can't reach the home network**, which is Phase 1 working as designed).
2. USB A-to-B cable → **t640-mon**. `lsusb` shows `Canon, Inc. MG3600 series`.

### 2.5 CUPS print server on t640-mon
```bash
sudo apt update
sudo apt install cups printer-driver-gutenprint
systemctl status cups --no-pager                       # active (running)
sudo lpinfo -v | grep -i usb                            # direct usb://Canon/MG3600%20series?serial=xxxx&interface=1
sudo lpinfo -m | grep -i mg36                           # gutenprint.5.3://bjc-PIXMA-MG3620/expert
URI=$(sudo lpinfo -v | grep -o 'usb://Canon[^ ]*')      # store the URI, so the serial is never typed
echo "$URI"
sudo lpadmin -p MG3620 -E -v "$URI" -m gutenprint.5.3://bjc-PIXMA-MG3620/expert
lpstat -p                                               # printer MG3620 is idle. enabled
lp -d MG3620 /usr/share/cups/data/testprint             # test page printed ✅
```
Gotchas:
- **Empty `$URI`:** the printer had **auto-powered off**, so the USB device disappeared. Power on, re-run.
- **`Bad device-uri "URI"`:** a missing `$`.
- **Typo:** created `MG36320`. Fixed with `sudo lpadmin -x MG36320` and a re-add.
- **"Printer drivers are deprecated":** a warning only. CUPS is moving to driverless; Gutenprint still works.
- **Stuck jobs:** `sudo cancel -a MG3620` then `sudo cupsenable MG3620`.

### 2.6 Sharing it to the home network — two layers
```
Client ──► [nftables: 192.168.8.0/24 → tcp 631 only] ──► [CUPS: @LOCAL, printer shared] ──► USB ──► printer
```
**Layer 1: nftables on t640-mon** (receiver first)
```bash
sudo cp /etc/nftables.conf /etc/nftables.conf.bak-2026-11-13
sudo nano /etc/nftables.conf
#   added under the syslog line:
#   ip saddr 192.168.8.0/24 tcp dport 631 accept
sudo nft -c -f /etc/nftables.conf          # silence = valid
sudo systemctl restart nftables
sudo nft list ruleset | grep 631
```
**Layer 2: CUPS**
```bash
sudo cupsctl --share-printers              # listen on the network, allow @LOCAL (this subnet only)
sudo lpadmin -p MG3620 -o printer-is-shared=true
ss -tlnp | grep 631                        # 0.0.0.0:631; nftables still restricts who reaches it
sudo cupsctl                               # _remote_admin=0 → anyone on the LAN can print, nobody remote can manage
```
- **Debian gotcha:** `cupsctl` is in `/usr/sbin`, which isn't on a normal user's PATH, so use `sudo cupsctl`.

### 2.7 Clients
| Client | Setup | Status |
|---|---|---|
| Windows | Add manually → shared printer by name → `http://192.168.8.24:631/printers/MG3620`, driver **Generic → MS Publisher Color Printer** (PostScript; CUPS converts it) | ✅ Test page printed |
| Mint | `sudo lpadmin -p MG3620 -E -v ipp://192.168.8.24:631/printers/MG3620 -m everywhere` | ⏳ Jobs queued but didn't print. Retest after the Mint update (fallback: `-m raw`) |
| Kali in the lab (gi3) | `nmap -p 631 192.168.8.24` | Blocked (OPNsense + nftables) |

### 2.8 Before / after
| | Before (Wi-Fi) | After (USB + CUPS) |
|---|---|---|
| On the network | Yes, 192.168.8.158 | **No IP** |
| Exposed services | HTTP, HTTPS (TLS grade C), LPD, IPP, UPnP, mDNS, IPsec | None on the printer. Only t640-mon :631, home subnet only |
| Admin access | Password-protected, but the password slip prints from the front panel | No network admin page. CUPS remote admin off |
| Reachable from the lab | — | No |

---

## 3. Quick wins from the backlog
| Item | Done |
|---|---|
| Proxmox timezone → `America/Chicago` (`timedatectl set-timezone America/Chicago`). Check the backup time in Datacenter → Backup | ✅ |
| SG300: new privilege-15 admin created, tested in a second session, `no username cisco`, `copy running-config startup-config`, `cisco` login refused | ✅ |
| Beryl: delete the stale route `192.168.20.0/24 via 192.168.8.165` | ⏳ Next. Find the section with `uci show network \| grep -B1 -A4 "192.168.20"` before deleting (the network reload blips the gateway) |

---

## 4. Mint update blocked by VirtualBox
- `dkms autoinstall … failed for virtualbox(10)` → `dpkg: error processing package linux-image-7.0.0-43-generic`.
- **Cause:** the update brought in kernel 7.0. **VirtualBox 7.0.16**'s kernel module can't build for it, so the kernel install stops.
- **Rule:** don't reboot with a half-installed kernel.
- **Fix:** remove VirtualBox (`sudo apt remove --purge 'virtualbox*'`) **or** upgrade it to 7.1+, then `sudo dpkg --configure -a`, then `sudo apt upgrade`. **Mint updated fine.**

---

## 5. Lessons learned
- **Most realistic way in = a compromised device or a public-facing app**, not a cracked Wi-Fi password. Assume breach: segment, monitor, limit reach.
- **Legacy IoT you can't patch → compensating control.** Here the device was removed from the network entirely.
- **Physical access matters:** a front-panel "print network config" page can hand out the admin password.
- **Two layers** (host firewall + application allow-list): either one alone still blocks.
- Phase 1 proved itself again: from gi3, Kali **couldn't reach the printer** on the home network.

## 6. Next
1. Mint → retest printing after the update.
2. Beryl stale-route removal.
3. Install the RAM (pve-node1 first, BIOS update before), memtest, then update the hardware docs.
4. Printer write-up for the Notebook.
5. Optional: turn off the printer's Auto Power Off so the CUPS queue doesn't stall.
6. Remaining backlog from the 2026-11-13 handoff (OPNsense config backup, Suricata EVE check, LibreNMS 2FA, SSH hardening, Beryl firmware, Phase 3).
