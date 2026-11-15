# SESSION 57: Beryl cleaned up and upgraded, servers (VLAN 10) moved behind the firewall
**Dates:** 2026-11-13 (evening) → 2026-11-15 (about 02:40 CST)
**Machines touched:** Beryl AX, SG300, t655-fw (OPNsense), pve-node1, Hermes, OpenClaw, t640-mon, Mint, Windows laptop
**Identifiers:** MAC addresses, serials and the SNMP community are written as `xxxx`. Real values are kept privately.

**Continuity:** follows `SESSION-56` (RAM 16 → 64 GB, OpenClaw back). This is **Phase 3**: Phase 1 put the
lab (VLAN 20) behind OPNsense in `SESSION-51`/`52`. This session does the same for the servers and closes
the README's known gap: until now, any device on the home Wi-Fi could reach the Proxmox login page.

---

## 1. Beryl: stale route removed

`192.168.20.0/24 via 192.168.8.165` pointed lab traffic at the switch. Since Phase 1 the lab is routed and
NAT'd by OPNsense, so nothing used it. The `192.168.10.0/24` route was still needed (until §3).

```bash
# Beryl (ssh root@192.168.8.1)
uci show network | grep -B1 -A4 "192.168.20"     # → network.@route[0] … target='192.168.20.0'
uci show network.@route[0]                         # only the 20.0 lines (section cfg14c8b4)
ip route del 192.168.20.0/24 via 192.168.8.165     # live, no network reload, no outage
uci delete network.@route[0]                       # saved config
uci commit network
ip route | grep 192.168                            # 8.0/24 + 10.0/24 via .165 only
uci show network | grep route                      # one route left: target 192.168.10.0
```
**Gotcha:** uci list indexes shift. After deleting `@route[0]`, the 10.x route **became** `@route[0]`.
Running the delete twice would have removed the route the servers depended on. Delete once, then re-read.

---

## 2. Beryl firmware 4.8.1 → 4.11.0

### 2.1 Is it a beta?
GL.iNet's own firmware-status page still listed 4.8.1 as the latest for the GL-MT3000, and newer builds had
been beta-only. But the router's **System → Upgrade** page offered **4.11.0, type "release3"**, with **Accept
Preview Plan OFF**. So this was a normal release, not a preview. Taken.

### 2.2 Backup first
```bash
# Beryl: record what must survive
opkg list-installed | grep -i snmp
uci show system | grep log_
uci show snmpd | head -20
sysupgrade -b /tmp/beryl-pre-4.11-2026-11-13.tar.gz
```
```bash
# Mint: two separate commands
scp -O root@192.168.8.1:/tmp/beryl-pre-4.11-2026-11-13.tar.gz ~/
ls -lh ~/beryl-pre-4.11-2026-11-13.tar.gz
```
Gotchas:
- `-O` is a **capital letter O** (legacy SCP protocol; the Beryl's dropbear has no SFTP server). `-0` (zero) → `unknown option`.
- Two commands typed on one line → scp treated `ls -lh …` as more files to copy.
- The backup holds Wi-Fi passwords and the SNMP community: **kept out of the repo**.

### 2.3 Upgrade (Keep Settings ON) and the snmpd loss
After the upgrade, `opkg list-installed | grep -i snmp` was **empty**. GL.iNet upgrades keep the *settings*
but drop packages you installed yourself. The config file survived; only the program was gone.
```bash
ls -l /etc/config/snmpd        # still there, with my community + t640-mon restriction
opkg update
opkg install snmpd             # kept the existing /etc/config/snmpd
/etc/init.d/snmpd enable
/etc/init.d/snmpd start
ps | grep [s]nmp
```
Verified: `snmpget … 192.168.8.1 sysName.0` from t640-mon, beryl-ax **up** in LibreNMS, fresh lines in
`/var/log/remote/192.168.8.1.log`. The 10.x route and `log_ip` survived the upgrade.

---

## 3. Phase 3: VLAN 10 behind OPNsense

### 3.1 Design
**OPNsense takes over the gateway address `192.168.10.1` from the switch.** Proxmox, Hermes and OpenClaw keep
their IPs, gateway and DNS. Nothing is changed inside the servers.

```
BEFORE                                          AFTER
Mint/t640 ─► Beryl ─► SG300 (.10.1, ACL) ─►      Mint/t640 ─► Beryl ─► OPNsense WAN .195
                        VLAN 10                                    │ stateful rules + logs
                                                                   ▼ (tag 10 on the gi5 trunk)
                                                              OPNsense SERVERS .10.1 ─► VLAN 10
Servers ─► SG300 ─► Beryl ─► internet           Servers ─► OPNsense (NAT) ─► Beryl ─► internet
```

| | Before | After |
|---|---|---|
| 192.168.10.1 | SG300 SVI | OPNsense `SERVERS` (opt2, device `vlan02`, re0 tag 10) |
| Beryl route 192.168.10.0/24 | via .8.165 (switch) | via **.8.195** (firewall) |
| gi5 trunk | VLAN 1 untagged + 20 tagged | + **10 tagged** |
| Who can reach the servers from VLAN 1 | every device | **Mint + Windows** (SSH 22, GUI 8006, ping), **t640-mon** (SNMP, ping) |
| Blocks | silent (switch ACL) | logged → t640-mon |

### 3.2 Pre-flight (read only)
- **OPNsense config backup** `pre-phase3` (outside the repo).
- **What the servers depend on:**

| Host | Gateway | DNS | Impact |
|---|---|---|---|
| pve-node1 | 192.168.10.1 | 104.222.16.6 (WhiteSky), 8.8.8.8 | Internet DNS → covered by the internet rule. No DNS rule needed |
| Hermes | 192.168.10.1 | 100.100.100.100 (Tailscale MagicDNS) | Goes out over the internet |
| OpenClaw | 192.168.10.1 | 100.100.100.100 (Tailscale MagicDNS) | Same |

- **Name every address:** OpenClaw's `Last login … from 192.168.8.166`. That line shows the **previous**
  login, not the current one. `ipconfig` proved .166 is the **Windows laptop** on FBI. It's the backup admin
  machine, so it joined `ADMINS`. On the apartment's WhiteSky Wi-Fi it gets a CGNAT address (`100.110.x.x`) and
  is outside my network entirely.

### 3.3 Prep (no impact)
**SG300:**
```
configure
interface gi5
switchport trunk allowed vlan add 10
exit
exit
show interfaces switchport gi5     # 1 Untagged · 10 Tagged · 20 Tagged
```

**OPNsense, three screens:**
1. **Interfaces → Devices → VLAN → +**: Device blank (auto-named `vlan02`), Parent `re0`, tag `10`, priority default, description `SERVERS` → Save → Apply.
2. **Interfaces → Assignments**: "Assign a new interface" → device `vlan02 (SERVERS)`, description `SERVERS` → **+ Add** → Save. Becomes `opt2`.
3. **Interfaces → [SERVERS]**: **Enable left OFF** (the switch still owned .10.1, so enabling would mean two devices with the same IP), Static IPv4 `192.168.10.1/24`, IPv6 None, block private/bogons off, **Upstream Gateway left on Auto-detect** (a gateway would make OPNsense treat it as a WAN).

**Aliases:**
| Name | Type | Content |
|---|---|---|
| `SERVERS_NET` | Network(s) | 192.168.10.0/24 |
| `ADMINS` | Host(s) | 192.168.8.142 (Mint wired), .186 (Mint Wi-Fi), .166 (Windows) |
| `MONITOR` | Host(s) | 192.168.8.24 (t640-mon) |
| `ADMIN_PORTS` | Port(s) | 22, 8006 |

I used a `SERVERS_NET` alias instead of the built-in "SERVERS net", because built-ins may not be offered while the interface is disabled.

**SERVERS rules** (first match wins; Quick, in, keep state):
| # | Action | Proto | Source | Destination | Port | Log | Description |
|---|---|---|---|---|---|---|---|
| S1 | Pass | ICMP echo | SERVERS_NET | 192.168.10.1 | — | off | Ping gateway |
| S2 | Block | any | SERVERS_NET | This Firewall | any | **on** | Protect firewall |
| S3 | Block | any | SERVERS_NET | PRIVATE_NETS | any | **on** | No home, no lab |
| S4 | Pass | any | SERVERS_NET | any | any | off | Internet |

S4 isn't logged on purpose: the Cloudflare tunnels, Tailscale and the Telegram bots talk constantly. Replies to Mint and t640-mon pass by **state**, not by a rule.

**WAN rules added** (after the existing 4):
| # | Action | Proto | Source | Destination | Port | Description |
|---|---|---|---|---|---|---|
| 5 | Pass | TCP | ADMINS | SERVERS_NET | ADMIN_PORTS | Admins → SSH/GUI |
| 6 | Pass | ICMP echo | ADMINS | SERVERS_NET | — | Admins ping servers |
| 7 | Pass | UDP | MONITOR | SERVERS_NET | 161 | SNMP from t640-mon |
| 8 | Pass | ICMP echo | MONITOR | SERVERS_NET | — | LibreNMS up/down |

**Caught in review before the cutover (by reading the rule list, not the descriptions):**
| Rule | Saved as | Problem | Fix |
|---|---|---|---|
| S1 Ping gateway | Protocol **any** | Being first, it would have let the servers reach the firewall's GUI and SSH, so S2 would never run | ICMP, echo request |
| S2 Protect firewall | Source **any** | Harmless on this interface, but inconsistent | SERVERS_NET |
| WAN 6 Admins ping | **UDP**, any port | Would have let admin machines send any UDP to the servers | ICMP, echo request |
| WAN 8 | missing | LibreNMS would mark all three servers down | added |

### 3.4 Cutover (Steps 5–7, back to back)
```
# SG300: give up 192.168.10.1   (outage starts)
configure
interface vlan 10
no ip address 192.168.10.1
exit
exit
show ip interface        # only 192.168.8.165 (vlan 1) left
```
**OPNsense:** Interfaces → [SERVERS] → **Enable** → Save → Apply.
```bash
# Beryl: send VLAN 10 traffic to the firewall
uci set network.@route[0].gateway='192.168.8.195'
uci commit network
ip route replace 192.168.10.0/24 via 192.168.8.195
ip route | grep 192.168.10        # 192.168.10.0/24 via 192.168.8.195
```
`vlan10-policy` was left bound on gi2: a stateless second layer (denies → lab, → switch login).

### 3.5 Proof
| # | Where | Test | Result |
|---|---|---|---|
| 1 | Proxmox | `ping -c3 192.168.10.1` | ✅ 0% loss, 0.18 ms |
| 2 | Proxmox | `ping -c3 1.1.1.1` | ✅ 0% loss, 7.2 ms (NAT + S4) |
| 3 | Proxmox | `ip neigh show 192.168.10.1` | ✅ the gateway's MAC is the **t655's re0** (see below) |
| 4 | OPNsense | Interfaces → Overview | ✅ SERVERS (opt2) up, `vlan02`, VLAN 10, 192.168.10.1/24 |
| 5 | Proxmox | `getent hosts google.com` | ✅ resolves |
| 6 | Proxmox | `ping -c3 192.168.8.24` | ✅ 100% loss: servers can't start connections home (S3) |
| 7 | Proxmox | `ping -c3 192.168.20.1` | ✅ 100% loss: no lab, no firewall |
| 8 | Mint | ssh OpenClaw + Hermes | ✅ both log in (WAN 5) |
| 9 | Mint | Proxmox GUI | ✅ |
| 10 | LibreNMS | all devices | ✅ all 7 up, including docker-agents, pve-node1 and hermes through the new rules |
| 11 | Cellular / Telegram | public sites, both bots | ✅ |
| 12 | Phone on FBI Wi-Fi | https://192.168.10.2:8006 | ✅ **blocked**: the known gap is closed |

**How the MAC proves the gateway moved (without publishing it):** OPNsense's WAN IPv6 link-local address
is `fe80::xxxx:xxff:fexx:xxxx`. IPv6 builds it from the NIC's MAC (EUI-64): flip one bit of the first byte
(e.g. `d0 → d2`) and insert `ff:fe` in the middle. Reversing it gives re0's MAC, and its first three bytes match
the MAC Proxmox now has cached for 192.168.10.1. VLAN interfaces use their parent's MAC, so the gateway is the t655.

**The traceroute that "failed" and proved the firewall:** `traceroute -n 192.168.10.2` from Mint showed
`192.168.8.1` and then `* * *` to hop 30. Linux traceroute sends **UDP** probes by default, and Mint is only
allowed TCP 22/8006 + ICMP echo. OPNsense dropped every probe silently. `traceroute -I` (ICMP echo, allowed by
WAN 6) is the way to see the path: `.8.1 → .8.195 → .10.2`.

### 3.6 Locked in
- SG300: `copy running-config startup-config` (otherwise a switch reboot undoes the trunk change and the SVI removal)
- OPNsense: `post-phase3` config backup downloaded (outside the repo)

---

## 4. State at end of session

| Device | Change |
|---|---|
| Beryl AX | **4.11.0**. Routes: `192.168.10.0/24 via 192.168.8.195` only. snmpd reinstalled, config kept |
| SG300 | gi5 trunk: VLAN 1 untagged + **10** + 20 tagged. **No VLAN 10 SVI**: the switch now routes nothing but its own management. Saved |
| t655-fw | Gateway for **LAB (VLAN 20) and SERVERS (VLAN 10)**. 4 SERVERS rules, 8 WAN rules, 4 new aliases |
| Servers | Unchanged inside: same IPs, gateway 192.168.10.1, same DNS |

```
Internet ─ WhiteSky ─ Beryl (.8.1) ─ SG300 gi1
                                      ├ gi4  Mint            VLAN 1
                                      ├ gi6  t640-mon        VLAN 1
                                      ├ gi5  t655 OPNsense   trunk: 1 (WAN .195) · 10 (SERVERS .10.1) · 20 (LAB .20.1)
                                      ├ gi2  pve-node1 + VMs VLAN 10 ─► gateway = the firewall
                                      └ gi3  testkit         VLAN 20 ─► gateway = the firewall
```

---

## 5. Lessons learned
- **uci indexes shift after a delete.** Delete once, then re-read.
- **"Keep Settings" ≠ keep packages.** GL.iNet upgrades keep `/etc/config` but drop add-ons like snmpd. Record what you installed before upgrading.
- **Check the router's own upgrade page, not just the vendor's status page.** The status page lagged the real release.
- **`Last login` is the previous login.** Read it as history, then name every address you find.
- **Take over the gateway IP, don't renumber the servers.** Same IP, new owner = zero changes inside the VMs.
- **Prepare disabled.** Building the interface and rules with the interface off kept the outage to the three cutover commands. Two live owners of one IP = a conflict.
- **Read the rule list, not your memory of the form.** Three of eight new rules were saved wrong (`any` / `UDP` instead of ICMP). One of them would have opened the firewall's GUI to the servers.
- **First match wins:** a too-broad Pass above a Block quietly cancels the Block.
- **A failed UDP traceroute through a strict firewall is a success.** Use `-I` when ICMP is the only thing allowed.
- **EUI-64 leaks the MAC into the IPv6 link-local address.** Useful for proof, so treat it as identifying too.

## 6. Next
1. Confirm the Beryl DHCP reservation for the Windows laptop (192.168.8.166) so `ADMINS` stays accurate.
2. Watch OPNsense Live View / `/var/log/remote/192.168.8.195.log` for a day: any legitimate block → a deliberate rule.
3. Suricata on **SERVERS** too: the public-facing apps are the likeliest way in.
4. Management VLAN (Phase 3c): admin GUIs reachable only from it.
5. Carried from SESSION-56: memtest86+ pass · VM 100 back in the nightly vzdump job · Hermes guest-agent freeze error (`/srv/storage`) · t640-mon RAM 4 → 32 GB · Wazuh (Phase 4).
