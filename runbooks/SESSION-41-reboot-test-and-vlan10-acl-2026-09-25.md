# SESSION 41 — Switch Reboot Test + VLAN 10 ACL — 2026-09-25

**Status: COMPLETE.** Two of the three items pinned since SESSION-34 are
done: the SG300's saved config survives a power loss, and production
(VLAN 10) now has a written access policy, same as the lab VLAN did.

The third pinned item, NTP, is **in progress** — see §3. Nothing has been
configured yet, only the command tree explored.

**Continuity:** follows `SESSION-39-beryl-wired-wan-and-failover-2026-09-23.md`.
Closes two of the three follow-ups in `hardware/sg300-10.md`'s Next steps.

---

## 1. Reboot durability test — PASSED

Never tested before tonight. Every piece of switch config built since
SESSION-24 — Router mode, three SVIs, the VLAN 20 ACL, the DHCP reservation
added on the Beryl for the switch's own address — had only ever been proven
to work *while running*. A saved config that doesn't reload correctly after
power loss is a config that only looks finished.

### What made this safe to test now

A DHCP reservation for the switch's MAC (`20:3a:07:f3:de:31` →
`192.168.8.165`) was added on the Beryl before this test. Without it, the
switch's own management address — the one every VLAN 10/20 return route on
the Beryl points at — was only staying the same by dnsmasq habit, not
guarantee. A reboot that came back with a different address would have taken
down Proxmox, both VMs, and the public sites, with nothing about the symptom
pointing at DHCP.

### Procedure

```
copy running-config startup-config      # confirm nothing unsaved
```
On Mint, in a second terminal, a continuous watch before rebooting:
```bash
ping -O 192.168.8.165
```
Then on the switch:
```
reload
```

### Result

The switch came back at `192.168.8.165` — the reservation held. Full
verification passed:

| Check | Result |
|---|---|
| `show system mode` | `Router` |
| `show ip interface` | All three SVIs `Valid` |
| `show access-lists` | `vlan20-policy` present, all four lines intact |
| `show running-config` | `service-acl input vlan20-policy` still under gi3 |
| `ping 192.168.10.2` from Mint | replies |
| Proxmox web UI | loads at `https://192.168.10.2:8006` |
| `richgibbs.dev` | loads |

Config, VLANs, SVIs, and the ACL all survived a cold reload intact.

---

## 2. VLAN 10 ACL — vlan10-policy

Built and tested the same way as VLAN 20 in SESSION-34: write the rules,
verify the list, attach to the physical port, test by function — never trust
that a config which was accepted is a config that works.

### The policy

Least privilege, not maximum lockdown. Production (VLAN 10) gets exactly the
access it needs and nothing more:

```
ip access-list extended vlan10-policy
 deny ip any 192.168.20.0 0.0.0.255     # lab (untrusted) — no reason to reach it
 deny ip any host 192.168.8.165         # switch's own management login
 permit ip any any                      # internet, and VLAN 1 — see below
```

Bound to the physical access port, same rule as VLAN 20's ACL — `service-acl`
only exists on physical ports on this firmware, not on `interface vlan X`:

```
interface gigabitethernet2
 service-acl input vlan10-policy
```

### Why VLAN 10 → VLAN 1 is deliberately left open

This is the one line worth understanding, not just copying. It would be
tempting to also deny VLAN 1, mirroring VLAN 20's policy exactly. That would
break management.

The SG300's ACLs are **stateless** — same fact as SESSION-34, different
consequence here. When Mint opens Proxmox's web UI or SSHes in, the *request*
originates on VLAN 1 and is unaffected either way. But Proxmox's *reply*
travels VLAN 10 → VLAN 1. A stateless ACL cannot tell "a reply to a
connection Mint opened" apart from "a new connection VLAN 10 is initiating."
Blocking that direction blocks both — Mint would lose the ability to manage
Proxmox and the switch entirely.

So VLAN 10 → VLAN 1 stays open, and that openness is a **known, accepted
limitation**, not an oversight:

> In a properly segmented network, management traffic runs on its own
> dedicated VLAN — separate from user devices (phones, laptops, TVs). Here,
> VLAN 1 is both "the management network" and "everything that isn't VLAN 10
> or 20." Anything on the Beryl's WiFi can currently reach Proxmox's login
> page. Fixing that properly means adding a fourth, dedicated management
> VLAN — a real future project, not a quick fix, and exactly what an
> enterprise network would do differently here.

### Verified by function

From the Proxmox console (SSH from Mint still works — VLAN 10 → VLAN 1
replies are permitted):

| Destination | Result | Proves |
|---|---|---|
| `1.1.1.1` | replies | Internet allowed |
| `192.168.8.1` (Beryl) | replies | VLAN 1 intentionally open |
| `192.168.8.165` (switch mgmt) | 100% loss | Line 2 works |
| `192.168.20.1` (lab) | 100% loss | Line 1 works |
| `getent hosts google.com` | resolves | DNS unaffected |
| `richgibbs.dev` | loads | Public sites unaffected |

Two allowed, two denied, DNS and the live sites both fine — the same
"split proves the policy" pattern as every ACL test in this project.

Config saved with `copy running-config startup-config` after applying.

---

## 3. NTP — IN PROGRESS, nothing configured yet

The switch clock is set manually and reports garbage dates from 2011 — its
own logs can't be lined up in time against anything else in the lab. Fixing
it is the last item pinned since SESSION-34.

This firmware speaks **SNTP** (Simple Network Time Protocol), a lighter
version of NTP — visible as its own top-level command, distinct from
`clock`. Command tree explored so far, from `(config)#`:

```
sntp ?
  authenticate         require auth for received NTP traffic
  authentication-key
  broadcast
  client
  server               <- configure the device to request/accept NTP from a server
  trusted-key
  unicast              <- global SNTP unicast config

clock ?
  dhcp                 take timezone + DST from DHCP
  source               external time source for the system clock
  summer-time
  timezone             <- display purposes only
```

**Not yet checked** — the actual next step: `clock source ?`,
`sntp unicast ?`, and `sntp server ?`. The last one matters most: it decides
whether this switch can be pointed at a hostname (`time.cloudflare.com`) or
only a raw IP, which depends on whether it can do DNS lookups at all.

Pick this up by reading those three, then deciding a server and a timezone,
then verifying with `show sntp` / `show clock` after it's configured — same
build-then-verify pattern as everything else in this file.

---

## 4. Documentation restructure — planned, not yet built

Discussed but deliberately deferred rather than done rushed. The `runbooks/`
folder is now a long, dated history — accurate, but not something to hand
someone (or future-self) who just wants "what does the network look like
right now."

Planned addition, once there's a natural pause:

```
networking/
  README.md            <- current state + links out
  topology.md          <- the diagram, VLAN/IP table
  acls.md              <- both policies, in one place, with how to test them
  switch-sg300.md      <- config, commands, firmware quirks
  router-beryl.md      <- WAN, failover, DHCP reservation
  troubleshooting.md   <- failure signatures, gotchas, recovery paths
```

The principle: `networking/` describes **how things are now**, kept short
enough to scan. `runbooks/` stays exactly as it is — the dated record of
**how it got that way**. Nothing gets deleted; this is a table of contents
layered on top, not a replacement.

---

## 5. State at end of session

```
WhiteSky (wired wall jack; WiFi repeater = tested failover backup)
        |
   Beryl AX  192.168.8.1  (DHCP reservation: switch MAC -> .165, fixed)
        |
   SG300-10  sg300-apt  192.168.8.165  (Router mode — reboot-tested)
        |
        +-- gi1  VLAN 1   Beryl uplink
        +-- gi2  VLAN 10  pve-node1 + VMs — vlan10-policy applied
        +-- gi3  VLAN 20  lab — vlan20-policy applied
        +-- gi5  VLAN 1   Mint
```

Both production and lab segments now carry written, tested ACL policy. The
switch's own saved state has been proven to survive a cold reboot. NTP and
the documentation restructure are the two open threads.

## Follow-ups

- [x] Reboot durability test — passed, see §1
- [x] VLAN 10 ACL — `vlan10-policy` applied to gi2, verified — see §2
- [ ] NTP — command tree explored, nothing configured, see §3
- [ ] Build the `networking/` folder — see §4
