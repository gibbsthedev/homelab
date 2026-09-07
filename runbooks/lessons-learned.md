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
