## 2026-07-15 - Flash Drive Patience
- Remember to wait until the flash drive finishes writing before pulling and use umount command to safely remove. 

## 2026-07-15 - Apartment network needs investigation
- M720q plugged into apartment ethernet wall jack. 
- Internet is SHARED WiFi
- Linux Mint Laptop Wifi: 100.110.148.128/26, gw 100.110.148.129

## 2026-07-16 - Foundation Complete
- GL.iNet Beryl AX connected to apt WiFi as uplink
- M720q reached at static IP: 192.168.8.2:8006
- Fixed subscription popup 
- Spent a lot of time fixing errors for apt upgrade, learned about 'cat <<' and 'EOF' multi-line is hard when you cannot copy paste
- Installed git on server

## 2026-07-17 -- Docker host built + agent migration started
- Created VM 'docker-agents' (VMID 100), Debian 13.6, on pve-node1. 
- Installed Docker CE successfully (hello-world verified).
- GOTCHA: Docker apt repo must be added BEFORE 'apt update', or packages show "no installation candidate." The long echo|tree command can fail on partial paste -- verify with 'cat docker.list'.
- Minimal Debian lacks common tools (sudo, rsync, curl) -- install as needed.
- User perms: 'usermod -aG docker rich' lets rich run docker w/o root. (but it takes effect on the next login). Enabling services needs root/sudo, not plain user. 
- Migration approach: AWS is cloud (public), home VM is behind NAT on apartment WiFi. AWS can't reach IN, but VM can reach OUT -- so I PULL data from the VM via rsync, not push from AWS.
- Transferred ~/.openclaw (6.3GB) from AWS (54.197.26.209) to VM via rsync over SSH, using the AWS .pem key (copied to VM, chmod 600).

## 2026-09-06 -- SG300 Layer 3 migration; the Beryl trunk is impossible
- Beryl AX (GL-MT3000) CANNOT do 802.1Q bridge VLAN filtering. LuCI draws the
  table, kernel can't deliver it: kernel 5.4.211, OpenWrt 21.02, target
  mediatek/mt7981, and `bridge` isn't even installed. Save & Apply rolls back
  after 90s every time. Don't retry this without a firmware upgrade first.
- LESSON: a UI offering a control is not proof the platform supports it.
  Get evidence (`ubus call system board`) before blaming your own config.
- LuCI rollback is a safety net, not a failure. Click **Revert changes** to
  discard staged config -- **Dismiss** only hides the banner and leaves it staged.
- GOTCHA: spent a whole diagnosis believing I was on the wired fallback while
  `enp1s0` was DOWN with no IP -- the session was on WiFi the entire time.
  ALWAYS prove the path with `ip route get <target>`, never assume.
- An unconfigured SG300 port reports `Port Mode: Trunk` by default. That's not
  a trunk -- read the VLAN membership table, not the one-word mode label.
- `set system mode router` is typed at the `#` prompt, NOT in `configure terminal`.
- A mode change WIPES startup-config and reboots -- in BOTH directions. L3 -> L2
  later is a second rebuild, not an undo.
- After the wipe SSH was OFF (port 22 refused). Web GUI is still on by default:
  recovered via http://192.168.8.165 and re-enabled `ip ssh server`.
- In Router mode, `ip address` on a VLAN ADDS an interface (silently). In L2
  mode the same command MOVED management = lockout. Same command, opposite
  meaning, depending on mode.
- Static default route failed with `No such instance exists` -- not a syntax
  error. DHCP had already installed one (`D 0.0.0.0/0` in `show ip route`) and
  the slot was occupied. Check the route table before assuming bad syntax.
- Empty VLANs don't appear as connected routes until something is plugged in.
- Beryl needs static return routes or replies never come back:
  `192.168.10.0/24 via 192.168.8.165`, same for `.20.0/24`. Verify with
  `ip route | grep 192.168` on the Beryl -- a LuCI form entry is not proof.

## 2026-09-07 -- Routing proven; three failures that weren't routing
- SG300 firmware 1.1.2.0 (2011) has NO DHCP server. `ip dhcp ?` offers only
  information/relay/tftp-server. Earlier "syntax errors" were the feature not
  existing. Check `<command> ?` for existence before debugging syntax.
- Went with static addressing on VLAN 10/20. DHCP was never the goal -- routing
  and ACL policy are. Proxmox is static anyway.
- BIG GOTCHA: NetworkManager silently reverts `ip addr add`. Set the address,
  confirmed it in `ip route`, then every ping failed with "Network is
  unreachable" -- including the directly-connected gateway. The IPv4 address
  had vanished; only IPv6 link-local remained. Use `nmcli con mod` instead.
- TELL: "Network is unreachable" pinging a gateway on your OWN subnet means the
  route table isn't what you think. That ping needs nothing but ARP.
- Inter-VLAN routing VERIFIED end to end from VLAN 20. Read the TTLs:
  .20.1 ttl=64, .8.165 ttl=64, .8.1 ttl=63, 1.1.1.1 ttl=57. The drop to 63 at
  the Beryl IS the proof the switch routed -- one decrement per router hop.
- DNS failed while ping worked. dnsmasq on the Beryl logs it plainly:
  "Ignoring query from non-local network". Cause: `localservice='1'` means it
  only answers directly-attached subnets, and VLAN 20 reaches it by static
  route. Firewall was innocent (lan zone input=ACCEPT).
- Fixed NARROW (point the client at 1.1.1.1) instead of BLUNT
  (`localservice='0'`, which turns the router into an open resolver and leaves
  only the WAN firewall protecting it). Don't disable a security control
  globally to solve a local problem.
- TRAP avoided: adding a secondary br-lan IP inside 192.168.20.0/24 would make
  dnsmasq treat it as local -- but the resulting connected route would beat the
  static route via .8.165 and silently break the return path.
- Two default routes (wired metric 100, WiFi metric 600) is the same trap as
  every isolation test so far. WiFi off before testing, or the result lies.
- STILL OPEN: VLAN 20 can currently reach VLAN 10, .8.165, the Beryl, and all
  VLAN 1 hosts incl. pve-node1 and both VMs. VLANs are broadcast separation,
  NOT a security boundary. Isolation has to be written as ACL policy next.

## 2026-09 -- WhiteSky wall jacks dead; Juniper AP's spare port is a live uplink
- `cat /sys/class/net/<if>/carrier` is the fastest Layer 1 test -- 0/1, no
  interpretation needed. Answers "is there a signal at all" before DHCP, IP,
  or DNS can even be relevant. Check this before anything above it.
- Isolate hardware faults by holding everything constant except the one thing
  under test: same laptop, same cable, against three different far ends
  (three dead wall jacks, one known-good switch, one unknown AP port). The
  laptop/cable were cleared entirely because they returned carrier=1 against
  two of the three destinations.
- Three independent wall-jack drops failing at once, after previously
  working, points upstream (patch panel / managed switch) -- not three
  simultaneous independent cable failures.
- A building AP can carry a second, independent wired uplink port alongside
  its own WiFi/PoE feed. Worth checking on any managed-AP hardware before
  assuming wired access requires the property's dedicated jacks.
- 100.64.0.0/10 addresses are CGNAT space -- expected on shared/managed
  building internet, not a misconfiguration.
- Still open: whether that AP port allows multiple client MACs behind it.
  Test with ONE additional device before assuming it'll support the whole
  homelab switch.

## 2026-09-13 -- Juniper spare port is single-MAC locked; Beryl migration still viable
- Confirmed by direct test: the Juniper AP's spare Ethernet port serves DHCP
  to any MAC but only forwards traffic past ARP for the FIRST MAC it learned.
  A second device (Proxmox, after Mint had already used the port) got a
  lease and then hit "Destination Host Unreachable" pinging its own gateway
  -- reseating both cable ends changed nothing. Classic sticky-MAC / port
  security behavior on a managed building switch.
- This does NOT block using the port as Beryl's WAN: Beryl already presents
  ONE MAC to the outside via NAT, same as it does today over WiFi. The lock
  only matters for devices plugged in raw, not behind a NAT router.
- Real throughput case for doing it anyway, independent of the above:
  FBI (via Beryl WiFi-repeater WAN) = 52 Mbps down / 94 up.
  WhiteSky-Prose direct = 203 Mbps down / 324 up. ~4x/~3.4x gap, same
  latency -- confirms the bottleneck is Beryl's WiFi-repeater WAN hop
  specifically, not WhiteSky throttling the apartment.
- GOTCHA: interrupted `dhclient -r && dhclient` with Ctrl-C mid-request --
  it fell back to a stale cached lease instead of failing cleanly. Let DHCP
  commands finish or time out; don't interrupt mid-negotiation.
- GOTCHA: `curl --interface <if>` pins the socket to that NIC but grants NO
  route. If that interface has no default route of its own (traffic-shifted
  scenario, only the OLD interface/bridge still holds `default via ...`),
  every request fails even though the interface and its own IP are fine.
  Needed an explicit `ip route add <target> via <gw> dev <if>` per target.
  Same failure mode as every NetworkManager stale-route trap this session,
  just on Proxmox instead of Mint.
- `speedtest-cli` (python/Ookla classic tool) returned a nonsense multi-day
  ping and ~1Mbit reading -- broken tool, not a broken network. Prefer a
  direct `curl -o /dev/null -w ...` against a known file host when this
  tool gives an implausible number.
- `apt install` hanging at "0% Working" with no error, on a host whose only
  route to the internet is dead, looks like a package-manager problem and
  is actually the routing problem one layer down. Check `ip route` before
  troubleshooting apt itself.

## 2026-09-20 -- ACLs on the SG300; wall jacks were a dead outlet
- CONCEPT: an ACL is read top-to-bottom, FIRST MATCH WINS, then it STOPS.
  Specific denies must go ABOVE the general permit. `permit ip any any` on
  line 1 makes every rule below it dead code that looks like security.
- CONCEPT: every Cisco ACL has an INVISIBLE `deny all` at the bottom. An
  empty ACL applied to an interface blocks 100% of traffic. Build the rules
  first, attach the ACL last. Also why the final rule must be an explicit
  `permit ip any any` or internet traffic dies too.
- CONCEPT: these ACLs are STATELESS -- no memory that you sent a request, so
  return traffic is judged cold against the same rules. AWS Network ACLs are
  stateless the same way; AWS Security Groups are STATEFUL and auto-allow
  return traffic. Real interview question, real production footgun.
- BIG ONE: VLANs are broadcast separation, NOT a security boundary. Before
  the L3 migration VLAN 20 was isolated only because nothing routed between
  segments (isolation by ABSENCE). Building a router removed that. Isolation
  then has to be written as explicit policy or it does not exist.
- Wildcard masks are the INVERSE of subnet masks: 1=wildcard(ignore),
  0=hit(must match). Whole /24 = 0.0.0.255 (not 255.255.255.0). Single host
  = 0.0.0.0. The switch helpfully rewrites `x.x.x.x 0.0.0.0` as `host x.x.x.x`.
- Use `ip` not `tcp` in the rule -- `ip` covers ICMP/TCP/UDP/everything.
  A tcp-only rule blocks web but leaves ping working = half a policy.
- Use EXTENDED not standard ACLs: standard matches source only, extended
  matches source AND destination. Policy about where traffic is GOING needs
  extended.
- FIRMWARE DEAD-END #3 (after `write memory` and `ip dhcp pool`):
  `service-acl` does NOT exist on `interface vlan X` on this image. The full
  VLAN-interface command list is just bridge/do/dot1x/end/exit/help/ip/
  ipv6/name/no/sntp. It DOES exist on physical ports. Bound to gi3 instead
  (the VLAN 20 access port) -- same effect, since all VLAN 20 traffic enters
  there. Only `input` direction is offered.
- METHOD: when a command is "Unrecognized," do NOT guess a second syntax.
  Run bare `?` in that context and enumerate what actually exists. Check
  other contexts too (VLAN interface vs physical port had different
  capabilities). Same move that cracked `set system mode router` and DHCP.
- VERIFY BY FUNCTION, not by config. Five destinations, three outcomes:
  own gateway + internet PASS, VLAN 10 + switch mgmt + VLAN 1 FAIL. If all
  five had worked the ACL wasn't enforced; if all five failed it was too
  broad. The SPLIT is the proof.
- ACL denies are SILENT -- 100% packet loss, no error message. Different
  from "Destination Host Unreachable" (that's ARP failing, a lower layer).
  Silence is deliberate: an explicit rejection confirms the target exists.
- Best proof was accidental: same ping to .8.165 = 100% loss from VLAN 20,
  0% loss from VLAN 1. Scope proven by contrast.
- Switching the laptop back to DHCP while still plugged into gi3 fails with
  "IP configuration could not be reserved" -- correct, there IS no DHCP
  server on VLAN 20. Move the cable to a VLAN 1 port first.

### Wall jacks: it was a dead outlet
- Found the structured media cabinet the original investigation concluded
  didn't exist. Wavenet 8-port patch panel, ports 1-4 punched and patched
  to a small unmanaged switch below it.
- The switch had NO LIGHTS. Swapped outlets -> lights on -> all three
  previously-dead wall jacks immediately gave carrier 1, a real WhiteSky
  CGNAT lease (100.110.30.7/22), and internet at ttl=59 / ~6.5ms.
- One dead outlet took out every jack in the apartment at once, which is
  exactly why all three failed simultaneously after previously working.
- The original diagnosis correctly predicted "common upstream failure" but
  assumed something sophisticated (reconfigured managed switch, unpatched
  ports). When the logical layer has nothing left to explain, GO LOOK AT
  THE PHYSICAL ONE.
- 6.5ms wired vs 13-14ms on either WiFi path. WhiteSky support cancelled.

## 2026-09-20 (evening) -- Staging the VLAN 10 migration; four silent config defects
- THEME: a config line accepted without error is NOT a config line that takes
  effect. All four defects tonight were that shape, and none would have been
  visible after the cutover -- they'd have surfaced as "sites are down" while
  ping worked fine.
- Tailscale OWNS /etc/resolv.conf on both VMs (MagicDNS, marked DO NOT EDIT).
  All queries go to 100.100.100.100 and forward upstream -- and that upstream
  was 192.168.8.1 (the Beryl) on both boxes. The `dns-nameservers` and `dns=`
  lines in the OS configs are decoration.
- Only surfaces under `sudo tailscale dns status`. Unprivileged output says
  "Access denied: dns-osconfig dump access denied" -- a non-root check shows
  nothing wrong.
- TWO SEPARATE TAILNETS exist: rgibbs21.rg@gmail.com (tail523202.ts.net, holds
  hermes) and richgibbs-prog@github (taila006d9.ts.net, holds docker-agents).
  A DNS change in one admin console does NOT affect the other VM. Both needed
  changing separately.
- Setting a global nameserver in Tailscale is ADVISORY until you also turn ON
  "Override DNS servers". With override off the client keeps preferring the
  captured local value. Force a re-fetch with
  `sudo tailscale set --accept-dns=true`.
- The Beryl's dnsmasq localservice='1' bites again -- same finding as session
  25, different victim. Any routed VLAN host pointed at 192.168.8.1 for DNS
  gets "Ignoring query from non-local network".
- Proxmox resolves its OWN node name via /etc/hosts and pveproxy binds based
  on it. A stale entry after an IP change makes the node go grey or the UI
  hang -- symptoms that look nothing like a hosts-file problem. This edit is
  LIVE immediately, unlike /etc/network/interfaces.
- `post-up /sbin/ethtool -K nico tso off` -- letter o, not zero. Erroring at
  every boot for months, so TSO was never actually disabled (GSO/GRO were).
  Offload-disabling on Realtek NICs is the classic fix for corrupt transfers
  under load; it had been running at two-thirds strength.
- DO NOT tag VLAN 10 on the VM's NIC in Proxmox. gi2 is an ACCESS port -- the
  switch places untagged frames into VLAN 10 itself. Tagging in Proxmox would
  double-tag into an access port and nothing would work. (VM100's self-report
  recommended this; it was guessing at an architecture it can't see.)
- `nmcli` reports the IN-MEMORY profile loaded at boot, not what's on disk. A
  staged static config being invisible to nmcli is CORRECT, not a failure.
- NetworkManager refuses to load a .nmconnection file unless it is mode 600.
- `qm guest exec <vmid>` runs commands as root inside a guest via the QEMU
  guest agent -- no network, no password. This is the real recovery path when
  a guest boots with broken networking, better than Tailscale because it
  doesn't depend on the network at all.
- `qm terminal 100` fails when no serial0 device is defined on the VM.
- Cloudflare Tunnel is an OUTBOUND connection from the VM. Cloudflare never
  dials in, so the VM's LAN address is irrelevant to it. What matters for site
  uptime is only how long the VM has no route and no DNS.
- STILL UNTESTED going into cutover: VLAN 10 has only ever been tested by a
  host with a second path available (Mint with WiFi up and /32 routes).
  "The path works" and "the path works as the SOLE path" are different claims.

## 2026-09-20 — VLAN 10 cutover (SESSION-36)

- A clean ping ladder names the broken layer: own IP → gateway → other VLAN →
  upstream router → internet → `getent hosts`. Each step adds one dependency.
- ttl=63 instead of 64 on a ping across VLANs is the switch's routing hop.
  Handy proof that traffic is being routed, not switched.
- The M720q can fall through to PXE network boot on reboot. That's the
  firmware not picking the disk, not the OS network config. F12 boot menu
  recovers it; fix the boot order in BIOS.
- Neither VM had `onboot: 1`. Check `qm config <id> | grep onboot` before any
  host reboot so a stopped VM isn't mistaken for a failure.
- After re-addressing, a fresh SSH host-key prompt is normal (new address,
  same key). "REMOTE HOST IDENTIFICATION HAS CHANGED" is not.
- Keep the address map handy after a migration: the only "outage" tonight was
  SSHing to the wrong VM.
