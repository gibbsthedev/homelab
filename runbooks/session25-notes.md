# Session 25 — Proving the Routing Works (and Three Ways It Lied First)

Date: 2026-09-07

Session 24 built inter-VLAN routing but never tested it from an actual client.
This session did — and hit three separate failures on the way, each of which
looked like a routing problem and wasn't. The routing itself turned out to
have been correct the whole time.

---

## PART 1 — THE SWITCH HAS NO DHCP SERVER

Session 24 ended with `ip dhcp pool` returning unrecognized-command output,
which read like a syntax problem on 2011-era firmware. Rather than trying
variations, ask the command tree what actually exists:

```
sg300-apt(config)#ip dhcp ?
  information     Dhcp information configuration commands
  relay           Configure DHCP relay
  tftp-server     IP DHCP client tftp server configuration
```

**No `pool`. No `server`.** This firmware (SW 1.1.2.0, 12-Nov-2011) has no
DHCP server at all. It was never a syntax problem — the feature does not exist
on this image. DHCP *server* support arrived in later SG300 firmware.

Options considered:
- **DHCP relay** (`ip helper-address`) → but the Beryl would then need pools
  for subnets it has no interface in. More config on firmware that had already
  failed once.
- **Static addressing** → zero new config, unblocks everything immediately.
- **A DHCP server on pve-node1 later** → proper long-term answer, but
  chicken-and-egg: Proxmox has to move to VLAN 10 first.

Chose static now. **DHCP was never the goal** — the goal is proving routing
and then writing ACL policy on top of it. Static addressing tests routing just
as well and removes a failure mode. Proxmox is statically addressed anyway, so
the eventual VLAN 10 move doesn't depend on DHCP either.

**Lesson: check whether a feature exists before debugging its syntax.**
`<command> ?` answers that in one line.

---

## PART 2 — NETWORKMANAGER SILENTLY UNDOES `ip addr add`

Set up the VLAN 20 test the obvious way:
```bash
sudo ip addr flush dev enp1s0
sudo ip addr add 192.168.20.100/24 dev enp1s0
sudo ip route add default via 192.168.20.1
```
`ip route` confirmed the address and default route were present.

Then every single ping failed:
```
ping -c 3 192.168.20.1
ping: connect: Network is unreachable
```

**A directly-connected gateway on the same subnet returning "Network is
unreachable" is the tell.** That ping needs no default route, no switch
routing, no Beryl — just ARP. If *that* fails, the route table isn't what you
think it is.

```
ip -br address
enp1s0   UP   fe80::3fbb:10ca:65b:bfd1/64      <-- IPv6 link-local ONLY
```

The IPv4 address was gone. **NetworkManager reasserted control of the
interface and wiped the manual config**, seconds after it was applied. Manual
`ip addr add` is not persistent against NM and can vanish without warning.

The fix is to configure through NM instead of behind its back:
```bash
sudo nmcli con mod "Wired connection 1" \
  ipv4.method manual \
  ipv4.addresses 192.168.20.100/24 \
  ipv4.gateway 192.168.20.1 \
  ipv4.dns 192.168.8.1
sudo nmcli con up "Wired connection 1"
```

**Lesson: on a NetworkManager system, `ip addr add` is a temporary override,
not a configuration.** Good to learn on a test laptop rather than on something
that matters.

---

## PART 3 — ROUTING PROVEN, AND THE TTL IS THE PROOF

With the address holding, the ladder from a VLAN 20 client (WiFi off, cable in
gi3):

```
ping 192.168.20.1     ttl=64   0% loss    own gateway (SVI)
ping 192.168.8.165    ttl=64   0% loss    switch management
ping 192.168.8.1      ttl=63   0% loss    Beryl
ping 1.1.1.1          ttl=57   0% loss    internet
```

**Read the TTLs, not just the successes.** 64 → 64 → **63** → 57. The drop to
63 at the Beryl is the packet being decremented as it crosses the SG300's
routing boundary. That single digit is the difference between "these hosts are
on one flat network" and "the switch routed my traffic between two subnets."

The first two stay at 64 because both the VLAN 20 SVI and the VLAN 1 SVI are
interfaces *on the switch itself* — the packet is delivered locally, not
forwarded.

Path proven:
```
Mint (VLAN 20, .20.100)
  → SG300 SVI .20.1
  → SG300 routes VLAN 20 → VLAN 1
  → Beryl .8.1   (session 24's return route doing its job)
  → NAT → building WiFi → internet
```

Without the Beryl static routes, the first three would pass and `1.1.1.1`
would fail. They were already in place, so the asymmetric-routing failure
never appeared.

---

## PART 4 — PING WORKS, DNS DOESN'T

```
ping -c 2 google.com
ping: google.com: Name or service not known
```

Routing was fine (`1.1.1.1` worked). Only name resolution failed — a clean,
isolated layer.

Mint's config was correct:
```
resolvectl status
Current DNS Server: 192.168.8.1
```

So query the resolver directly and bypass everything local:
```
nslookup google.com 192.168.8.1
;; communications error to 192.168.8.1#53: timed out
;; no servers could be reached
```

**Same host answers ICMP but not port 53.** That rules out routing entirely —
the packets are arriving. Confirmed on the Beryl:

```bash
uci show dhcp.@dnsmasq[0] | grep -E 'interface|localservice'
dhcp.cfg01411c.localservice='1'

logread | grep -i dnsmasq | tail
dnsmasq[15779]: Ignoring query from non-local network
dnsmasq[15780]: Ignoring query from non-local network
...
```

dnsmasq **logged the decision explicitly**. `localservice=1` means it only
answers clients on directly-attached subnets. Queries from `192.168.20.100`
arrive from a subnet the Beryl reaches by *static route*, not by interface, so
dnsmasq classifies them as non-local and drops them.

The firewall was innocent — `firewall.@zone[0].input='ACCEPT'` on the lan zone.

### Two fixes, and why the narrow one wins

- **Blunt:** `uci set dhcp.@dnsmasq[0].localservice='0'`. Works instantly. But
  it doesn't scope anything — it removes the check entirely, leaving only the
  WAN firewall between this resolver and the internet. Open resolvers get
  scanned for and abused in DNS amplification attacks. One layer instead of
  two, to solve a local problem.
- **Narrow:** point VLAN 20 clients at `1.1.1.1` instead. Zero change to the
  router's security posture. Loses `.lan` name resolution on that VLAN — which
  for an isolated lab segment is arguably correct. A quarantined segment
  having no view of the internal namespace is a feature.

Both were tried; `localservice` was set back to `1` and DNS still resolved,
which proves the client-side fix is what's carrying it.

**A trap worth naming:** it's tempting to add a secondary IP inside
`192.168.20.0/24` on `br-lan` so dnsmasq treats the subnet as local, keeping
`localservice=1`. **Don't.** The Beryl would then consider that subnet
directly connected, and the connected route would beat the static route via
`192.168.8.165` — silently breaking the return path. Elegant-looking, quietly
destructive.

**Lesson: don't disable a security control globally to solve a local problem.**
Ask what the control is for, then make the narrowest change that meets the
requirement.

**Long term:** DNS is a service and belongs on a server, not the edge router.
A resolver on pve-node1 would give per-VLAN policy, local names, and — for the
SOC track — **DNS query logging**, one of the highest-value telemetry sources
there is.

---

## PART 5 — CLEANUP

Test scaffolding removed; real infrastructure kept.

Undone:
```bash
nmcli radio wifi on
sudo nmcli con mod "Wired connection 1" ipv4.method auto \
  ipv4.addresses "" ipv4.gateway "" ipv4.dns ""
sudo nmcli con up "Wired connection 1"
# cable moved back to a VLAN 1 port (gi4-gi10)
```

Kept permanently:
- Beryl static routes for `192.168.10.0/24` and `192.168.20.0/24`
- `localservice='1'` on the Beryl (reverted to the secure default)

Verified after cleanup — note both defaults present, wired winning on metric:
```
enp1s0   UP   192.168.8.142/24    proto dhcp   metric 100
wlp2s0   UP   192.168.8.186/24    proto dhcp   metric 600
ping 192.168.8.165 → ttl=64, 0% loss
```

That two-path state is exactly the trap from session 23 and from Part 2 of
session 24. **Turn WiFi off before any isolation test, or the result lies.**

---

## STATE AT END OF SESSION

Proven working:
- Inter-VLAN routing, VLAN 20 → VLAN 1 → Beryl → internet, TTL-verified
- DNS from VLAN 20 (via public resolver)
- Beryl return routes confirmed in the kernel table

Known limits:
- **No DHCP server on this switch firmware.** Static addressing on VLANs 10/20.
- No local name resolution on routed VLANs (by choice).

Not done:
- **ACLs.** VLAN 20 currently reaches VLAN 10, the switch management IP, the
  Beryl, and every VLAN 1 host including pve-node1 and both VMs. That is
  routing, not isolation. Isolation must now be written as explicit policy —
  this is the next session.
- pve-node1 still on VLAN 1
- Reboot durability test not run
- SG300 NTP still unconfigured
