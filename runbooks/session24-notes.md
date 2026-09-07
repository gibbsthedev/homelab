# Session 24 — Abandoning the Trunk, Migrating the SG300 to Layer 3

Date: 2026-09-06

Session 23 left one obvious next step: build an 802.1Q trunk on gi1 so the
Beryl could serve DHCP for VLANs 10 and 20. That step turned out to be
impossible on the current hardware, and finding out *why* — with evidence
rather than guessing — was most of this session. The answer changed the whole
design: instead of the router handling VLANs, the switch became the router.

---

## PART 1 — THE TRUNK THAT COULDN'T EXIST

### gi1 was never actually a trunk

`show interfaces switchport gi1` reported:
```
Port Mode: Trunk
```
which looks like the work was already done. It wasn't. The membership table
underneath told the real story:
```
Vlan    Name    Egress rule    Port Membership Type
1       1       Untagged       System
```

Member of VLAN 1 only, untagged, and `System` means it was never explicitly
configured. **On the SG300 every unconfigured port defaults to "Trunk" mode.**
A trunk carrying exactly one untagged VLAN is functionally an access port.
No 802.1Q tagging was happening at all — which is why the Beryl was perfectly
happy.

**Lesson: a mode label is not a behaviour.** Read the membership table, not
the one-word summary.

### The Beryl side: LuCI draws what the kernel can't do

The GL-MT3000 runs OpenWrt with modern LuCI, and LuCI happily renders the
Bridge VLAN filtering table. Filling it in correctly and clicking Save & Apply
rolled back every single time after 90 seconds.

Ruled out, in order, before blaming the firmware:
- **Wrong path.** Suspected the browser was on WiFi, not the copper test link.
  Confirmed with `nmcli radio wifi off` + `ip route get 192.168.8.1` → still
  rolled back.
- **A real gotcha found here:** the "wired fallback" wasn't actually up.
  `ip -br a` showed `enp1s0 DOWN` with no address — the whole session had been
  running over WiFi while I believed otherwise. Same class of error as the
  session 23 isolation test. **Always prove the path with `ip route get`.**
- **Config error.** Verified the VLAN table cell by cell: VLAN 1 `u*` on both
  eth0/eth1, VLAN 10 and 20 `t` on eth0 only, Local ticked on all three.
  Still rolled back.

Then went and got actual evidence from the Beryl:
```bash
ssh root@192.168.8.1
ubus call system board
bridge vlan show
```
```
kernel   5.4.211
release  OpenWrt 21.02-SNAPSHOT
target   mediatek/mt7981
bridge   -ash: bridge: not found
```

**That's the root cause.** DSA-style bridge VLAN filtering needs kernel 5.10+
on this target; this is 5.4. The `ip-bridge` package isn't even installed.
GL.iNet shipped a modern LuCI on an old kernel, so the UI offers a feature the
kernel cannot deliver.

All Beryl changes were reverted (`Revert changes` in the LuCI diff view, not
`Dismiss` — Dismiss only hides the banner and leaves changes staged).
`uci changes` returned empty afterward.

**Lesson: the UI offering a control is not proof the platform supports it.**
This is the same shape as an AWS feature being region-gated, or a Terraform
provider not yet supporting a resource.

---

## PART 2 — THE DESIGN PIVOT

With the trunk dead, two options remained:

1. Upgrade Beryl firmware — correct long-term fix, but it's the gateway for
   the entire apartment and the backup archive was taken on 21.02. Deferred to
   its own session.
2. **Make the switch route.** The SG300 supports a Layer 3 mode. VLANs 10/20
   get SVIs on the switch, the switch routes between them, gi1 stays plain
   untagged VLAN 1, and the Beryl never sees a tag.

Chose option 2. The elegant part: the broken firmware stops mattering
entirely, because no tagged frame is ever sent to the Beryl.

---

## PART 3 — THE MODE CHANGE (DESTRUCTIVE)

### The command is not where the documentation implies

Typing `set` inside `configure terminal` returns `% Unrecognized command`.
Several minutes were lost here. The command lives at the **privileged `#`
prompt**, not in config mode:

```
sg300-apt#set system mode ?
  router    System will run as a IP router
  switch    System will run as a switch
```

Checking the command tree with `?` first — instead of pasting from a tutorial
— is the habit that eventually found it.

### What it warns, and what it means

```
sg300-apt#set system mode router
Changing the switch working mode will *delete* the startup configuration file
and reset the device right after that. It is highly recommended that you will
backup it before changing the mode, continue ? (Y/N)[N] Y

%FILE-I-DELETE: File Delete - file URL flash://startup-config
client_loop: send disconnect: Broken pipe
```

The dropped session is expected, not a crash. **Going back from L3 to L2 wipes
config too** — there is no free undo, only a second rebuild.

### Recovery detail worth remembering

After the reboot, SSH was **off** — `ip ssh server` was wiped with everything
else, and port 22 refused connections. The **web GUI is enabled by default
after a reset**, so recovery went through `http://192.168.8.165` to re-enable
SSH.

The switch kept its DHCP lease at `.165`, so the `Host sg300` block in
`~/.ssh/config` still worked. The `192.168.1.254` fallback (factory default if
DHCP fails) was prepared but never needed.

Confirmed the mode actually took before building anything on top of it:
```
sg300-apt#show system mode
Mode:                 Router
Qos:                  Active
Policy-based-vlans:   inActive
```

---

## PART 4 — REBUILD, AND THE STEP THAT WAS FORBIDDEN LAST SESSION

Base config, VLANs, and port assignments were retyped from the session 23
backup, saving between phases with `copy running-config startup-config`.
(The password could not be reused — `password cannot repeat current` — so a
new one was set.)

Then the payoff:

```
configure terminal
interface vlan 10
 ip address 192.168.10.1 255.255.255.0
exit
interface vlan 20
 ip address 192.168.20.1 255.255.255.0
exit
```

**Both accepted silently, with no prompt.** In session 23 this exact command
triggered the `(Y/N)[N]` warning that had to be answered `N`, because in
Layer 2 mode assigning an IP to another VLAN *moves* the single management
interface — an instant lockout. In Router mode it genuinely adds an interface.

That silence is the concrete proof the mode change did what it was supposed
to do.

```
show ip interface
192.168.8.165/24   vlan 1    DHCP     Valid
192.168.10.1/24    vlan 10   Static   Valid
192.168.20.1/24    vlan 20   Static   Valid
```

### The static default route that shouldn't be added

The plan called for `ip route 0.0.0.0 0.0.0.0 192.168.8.1`. It failed:
```
No such instance exists.
```

Not a syntax error. `show ip route` explained it:
```
IP Forwarding: enabled
Codes: C - connected, S - static, D - DHCP

D  0.0.0.0/0        [1/2] via 192.168.8.1   vlan 1
C  192.168.8.0/24   is directly connected   vlan 1
```

**The `D` code means DHCP already installed a default route.** VLAN 1 gets its
address by DHCP from the Beryl, and that lease carries the gateway. The static
route was refused because the slot was already occupied. The plan had assumed
a statically-addressed management interface; that assumption was wrong.

`IP Forwarding: enabled` is the switch confirming it now routes.

**Expected oddity:** VLANs 10 and 20 do *not* appear as connected routes until
a device is physically plugged into a port in that VLAN. Empty VLANs stay out
of the table on this platform.

---

## PART 5 — THE RETURN-ROUTE PROBLEM

The Beryl knows `192.168.8.0/24`. It has never heard of `192.168.10.0/24` or
`192.168.20.0/24`. Traffic from those subnets would leave fine and the replies
would have nowhere to go — the classic asymmetric-routing failure where
`ping 192.168.8.1` works and `ping 1.1.1.1` doesn't.

Two static routes added on the Beryl (LuCI → Network → Routing), then verified
in the actual kernel table rather than trusting the UI:
```bash
ssh root@192.168.8.1
ip route | grep 192.168
```
```
192.168.8.0/24  dev br-lan proto kernel scope link src 192.168.8.1
192.168.10.0/24 via 192.168.8.165 dev br-lan proto static
192.168.20.0/24 via 192.168.8.165 dev br-lan proto static
```

**Adding a route in a web UI and having it in the kernel table are two
different claims.** Verify the second one.

---

## STATE AT END OF SESSION

Working:
- SG300 in Router mode, IP forwarding enabled
- VLAN 1 `.8.165` (DHCP), VLAN 10 `.10.1`, VLAN 20 `.20.1` — all Valid
- Beryl return routes live in the kernel table
- Config saved to startup-config

Not done:
- DHCP for VLANs 10/20 (attempted, `ip dhcp pool` unrecognized — unresolved
  at this point; see session 25)
- End-to-end routing never actually tested from a client
- No ACLs — VLAN 20 can reach everything
- pve-node1 still on VLAN 1, deliberately untouched
