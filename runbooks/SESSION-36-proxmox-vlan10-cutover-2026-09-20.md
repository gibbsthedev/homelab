# SESSION 36 — Proxmox + VMs Cut Over to VLAN 10 — 2026-09-20

**Status: COMPLETE.** The Proxmox host and both production VMs moved from the
flat `192.168.8.0/24` LAN onto the SG300's VLAN 10 (`192.168.10.0/24`). All
services returned, public sites verified from outside, cross-VLAN management
works.

**Continuity:** executes the plan staged in
`SESSION-35-vlan10-migration-staging-2026-09-20.md`. The pre-cutover defects
(DNS via Tailscale, `/etc/hosts`, the `nico` typo, the blog automation address)
are documented there and not repeated here. This file covers the execution and
what it proved.

---

## Final addressing

| Machine | Old | New | Login |
|---|---|---|---|
| Proxmox `pve-node1` | 192.168.8.2 | **192.168.10.2** | root (console / SSH) |
| VM100 `docker-agents` (OpenClaw / Tuck) | 192.168.8.10 (DHCP) | **192.168.10.10** static | `ubuntu` |
| VM101 `hermes` | 192.168.8.206 (DHCP) | **192.168.10.206** static | `ec2-user` |

Gateway for all three: `192.168.10.1` (SG300 VLAN 10 SVI).
Proxmox web UI: **`https://192.168.10.2:8006`**

---

## Physical change

```
Before:  Beryl eth1 ─── Proxmox
After:   Beryl eth0 ─── SG300 gi1 (VLAN 1) … routed … SG300 gi2 (VLAN 10, access) ─── Proxmox
```

Cable replaced with a better one during the move; gi2 link light confirmed
before rebooting. **Beryl eth1 is now free** for the wired WAN.

---

## Cutover sequence (as executed)

```
qm config 100 | grep onboot; qm config 101 | grep onboot   → no output: neither autostarts
qm shutdown 100 && qm status 100                           → stopped
qm shutdown 101 && qm status 101                           → stopped
unplug Mint from gi2
move Proxmox cable: Beryl eth1 → SG300 gi2                 → gi2 link light on
reboot
```

With both VMs stopped, moving the cable while the host was still running was
safe: nothing was running that could be interrupted.

### Surprise: PXE boot

On reboot the M720q fell through to **PXE network boot** (IPv4, then IPv6,
MAC 98-FA-9B-4C-65-20) instead of booting the disk. Unrelated to the network
change: the firmware never reads the OS network config. Recovered by power
cycling and choosing the disk from the **F12 boot menu**. BIOS boot order still
needs fixing.

---

## Verification — host, from the physical console

| Test | Result | Proves |
|---|---|---|
| `ip -br addr show vmbr0` | UP, 192.168.10.2/24 | new config loaded |
| `ping -c 3 192.168.10.1` | 0% loss, ~1.3 ms | cable + gi2 + VLAN 10 |
| `ping -c 3 192.168.8.165` | 0% loss, ~1.4 ms | inter-VLAN routing |
| `ping -c 3 192.168.8.1` | 0% loss, ~0.8 ms, **ttl=63** | Beryl's static return route |
| `ping -c 3 1.1.1.1` | 0% loss, ~10.7 ms | internet as the **only** path |
| `getent hosts google.com` | resolved | DNS |

- **ttl=63** at the Beryl (vs 64 one hop away) is the switch's routing hop,
  visible in the output.
- The 1.1.1.1 ping **closes the open question** from SESSION-35: VLAN 10 had
  only been tested from Mint with WiFi up and `/32` routes, never from a host
  whose *sole* route was via `192.168.10.1`. Now proven.
- The console login banner showed `https://192.168.10.2:8006/`, confirming the
  `/etc/hosts` fix took.
- `getent hosts google.com` printed only IPv6 records. Normal: the lookup
  worked, and the host has no global IPv6, so programs connect over IPv4.

## Verification — VMs

- `ping 192.168.10.10` from the host → replies. VM100's static config worked,
  including the `allow-hotplug ens18` line flagged as a possible risk.
- Both VMs started by hand (`qm start 100`, `qm start 101`).
- Both agents answered over Telegram → proves outbound internet **and** DNS on
  each VM, by function.
- Both agents reported their new addresses.
- **\*.richgibbs.dev loaded from a phone on mobile data** → Cloudflare Tunnels
  reconnected; verified from outside the network, not via a LAN shortcut.
- SSH from Mint (192.168.8.142, VLAN 1) to VM100 (192.168.10.10, VLAN 10)
  works across VLANs. Fresh host-key prompt was expected (new address, same
  key), not a warning.

### False alarm: "SSH is broken"

SSH to VM100 appeared broken after the move. Cause: connecting to
`192.168.10.206` (Hermes) instead of `192.168.10.10` (VM100), and `ubuntu`
doesn't exist on Hermes. Hermes separately reported that it changed no SSH or
password settings; its "SSH works again" referred to its own key-based link to
VM100, i.e. the blog automation `ssh_host` fix from SESSION-35.

---

## Lessons

**Order changes by dependency, and do one at a time.** Proxmox had to move
first because it occupied the port the wired WAN needs. The WAN change is
deliberately a separate session so any failure has one cause.

**Test from the bottom of the stack up.** Own address → gateway → other VLAN →
upstream router → internet → DNS. Each ping adds exactly one new dependency, so
the first failure names the broken layer.

**Verify by function.** A Telegram reply and a site loading over mobile data
proved more about each VM than reading its config could.

**Boot-time surprises aren't always about the last change.** The PXE fallback
looked alarming right after a network change, but it was the firmware failing
to pick the disk. Read what the screen actually says before assuming cause.

**Know the address map before debugging.** The one "failure" was a wrong IP.

---

## Follow-ups

- [ ] **Autostart VMs:** `qm set 100 --onboot 1` and `qm set 101 --onboot 1`.
      Today a power outage leaves every service down until started by hand.
- [ ] **Fix M720q boot order** in BIOS (disk first, PXE off).
- [ ] Update SSH configs and bookmarks to the new addresses.
- [ ] Watch `dmesg -T | grep -i hang`: TSO offload is now actually disabled for
      the first time (the `nico` typo), which should help the e1000e hang.
- [ ] Remove stale `/etc/resolv.pre-tailscale-backup.conf` on both VMs.
- [ ] Delete the dead AWS node from both tailnets.
- [ ] VLAN 10 ACL (note: VLAN 1 → VLAN 10 is open today, which is what keeps
      laptop management working).
- [ ] SG300 NTP; reboot durability test (pre-existing).
- [ ] **Next: Beryl wired WAN** on the freed eth1, Multi-WAN failover with the
      WiFi repeater kept as backup.

Unrelated change in the same window: Hermes deployed a new blog-post recovery
mechanism (approved separately). If anything misbehaves soon, that's a second
variable.
