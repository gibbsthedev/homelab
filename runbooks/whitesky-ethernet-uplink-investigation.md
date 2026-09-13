# WhiteSky Wired Ethernet — Wall Jacks Dead, Juniper AP Found as Working Uplink

Date: 2026-09 (diagnosed in a separate troubleshooting session, written up here
for continuity)

All three built-in Ethernet wall jacks in the apartment stopped working. They
had worked before. Wi-Fi was unaffected. This is the diagnosis and the
discovery that came out of it: a second, unused Ethernet port on the
building's Juniper Mist access point is a live, working WhiteSky uplink.

---

## Symptom

Three separate wall jacks, tested with the same laptop and cable:
```
cat /sys/class/net/enp1s0/carrier
0
```
`carrier` is the Layer 1 test — it answers one question only: *is there an
electrical link pulse from the far end, at all.* `0` means no signal is
arriving, before DHCP, before IP, before anything software can go wrong.
Getting `0` from three independent jacks, with the same NIC and cable that
returned `1` elsewhere, points at something the jacks share — not the jacks
themselves, and not the laptop.

## Ruling out the laptop first

Before blaming apartment wiring, the same NIC and cable were tested against
two known-good sources:
```
Mint -> own switch        carrier = 1
Mint -> Juniper AP port   carrier = 1
Mint -> wall jack #1/2/3  carrier = 0  (all three)
```
**Same hardware, different result depending on the far end.** That's the
structure of a good hardware test — hold the variable you're unsure about
constant, change only the thing you're testing. It rules the laptop and cable
out completely and pushes the fault upstream of the jacks: the in-wall run,
the patch panel, or a managed switch port at the far end.

## The discovery: the Juniper AP has a second, live port

The apartment's ceiling-mounted Juniper Mist access point has two RJ45 ports.
One (red cable) is its own uplink/PoE feed — left alone. The second was
empty. Plugging directly into it:
```
cat /sys/class/net/enp1s0/carrier
1

sudo ethtool enp1s0 | grep -E 'Speed|Duplex|Link detected'
Speed: 100Mb/s
Duplex: Full
Link detected: yes
```
100 Mbps is the laptop's own Realtek NIC ceiling (Fast Ethernet only), not
necessarily a limit of the Juniper port.

Full stack verified working, WiFi disabled to prove nothing was riding the
wireless path:
```
nmcli radio wifi off
ip addr show enp1s0
inet 100.110.18.248/22

ping -c 4 100.110.16.1     # gateway       — 0% loss
ping -c 4 1.1.1.1          # internet, IP  — 0% loss
ping -c 4 google.com       # DNS + internet — 0% loss
```
`100.110.0.0/10` is CGNAT space — WhiteSky is doing carrier-grade NAT behind
the scenes, same as an ISP would for a whole apartment block. Not a
misconfiguration, just how a shared building network is provisioned.

**Conclusion: WhiteSky wired internet is fully available. The fault is
specific to the three wall-jack drops** — most likely an unpatched port, a
replaced/reconfigured managed switch, or a changed provisioning rule upstream
of the jacks. Not something fixable from inside the apartment.

## What this rules out, so it doesn't get re-litigated

- Not the Linux NIC or driver (`r8169`/RTL8105e) — confirmed working elsewhere
- Not the patch cable
- Not DNS, DHCP, or NetworkManager config
- Not Wi-Fi interference — WiFi was off for the successful tests

## Open question, not yet tested

Whether WhiteSky's provisioning allows **multiple MAC addresses** behind that
one Juniper port, or locks it to a single client:

- **If multiple clients work:** the Juniper's second port becomes a real wired
  uplink for the whole homelab switch — a meaningfully better path than the
  Beryl repeating WhiteSky over WiFi as its WAN.
- **If it's locked to one MAC:** the Beryl (or another router) sits directly
  behind that port as the single visible client, with everything else routed
  behind it as today.

Test plan, in order — don't skip straight to the full topology:
1. Juniper port → laptop directly (already done, passed)
2. Juniper port → homelab switch → laptop on one port (confirms the switch
   and the longer cable run don't introduce a fault)
3. Juniper port → switch → **laptop and Proxmox simultaneously** — this is
   the actual test of the open question above

If step 3 works for both, this changes the network topology (which device
sits at the edge). That is its own decision, for its own session — not an
automatic follow-on to buying a longer cable.

## Cable needed

25-30 ft run from the ceiling AP down to where the switch/laptop sits.
- **Cat6**, not Cat6a — plenty for gigabit at this length, and everything
  downstream (Beryl, switch, Proxmox) is gigabit even though this laptop's
  NIC caps at 100 Mbps
- **Pure copper (24AWG), not CCA (copper-clad aluminum)** — CCA is
  out-of-spec and common in cheap listings
- **Round**, not flat — better shielding for a ceiling run
- Buy 30 ft even if the measured run is closer to 20-25, for slack around
  corners and fixtures
- Route with removable clips or adhesive raceway — no staples into
  drywall/ceiling in a rental

---

## Update — Cable run and live test (2026-09-13)

Cable arrived, routed to where Proxmox sits. Rather than the full switch
test from the original plan, went straight to the sharper version of step 3:
swap Proxmox directly onto the Juniper's spare port, one device at a time,
since the switch adds nothing to the specific question being asked (does
this port serve more than one MAC).

### Baseline, before touching anything

Real speedtest numbers, not the port test — these answer "is wired worth it
at all," independently of the multi-client question:

| Path | Download | Upload | Ping |
|---|---|---|---|
| FBI (via Beryl, WiFi-repeated WAN) | 52.36 Mbps | 94.37 Mbps | 14ms |
| WhiteSky-Prose (phone, direct) | 203.47 Mbps | 323.93 Mbps | 13ms |

Roughly 4x download, 3.4x upload, near-identical latency. Confirms the
bottleneck is specifically Beryl's WiFi-repeater WAN hop, not WhiteSky
rationing the apartment's bandwidth — if it were the latter, the direct test
would be capped too, and it isn't. This number stands on its own regardless
of what the port test below found.

### The port test

Swapped Proxmox's cable from Beryl onto the Juniper's spare port.

```
nic0 (physical NIC): DHCP lease obtained fine — 100.110.18.246/22
ping -c 3 100.110.16.1  (the gateway itself)
  From 100.110.18.246 icmp_seq=1 Destination Host Unreachable
  From 100.110.18.246 icmp_seq=2 Destination Host Unreachable
  From 100.110.18.246 icmp_seq=3 Destination Host Unreachable
```

DHCP succeeded but the gateway was unreachable at ARP — a layer below
routing, below DNS, below anything else tried. Reseated both cable ends
(Juniper side and Proxmox side) and retried: identical result.

**Conclusion: this Juniper port locks to the first MAC it sees.** Mint
worked on it cleanly in the original investigation. Proxmox, a different
MAC, got a lease (DHCP is often handled separately/earlier) but was denied
at Layer 2 for everything else — the signature of sticky-MAC / port security
on the building's managed switch, not a fault in Proxmox, the cable, or the
new run.

**This closes the open question from the original investigation:** the port
does NOT serve multiple client devices. Confirmed, not assumed.

### Why this doesn't kill the plan

The port lock only matters if multiple *raw* devices sit behind it directly.
Beryl already presents exactly **one** MAC address to the outside no matter
how many devices are behind it — that's what NAT does, and it's already
true today over the WiFi-repeater WAN. Swapping Beryl's WAN from WiFi to
this Juniper port doesn't change that; Beryl becomes the single client the
port sees, same as it already is. The switch, Proxmox, and every VM stay
invisible to WhiteSky's provisioning either way.

**Decision: proceed with the Beryl WAN migration**, as its own dedicated
session — same discipline as the SG300 mode change: rollback path planned
before touching anything (keep the WiFi-repeater WAN configured as a
fallback rather than deleting it), verify the freed port assignment first,
confirm with a wired-vs-current speedtest once cut over.

### Debugging detours worth remembering (none were the actual problem)

- **`dhclient -r nic0 && dhclient nic0`, Ctrl-C'd mid-request** — killed
  before it got a fresh answer, fell back to a stale cached lease
  (`192.168.8.146`) instead of erroring cleanly. Let DHCP commands finish or
  time out on their own; interrupting mid-negotiation leaves stale state
  that looks like a new problem.
- **`vmbr0` kept its own default route (`via 192.168.8.1`) the whole time**,
  even after the physical cable moved to Juniper. `nic0` only had a route to
  its own /22 — no default route of its own. `curl --interface nic0` pins
  the *socket* to that NIC but does not grant it a route; without an
  explicit `ip route add <target> via <gateway> dev nic0`, anything without
  a matching connected route fails even though the interface itself is
  correctly configured. This is the same class of issue as the earlier
  NetworkManager/stale-route traps, just on Proxmox instead of Mint.
- **`speedtest-cli` (the classic Python tool) is unreliable** — returned a
  nonsensical multi-day ping and ~1 Mbit/s reading that had nothing to do
  with the actual link. Treat a wildly-wrong result from this specific tool
  as a broken tool, not a broken network, and fall back to a direct `curl`
  download test against a known file host instead of chasing it.
- **`apt install` hangs silently ("0% Working") rather than erroring** when
  the host's only route to the internet is dead. Looks like a package-
  manager problem; is actually the routing problem above, one layer down.

## Cable needed — SATISFIED

Cable purchased and run (Cable Matters Cat6, 30ft, 24AWG pure copper,
snagless). Original spec below kept for reference on future runs.
