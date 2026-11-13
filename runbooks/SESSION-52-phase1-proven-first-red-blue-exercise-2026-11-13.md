# SESSION 52 — Phase 1 Proven + First Red/Blue Exercise — 2026-11-13 (night)

**Status: Phase 1 COMPLETE.** The lab (VLAN 20) is routed, NAT'd and filtered by OPNsense,
proven from a real client. Then the first attack exercise: the same scan from two positions,
measured from both sides. It also exposed a logging blind spot, which was fixed.

**Continuity:** follows `SESSION-51` (Steps 1–7 built). Runbook: `PHASE-1-runbook-vlan20-behind-opnsense.md`.
Attacker machine: **testkit** (Kali). Defender views: OPNsense Live View, `tcpdump -n -e -ttt -i pflog0`
on the t655 console, and the central log on t640-mon.

---

## 1. The test client: USB Ethernet adapter

| Expected | Actual (`dmesg`) |
|---|---|
| Realtek RTL8153, driver `r8152` | **ASIX AX88179B** (USB ID `0b95:1790`), driver **`cdc_ncm`**, appears as `eth0` |

Works out of the box on Kali. Lesson: identify hardware from `dmesg`/`lsusb`, not the listing.

- First plugged into a USB 2.0 port → `new high-speed USB device` (max 480 Mbps). Moved to the
  blue port → **`new SuperSpeed USB device`** (USB 3).
- **On battery, the adapter vanished** after an unplug/replug: no light, gi3 down, gone from
  `ip link`. Plugging in the **charger** brought it back: USB power saving had cut it.
  Lab sessions: keep testkit on the charger.
- testkit's Wi-Fi and the adapter were **both connected** at once (`nmcli device status`).
  Linux then picks one route per destination. Always check before testing:
  `ip route get <target>` → which `dev` and `src` will be used.

DHCP gave testkit **192.168.20.185** (inside the .100–.199 pool). The switch never ran DHCP, so
this alone proved OPNsense, the gi5 trunk, the VLAN tag and gi3 all work.

---

## 2. Step 8 — the nine tests (Wi-Fi off)

| # | Test | Result | Proves |
|---|---|---|---|
| 1 | Address | 192.168.20.185 | DHCP (LAB only) |
| 2 | `ip route` | default via 192.168.20.1 | OPNsense is the gateway |
| 3 | `ping 192.168.20.1` | replies | Rule 2 |
| 4 | `ping 1.1.1.1` | replies | NAT + rule 5 (internet) |
| 5 | `dig example.com` | answered by 192.168.20.1 | Rule 1 (DNS via Unbound) |
| 6 | `ping 192.168.8.186` (Mint) | 100% loss | Home blocked |
| 7 | `ping 192.168.10.2` (Proxmox) | 100% loss | Servers blocked |
| 8 | `curl https://192.168.20.1` | timeout | Rule 3 (firewall GUI from lab) |
| 9 | `curl https://192.168.8.195` | timeout | Firewall's WAN side from lab |

(Typo worth keeping: `curl -k m5 …` instead of `-k -m5` → curl tried a host called "m5" first.)

### NAT, seen in two log lines
From the pflog capture, **one packet (IP id 38757)** on both sides of the firewall:
```
pass in  on vlan01: 192.168.20.185.45588 > 108.156.211.27.443  ttl 64
pass out on re0:    192.168.8.195.3307   > 108.156.211.27.443  ttl 63
```
Source rewritten .20.185 → .195 (**NAT**), TTL −1 (**one routing hop**). Live View shows the
same pair: `LAB In … Internet only` then `WAN Out … let out anything from firewall host itself`.

---

## 3. Red/blue exercise: the ping sweep

Attacker (testkit, lab): `sudo nmap -sn 192.168.8.0/24`
Defender (t640-mon): `sudo grep '192.168.20.185' /var/log/remote/192.168.8.195.log | grep -c block`

### Run 1 — attacker blind, but so was the defender
`256 IP addresses (0 hosts up) scanned in 206.34 seconds`, yet the block count stayed **8 → 8**.

**Cause: the switch ACL.** `vlan20-policy` on gi3 denies → 192.168.8.0/24 and → 192.168.10.0/24.
Those probes were dropped **at the switch, silently**, and never reached OPNsense, which therefore
had nothing to log. (The 8 earlier blocks were the curl tests to 192.168.20.1, which the ACL
doesn't cover, so they reached OPNsense rule 3.)

**Lesson: an attack stopped by a device that doesn't log is invisible.** Defense in depth
protected the network but created a blind spot.

### Proving it
```
configure
interface gi3
no service-acl input
```
→ `ping 192.168.8.186` from testkit: still 100% loss, now **3 red lines in Live View**:
`LAB In ICMP 192.168.20.185 → 192.168.8.186 block "No home, no servers"`. Count **8 → 11 (+3)**.

### Run 2 — every probe accounted for
Count **11 → 2059 = +2048 = 256 addresses × 4 probes × 2 attempts.** `nmap -sn` sends
ICMP echo, TCP SYN :443, TCP ACK :80 and ICMP timestamp per address, and retries once when nothing
answers. **The defender could reconstruct the attacker's tool behavior from the logs alone.**

A further +2048 (→ 4107) came from a scan intended to go over Wi-Fi. Its source was still
192.168.20.185, so it had left through the adapter (both interfaces connected; see §1).

### Run 3 — from home Wi-Fi (VLAN 1)
Adapter off, `ip route get` → `dev wlan0`: **several hosts up and responding**, and the firewall
count did **not** move: nothing crossed it.

| Attacker position | Attacker sees | Defender (firewall) sees |
|---|---|---|
| **Lab (VLAN 20)** | **0 hosts** | **Every probe** (2048 per scan) |
| **Home Wi-Fi (VLAN 1)** | **Everything** | **Nothing** |

**Conclusion: segmentation protects only what's behind it.** The lab is contained, so anything
compromised there inherits a position that can see nothing private. The home network is still
flat: a device that's already inside (knows the Wi-Fi password, or a compromised phone/TV) can
see everything. Nothing on the internet can: Beryl NAT + building CGNAT + no port forwards.

### Decision: switch ACL on gi3 stays OFF
| | Protection | Visibility |
|---|---|---|
| ACL on | Two layers | Lab → home/servers attempts **invisible** |
| **ACL off (chosen)** | OPNsense, stateful, default deny | **Every attempt logged** |

The lab exists to be attacked and observed. `vlan20-policy` is kept defined on the switch (not
bound) so it can be re-attached in one line. **Switch config saved** with the ACL off.
For VLAN 10 (production) in Phase 3, two layers may still be worth it — separate decision.

---

## 4. Time, and the "attacker in Hawaii"

testkit's clock is deliberately left on **UTC−10 (Hawaii)** to play an out-of-town attacker.
nmap's banner: `21:37 -1000`. The firewall log: `01:37 -06:00`. **Same instant: 07:37Z.**

| Source | Local | UTC |
|---|---|---|
| Attacker (testkit) | 21:37 −10:00 | 07:37Z |
| Firewall (t655 → t640-mon) | 01:37 −06:00 | 07:37Z |
| Switch (SG300) | UTC | 07:37Z |

Rule: **correlate in UTC**, and never trust a timestamp without its offset. This is why the
switch's UTC clock is the professional default, not a mistake.

---

## 5. Things learned reading raw logs

- **TTL fingerprinting:** DHCP requests with TTL 255 (Apple devices) vs 64 (most Linux).
- **mDNS noise:** `block in on re0 … 224.0.0.251.5353` / `ff02::fb.5353` from Mint, the phone,
  .142, .186: devices announcing printers/AirPlay. Blocked by default deny on WAN. Harmless, but it's
  the bulk of WAN blocks and worth filtering out when hunting.
- **NTP:** `192.168.8.195.123 > …:123 NTPv4 Client`: the firewall keeping its own clock right.

---

## State at end of session

- **Phase 1 done:** lab gateway/DHCP/DNS/rules/NAT on OPNsense, proven; switch config saved.
- SG300: VLAN 20 SVI removed; gi5 trunk (1 untagged, 20 tagged); **gi3 ACL unbound**.
- Every lab → private attempt lands in Live View and in `/var/log/remote/192.168.8.195.log`.

## Next
- [ ] Post-phase1 OPNsense config backup (outside the repo)
- [ ] LibreNMS: rediscover t655-fw (LAB / vlan01 graph); keep gi3 ignored (testkit unplugs often)
- [ ] **Suricata (IDS) on the t655, LAB interface** → scan from the lab → "port scan detected" →
      Telegram. Attack → detection → phone
- [ ] Optional: disable USB autosuspend for the adapter on testkit
- [ ] **Phase 3:** VLAN 10 (servers) behind OPNsense. Then consider a VLAN for TV/guests
- [ ] Carried: Hermes nightly backup size + `df -h /` · Proxmox timezone · SG300 `cisco` user ·
      Beryl stale route `192.168.20.0/24 via .165` · 2FA · SSH hardening (Proxmox + Hermes) · Beryl firmware
