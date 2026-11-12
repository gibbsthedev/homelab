# SESSION 51 — Phase 1 Build: VLAN 20 Moves Behind OPNsense — 2026-11-12 (night)

**Status: Steps 1–7 of 8 DONE. Step 8 (proof from a lab client) waits for the USB
Ethernet adapter.** The lab VLAN is no longer routed by the switch's stateless ACL. The
t655 (OPNsense) is now its gateway, DHCP server, DNS resolver and **stateful** firewall,
with NAT out to the internet. This is the "stateless → stateful" step the README describes.

**Continuity:** follows `SESSION-50`. Plan + step-by-step: `PHASE-1-runbook-vlan20-behind-opnsense.md`.
Session ran ~02:00–04:45. Switch config deliberately **not saved** until Step 8 passes,
so a switch reboot reverts the switch side to the known-good state.

---

## 0. Why VLAN 20 first

VLAN 20 was **empty**: gi3 had nothing plugged in, which is also why LibreNMS reported
"Port 20 down" in SESSION-47. So every change could be made and undone without touching
anything in use (Hermes, the public sites, Mint, the internet). VLAN 10 (production)
moves later, in Phase 3, once this pattern is proven.

---

## 1. Prep

### Test client hardware
A USB 3.0 Gigabit adapter with a **Realtek RTL8153** chip (e.g. TP-Link UE300 / UE300C).
Linux ships its driver (`r8152`), so it works on plug-in. Avoid USB 2.0 "10/100"
adapters (RTL8152/AX88772) and listings that don't name the chip. Arrives 11-12.

### OPNsense config backup
System → Configuration → Backups → Download → saved as `…-pre-phase1-2026-11-12.xml`,
**outside the repo** (root password hash, SNMP community).

### Defunct rules deleted
Firewall → Rules → **Defunct rules**: two leftovers from the wizard, when it briefly made
`re0` the LAN (the October rogue-DHCP incident):

| Rule | Interface | Source → Destination |
|---|---|---|
| IPv4 | `lan` (no longer exists) | lan net → any, any protocol |
| IPv6 | `lan` | lan net → any, any protocol |

"Allow LAN to anything", waiting for an interface named `lan` to exist. Deleted → Apply
(24 → 22 entries). The new interface ended up as `opt1`, so they wouldn't have attached,
but **a firewall should never hold rules nobody wrote or reviewed.**

### Reading the switch (read-only)
| Command | Finding |
|---|---|
| `show vlan` | VLAN 1 = gi1, gi4–10 · VLAN 10 = gi2 · **VLAN 20 = gi3** → the test port |
| `show interfaces switchport gi5` | **Already `Trunk`**, native VLAN 1 untagged, member of VLAN 1 only. Docs said "access" → **doc drift** (good drift: Step 1 became one command) |
| `show ip interface` | SVIs: .8.165 (vlan 1), .10.1 (vlan 10), **.20.1 (vlan 20)** |
| `show access-lists` | `vlan20-policy`: deny → 192.168.10.0/24, deny → 192.168.8.165, deny → 192.168.8.0/24, permit any |
| `show running-config` | `interface gigabitethernet3` → `service-acl input vlan20-policy` (and gi2 → `vlan10-policy`). **Port-bound**, so it keeps working after the hand-off |

Gotcha: firmware 1.1.2.0 has **no `| include` filter** (`% Wrong number of parameters`).
Page through `show running-config` with space instead.

---

## 2. The build

### Step 1 — Switch: carry VLAN 20 to the firewall
```
configure
interface gi5
switchport trunk allowed vlan add 20
```
The switch logged `%LINK-I-Up: Vlan 20` immediately. **Why:** a VLAN interface only comes up
when at least one member port has a live link. gi3 is empty, but gi5 (the t655) is plugged
in. That's the same rule that made LibreNMS alert "Port 20 down" in SESSION-47.
Result: gi5 = VLAN 1 **Untagged** + VLAN 20 **Tagged**. The t655's WAN is unchanged.

### Step 2 — OPNsense: VLAN device
Interfaces → Devices → VLAN → **+**: parent `re0`, tag `20`, description LAB → device **`vlan01`**.

### Step 3 — The hand-off: switch gives up 192.168.20.1
```
configure
interface vlan 20
no ip address 192.168.20.1        # the form WITH the mask is rejected on 1.1.2.0
```
`show ip interface` → only .8.165 and .10.1 remain. **The switch no longer routes VLAN 20.**
The only way out of the lab is now through the firewall.

### Step 4 — OPNsense: LAB interface
Interfaces → Assignments → red **+** (26.x layout) → device `vlan01`, description LAB →
identifier **`opt1`**. Enabled, **Static IPv4 192.168.20.1/24**, no upstream gateway,
IPv6 none, block private/bogons off. Assignments now: `wan` → re0 (DHCP) · `opt1` LAB → vlan01 (192.168.20.1/24).

### Step 5 — DHCP for the lab only
Services → Dnsmasq DNS & DHCP. Its own help text describes the October incident:
> *"If no interfaces are selected, Dnsmasq will listen on all available IPv4 and IPv6 addresses."*

- General: **Interface = LAB only**, Enable ✅, listen port left at `53053` (placeholder),
  so dnsmasq stays off port 53 and **Unbound** keeps DNS.
- DHCP ranges: LAB, **192.168.20.100 – 192.168.20.199**.
- Two layers protect VLAN 1: (1) dnsmasq is scoped to LAB; (2) OPNsense only creates DHCP
  firewall rules on explicitly selected interfaces, so WAN still blocks inbound DHCP.
  The rule count went 22 → 30: the automatic LAB rules (DHCP + default block) appeared.
- Check on t640-mon:
  ```
  sudo grep 'block' /var/log/remote/192.168.8.195.log | grep ',68,67,' | tail -3
  ```
  → `re0 … block,in … 0.0.0.0 → 255.255.255.255 … 68,67`. Latest lines were from 03:29,
  **before** dnsmasq was enabled. **Re-run later** to see blocks after enabling.

### Step 6 — Rules on LAB
Alias **`PRIVATE_NETS`** = `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`.

| # | Action | Proto | Source | Destination | Port | Log | Description |
|---|---|---|---|---|---|---|---|
| 1 | Pass | TCP/UDP | LAB net | LAB address | 53 | off | DNS from firewall |
| 2 | Pass | ICMP echo | LAB net | LAB address | — | off | Ping gateway |
| 3 | Block | any | LAB net | This Firewall | any | ✅ | Protect firewall |
| 4 | Block | any | LAB net | PRIVATE_NETS | any | ✅ | No home, no servers |
| 5 | Pass | any | LAB net | any | any | ✅ | Internet only |

All: interface LAB, direction in, Quick, **keep state**. First match wins.

**Bug caught before testing:** rule 5 was saved as **Block**. The description said
"Internet only", but the icon was a red ✕. The lab would have had DNS and nothing else, and
Step 8's internet test would have sent the troubleshooting toward NAT. Fixed → Pass → Apply.
**A firewall does what the action says, not what the label says. Read the icons.**

Rule 2 was added during the build: without it, rule 3 blocks pinging the gateway, which is
the first thing anyone does when troubleshooting.

**Old ACL vs new rules:** → .10/24 ✅ rule 4 · → switch .165 ✅ rule 4 · → .8/24 ✅ rule 4 ·
permit any ✅ rule 5. New: the firewall itself is protected (3), traffic is **stateful**, and
blocks are **logged** to t640-mon. The old ACL still runs on gi3 in front of it: **two layers.**

### Step 7 — NAT (read-only check)
Firewall → NAT → Outbound (Source NAT): **Automatic Source NAT rule generation**, auto rule on
`wan`, source **LAB network** (+ loopback) → **interface address** (192.168.8.195).

---

## 3. The IP addresses — why each one, and where it went

### The pattern behind every number
```
192.168.  20  .  1
          │      └── host part: .1 = the gateway (same convention as .8.1 Beryl, .10.1 switch)
          └── network: matches the VLAN ID — VLAN 10 = 192.168.10.x, VLAN 20 = 192.168.20.x
```
**Third octet = VLAN number.** When `192.168.20.137` shows up in a log, you know it's the lab
without looking anything up. The /24 mask means the first three octets name the network and
the last one the host: 254 usable addresses, more than enough, the same size as the others.

### Every address in Phase 1

| Address / value | What it is | Where it was entered | Why this value |
|---|---|---|---|
| **192.168.20.0/24** | The lab network | Already existed since SESSION-41 (switch SVI) — **kept** | Matches VLAN 20. Keeping it means the existing ACLs (`vlan10-policy` denies → 192.168.20.0/24) and the docs stay correct |
| **192.168.20.1** | The lab's **gateway** | **Removed** from the switch (Step 3: `interface vlan 20` → `no ip address`), then **entered** on OPNsense (Step 4: LAB interface, Static IPv4) | `.1` = gateway convention. **Same address, different device behind it:** clients keep the same gateway. Removed *before* adding, because two devices answering for one IP fight over it (an ARP conflict) |
| **/24** | Network size (mask) | Step 4, next to 192.168.20.1 | Must match the real network size. A wrong mask would make OPNsense think lab hosts are elsewhere |
| *(no upstream gateway)* | Where LAB sends traffic onward | Step 4, left empty | Only the **outside** interface (WAN) has a gateway (the Beryl, learned by DHCP). A gateway on LAB would make OPNsense treat LAB as a second internet link, and automatic NAT would skip it |
| **VLAN tag 20** | A label on frames, not an address | Step 1 (switch trunk: `allowed vlan add 20`) and Step 2 (OPNsense device: tag 20) | **Both ends of the trunk must use the same number,** or the frames are dropped as unknown |
| **192.168.20.100 – .199** | DHCP pool (addresses handed out automatically) | Step 5, Dnsmasq → DHCP ranges | Leaves **.2–.99 for fixed (static) addresses**: a Kali attack box, a vulnerable VM, a future IDS sensor. Those need addresses that never change, so rules and logs can name them. "1xx" = handed out by DHCP; easy to recognise |
| **192.168.20.1 as DNS** | The DNS server given to clients | Automatic (DHCP hands out the interface address) | Unbound on the firewall. Every lab lookup happens in one place you control and can log. Rule 1 allows exactly this |
| **53053** | dnsmasq's DNS port | Step 5, left at the default | Port 53 belongs to Unbound. Two services on one port means one fails to start |
| **192.168.8.195** | t655's WAN address (VLAN 1) | Unchanged: Beryl DHCP reservation (SESSION-42) | **NAT target** (Step 7). Lab traffic leaves looking like it came from .195, an address the Beryl already knows |
| **10/8, 172.16/12, 192.168/16** | `PRIVATE_NETS` alias | Step 6, Firewall → Aliases | The three private ranges reserved by RFC 1918. They cover home (.8), servers (.10) and any future VLAN, so one rule blocks them all |
| .186 / .142 / .24 | Mint wifi / Mint wired / t640-mon | Unchanged WAN rules | Still the only devices that can reach the firewall's GUI / SNMP |
| 192.168.10.1 | VLAN 10 gateway | Unchanged, still on the switch | VLAN 10 moves behind the firewall in Phase 3, not now |

### Why NAT instead of a route (and a stale route to clean up)
The Beryl has a **return route** (added 2026-09-06, Layer 3 migration): `192.168.20.0/24 via 192.168.8.165` (the
switch). It's now **stale**: the switch no longer has VLAN 20's gateway. It does no harm,
because with NAT the lab's traffic comes back to **192.168.8.195**, not to 192.168.20.x, so the
Beryl never needs that route. **To-do:** remove it from the Beryl. A dead route is a future
troubleshooting trap.

### A packet's trip: testkit → internet
```
testkit 192.168.20.1xx ──(untagged)── gi3 [VLAN 20 access]
   │ switch: vlan20-policy ACL on gi3 (stateless layer 1)
   ▼
SG300 ──(tag 20)── gi5 trunk ──► t655 vlan01 = LAB 192.168.20.1
   │ OPNsense: rules 1→5 (stateful layer 2), log, keep state
   │ NAT: source 192.168.20.1xx → 192.168.8.195
   ▼
t655 re0 = WAN 192.168.8.195 ──(untagged, VLAN 1)── gi5 ──► gi1 ──► Beryl 192.168.8.1 ──► internet
```
The reply comes back to .195. OPNsense's **state table** remembers the conversation, reverses
the NAT and hands it to testkit. No rule is needed for replies: that's "stateful".

---

## 4. Findings along the way (side quests → to-do)

- **SG300 SNMP community photographed** (visible in `show running-config`), and it was a
  name/date pattern, not random. **Rotated 11-12.** LibreNMS's credential must be updated or
  the switch alerts as unreachable. (Lesson: `show running-config` prints secrets. Crop or
  don't photograph it.)
- **SG300 admin user is the factory default `cisco`.** Attackers then only need the password.
  → Create a new admin, remove `cisco`.
- **Clocks:** the switch is correct UTC via SNTP, but has **no timezone** (logs show 09:29 for
  03:29 local). **Mint's clock shows the wrong zone too** (02:18 vs 04:18). Mismatched times
  make log correlation across devices painful.
- **Beryl's stale route** `192.168.20.0/24 via 192.168.8.165` (see §3).
- **Docs drift:** gi5 was already a trunk; `sg300-10.md` said access.

---

## State at end of session

```
VLAN 20 (lab) ── gi3 ── [ACL vlan20-policy] ── SG300 ══ gi5 trunk (1 untagged, 20 tagged) ══ t655
                                                                        │ LAB 192.168.20.1/24
                                                                        │ DHCP .100–.199 (LAB only)
                                                                        │ rules 1–5, logged
                                                                        │ NAT → WAN .195 → Beryl
```
- Switch: VLAN 20 SVI **removed**; gi5 carries VLAN 20 tagged; **config NOT saved** (on purpose).
- OPNsense: LAB = opt1 / vlan01, dnsmasq DHCP on LAB only, 5 rules, automatic outbound NAT.

## Next (Step 8 — with the adapter)
1. testkit: `ip link` / `dmesg | tail` → new interface + `r8152`.
2. **Wi-Fi OFF**, plug into **gi3**.
3. The nine tests (runbook §8): DHCP, gateway, internet, DNS, then blocks to Mint, Proxmox, the
   GUI from both sides.
4. Watch the blocks in Live View and on t640-mon (`grep block … 192.168.8.195.log`).
5. Pass → `copy running-config startup-config` on the switch, fresh OPNsense backup, docs.

## Follow-ups
- [ ] Re-run the DHCP-block grep (post-dnsmasq timestamps)
- [ ] Step 8 tests · save switch config · post-phase1 OPNsense backup
- [ ] Update LibreNMS credential for the rotated switch community
- [ ] SG300: replace the `cisco` admin user · set timezone (`clock timezone`)
- [ ] Mint: fix timezone
- [ ] Beryl: remove the stale route `192.168.20.0/24 via 192.168.8.165`
- [ ] Then: red/blue exercise (nmap from the lab, watch the logs) → Suricata on the t655
- [ ] Carried: Hermes first-backup check · 2FA · Proxmox SSH hardening · Beryl firmware
