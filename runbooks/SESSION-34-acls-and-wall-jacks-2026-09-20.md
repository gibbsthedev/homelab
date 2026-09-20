# SESSION 34 — ACLs: Turning Isolation From an Accident Into a Decision

**Date:** 2026-09-20
**Box:** Cisco SG300-10 (`sg300-apt`, 192.168.8.165) + apartment media cabinet
**Continuity:** Follows `session25-notes.md` (2026-09-07) for the switch thread.
The SESSION-26 through SESSION-33 files cover the honeypot VPS, a separate
thread that ran in parallel.

This is the session where VLAN 20 stopped being "isolated because nothing
connects it" and started being "isolated because I wrote down that it should
be, and the switch enforces it." That difference is the entire point, and it
took one firmware dead-end and three `?` lookups to get there.

Also closed, unexpectedly: the apartment wall jacks that had been dead since
before this project started.

---

## PART 0 — WHAT AN ACL ACTUALLY IS (the concept, first)

An **ACL (Access Control List)** is a list of rules the switch checks every
packet against. That's it. Each rule says "traffic matching this description
is allowed" or "traffic matching this description is dropped."

Three properties that matter more than the syntax:

### 1. Top to bottom, first match wins, then it STOPS

The switch reads rule 1. If the packet matches, it does what that rule says
and stops looking. It never reads rules 2, 3, or 4.

This is why order is the whole game. Consider these two lists, with identical
rules in different order:

```
BROKEN                              WORKING
permit ip any any        <-- 1      deny ip any 192.168.10.0 0.0.0.255
deny ip any 192.168.10.0 0.0.0.255  deny ip any 192.168.8.0 0.0.0.255
                                    permit ip any any
```

In the BROKEN version, *every* packet matches `permit ip any any` on line 1
and stops. The deny below it is dead code — it looks like security, does
nothing. Specific denies go ABOVE the general permit. Always.

### 2. There is an invisible `deny all` at the bottom

Every Cisco ACL ends with an implicit "deny everything else" that isn't shown
in the config. If a packet reaches the end without matching any rule, it's
dropped.

Consequence: an **empty ACL applied to an interface blocks 100% of traffic.**
This is why the rules get written first and the ACL gets attached last.

It's also why the last rule has to be an explicit `permit ip any any` — without
it, the invisible deny would block internet traffic too.

### 3. These ACLs are STATELESS — this matters for cloud work

Stateless means the switch has **no memory** that you sent a request.

When a VLAN 20 machine sends a DNS query out to the internet and the reply
comes back, the switch doesn't think "ah, this is the answer to that question
from earlier." It judges the returning packet cold, on its own merits, against
the same rule list.

Why this matters beyond this switch:
- **AWS Network ACLs are stateless** — same behavior, same gotcha
- **AWS Security Groups are stateful** — they DO remember outbound
  connections and automatically allow the return traffic

This is a real interview question and a real production footgun. On a
stateless system you have to think about both directions. On a stateful one
you don't. Getting this backwards is how people end up with security groups
that are wide open or NACLs that mysteriously break return traffic.

---

## PART 1 — WHY VLAN 20 NEEDED A POLICY AT ALL

Here's the thing that's easy to miss about the work in sessions 23-25:

**Before the Layer 3 migration**, VLAN 20 couldn't reach VLAN 10 or VLAN 1.
Not because of any security rule — because the switch was a dumb Layer 2
device and there was simply no path between the segments. Isolation by
*absence*.

**After the Layer 3 migration**, the switch became a router. Session 25 proved
it works: a VLAN 20 client could ping VLAN 10, the switch's own management
IP, the Beryl, and the internet. The routing worked perfectly — which meant
the isolation was gone.

That's not a mistake. That's what building a router does. But it means:

> **VLANs are broadcast separation. They are NOT a security boundary on their
> own.** Once something routes between them, isolation has to be written as
> explicit policy or it doesn't exist.

That sentence is the single most portable thing in this session.

---

## PART 2 — THE DESIGN DECISION (before any commands)

VLAN 20 is the untrusted lab segment — where sketchy containers, malware
samples, or test devices go. The question: what should it be allowed to reach?

| Destination | Decision | Reasoning |
|---|---|---|
| The internet | **ALLOW** | Otherwise the segment is useless — can't download tools or test anything |
| VLAN 10 (production) | **DENY** | Where Proxmox and the VMs will live. Something compromised on VLAN 20 must not reach real infrastructure. This is the entire reason two VLANs exist. |
| Switch mgmt IP (.8.165) | **DENY** | Never let the thing being contained talk to the thing doing the containing. An attacker who reaches the switch's admin interface can try to log in and *turn off the rules blocking them*. |
| VLAN 1 hosts (.8.0/24) | **DENY** | Beryl and everything else on the main LAN. Same logic as VLAN 10. |

**The cloud parallel:** this is a VPC security group. Untrusted workload in a
private subnet, outbound internet allowed, lateral movement to other subnets
explicitly denied. Same reasoning, different console.

**Accepted tradeoff:** from a VLAN 20 machine you can no longer SSH to the
switch or open the Beryl admin page. That's intended, but it means all
management happens from VLAN 1.

---

## PART 3 — WILDCARD MASKS (the syntax trap)

Before writing rules, checked what the firmware expects:

```
sg300-apt(config-ip-al)#deny ip ?
  any          Define Any.
  <sequence>   Specify IP source address and mask (1=wildcard, 0=hit).
               Use "any" for all IP addresses. For example, to set
               176.212.xx.xx use addr 176.212.0.0 with mask 0.0.255.255
```

**`1=wildcard, 0=hit`.** A `0` bit means "this bit must match." A `1` bit
means "ignore this bit."

This is the **inverse of a subnet mask**, and mixing them up is one of the
most common ACL mistakes:

| Goal | Subnet mask | Wildcard mask |
|---|---|---|
| Match a whole /24 (192.168.10.x) | 255.255.255.0 | **0.0.0.255** |
| Match one exact host | 255.255.255.255 | **0.0.0.0** |
| Match a /16 (176.212.x.x) | 255.255.0.0 | **0.0.255.255** |

Read it as: zeros lock the octet down, ones set it free.

---

## PART 4 — WRITING THE ACL

```
configure terminal
ip access-list extended vlan20-policy
```

**Why "extended":** a *standard* ACL can only match on SOURCE address. An
*extended* ACL matches source AND destination (plus protocol and ports).
Our entire policy is about where traffic is GOING, so standard would be
useless here.

Creating the ACL drops into a sub-mode (`(config-ip-al)#`). **Nothing is
enforced yet** — an ACL that isn't attached to an interface does nothing at
all. Safe to build.

The four rules, typed one at a time:

```
deny ip any 192.168.10.0 0.0.0.255     <- block VLAN 10 (production)
deny ip any 192.168.8.165 0.0.0.0      <- block the switch's own mgmt IP
deny ip any 192.168.8.0 0.0.0.255      <- block the rest of VLAN 1
permit ip any any                       <- allow everything else (internet)
```

**Why `ip` and not `tcp`:** `ip` matches ALL IP traffic — ICMP, TCP, UDP,
everything. Using `tcp` would block web traffic but leave ping working, which
is a half-policy that gives false confidence.

**Why rules 2 and 3 are separate** even though `.165` is inside
`192.168.8.0/24` and rule 3 would cover it anyway: reading this config in six
months, rule 2 documents an *intentional decision* about protecting the
management plane. Rule 3 covering it would just be an accident of subnet math.
Self-documenting config is worth a redundant line.

Verified before applying:

```
sg300-apt#show access-lists
Extended IP access list vlan20-policy
   deny    ip any 192.168.10.0 0.0.0.255
   deny    ip any host 192.168.8.165
   deny    ip any 192.168.8.0 0.0.0.255
   permit  ip any any
```

Note the switch rewrote `192.168.8.165 0.0.0.0` as `host 192.168.8.165` — it
recognized an all-zeros wildcard means a single host and displayed it more
clearly. Same rule, better readability.

---

## PART 5 — THE FIRMWARE DEAD-END (and how to get out of one)

Applying the ACL to the VLAN interface, per standard SG300 documentation:

```
sg300-apt(config)#interface vlan 20
sg300-apt(config-if)#service-acl
% Unrecognized command
```

**Third time this 2011 firmware diverged from documentation** (after
`write memory` and `ip dhcp pool`). So: don't guess a second syntax. Ask the
device what it actually supports.

```
sg300-apt(config-if)#?
  bridge, do, dot1x, end, exit, help, ip, ipv6, name, no, sntp
```

That's the complete command list for a VLAN interface on this image.
**Nothing ACL-related exists at the VLAN level at all.**

Rather than conclude the feature is missing, checked the *physical port*
instead — different config context, potentially different capabilities:

```
sg300-apt(config)#interface gigabitethernet3
sg300-apt(config-if)#?
  [long list, paginated with More: <space>]
  ...
  service-acl      Apply an ACL to particular interface.
  ...
```

**There it is.** ACL binding exists on physical ports, not VLAN interfaces.

### Why binding to gi3 achieves the same thing

gi3 IS the VLAN 20 access port — it's the only physical way into that segment.
Every packet from VLAN 20 enters the switch through gi3. Filtering at the port
catches exactly the same traffic as filtering at the VLAN would have.

```
sg300-apt(config-if)#service-acl ?
  input     Specify the input direction
```

Only `input`, no `output`. Fine — `input` is what we want: evaluate traffic
**as it enters the switch from VLAN 20**, before the switch routes it
anywhere. Catch it at the door.

```
service-acl input vlan20-policy
```

> **The generalizable lesson:** on old or unfamiliar gear, the device's own
> `?` help is more authoritative than any documentation online, because docs
> describe whatever version the writer had. When a command is "unrecognized,"
> don't try a second guess — enumerate what's actually there. That same move
> cracked `set system mode router` (session 24) and the DHCP question
> (session 25).

---

## PART 6 — VERIFYING BY FUNCTION, NOT BY CONFIG

A config that *looks* right and a policy that *works* are different claims.
The only way to prove the second one is to test each destination separately
and check that the results differ in exactly the way the policy says they
should.

Setup — Mint on gi3 (VLAN 20), WiFi OFF, static address via NetworkManager:

```bash
nmcli radio wifi off
sudo nmcli con mod "Wired connection 1" ipv4.method manual \
  ipv4.addresses 192.168.20.100/24 ipv4.gateway 192.168.20.1 ipv4.dns 1.1.1.1
sudo nmcli con up "Wired connection 1"
```

(WiFi off matters. With it on, Mint has a second path to everything and the
test silently lies — this has bitten this project in three separate sessions.)

### Results

| Destination | What it is | Result | Expected |
|---|---|---|---|
| 192.168.20.1 | Own gateway (VLAN 20 SVI) | 0% loss, ttl=64 | ALLOW ✓ |
| 192.168.10.1 | VLAN 10 (production) | **100% loss** | DENY ✓ |
| 192.168.8.165 | Switch management | **100% loss** | DENY ✓ |
| 192.168.8.1 | Beryl (VLAN 1) | **100% loss** | DENY ✓ |
| 1.1.1.1 | Internet | 0% loss, ttl=57 | ALLOW ✓ |

**Five destinations, three different outcomes, every one matching the written
policy.** If all five had worked, the ACL wasn't being enforced. If all five
had failed, it was too broad. The split is the proof.

### Two details worth noticing in the output

**1. The denials are silent.** 100% packet loss with no error message — not
"Destination Host Unreachable," just nothing. That's what an ACL deny looks
like: the switch receives the packet, matches a deny rule, and discards it
without telling anyone.

This is deliberate. An explicit rejection message would confirm to an attacker
that the destination exists and something is filtering. Silence gives them
nothing.

(Contrast: the "Destination Host Unreachable" seen on the Juniper port in a
previous session was an ARP failure — a different layer, different meaning.)

**2. The allowed traffic still routes correctly.** `1.1.1.1` came back at
ttl=57, meaning the packet still crossed the switch, hit the Beryl, got NATed,
and reached the internet. The permit rule works and the denies aren't
overreaching.

### The contrast test (accidental, but the best proof)

After cleanup, back on VLAN 1:

```
ping -c 2 192.168.8.165  ->  0% loss, ttl=64
```

Same target that showed 100% loss from VLAN 20. **The ACL is scoped exactly
to gi3 and nowhere else** — proven by contrast, not just by reading config.

---

## PART 7 — THE WALL JACKS (unrelated, finally solved)

Separate thread, closed this session.

Three apartment Ethernet wall jacks had been dead since before this project
started — all returning `carrier: 0` (no physical link at all). The original
investigation ruled out the laptop, the cable, and the NIC, and concluded the
fault was "upstream, in a patch panel or managed switch we can't see."

This session: **found the structured media cabinet** (which the original
investigation had concluded didn't exist in the apartment).

Inside:
- A Wavenet 8-port patch panel, ports 1-4 punched down with the in-wall runs
- A small unmanaged switch below it
- Four blue patch cables connecting panel ports 1-4 to the switch
- A power adapter for the switch, plugged into a wall outlet

**The switch had no lights.** Swapped which outlet it was plugged into —
lights came on.

Verification from Mint, plugged into a previously-dead wall jack, WiFi off:

```
cat /sys/class/net/enp1s0/carrier   ->  1
ip -br a                            ->  enp1s0: 100.110.30.7/22 (WhiteSky CGNAT)
ping -c 3 1.1.1.1                   ->  0% loss, ttl=59, ~6.5ms
```

**The jacks were never broken.** The media-cabinet switch had lost power. One
dead outlet took out every jack in the apartment simultaneously — which is
exactly why all three failed at once after previously working.

Latency note: 6.5ms wired vs 13-14ms on either WiFi path. Not just working —
measurably better.

> **Lesson:** the original diagnosis correctly predicted "a common upstream
> failure." It just assumed something sophisticated (reconfigured managed
> switch, unpatched ports) when the answer was a dead outlet. When the logical
> layer has nothing left to explain, go look at the physical one.

WhiteSky support visit: **cancelled, not needed.**

---

## STATE AT END OF SESSION

**Complete — the segmentation work is done end to end:**
- VLANs 1/10/20 exist, switch routes between them (L3 mode)
- `vlan20-policy` ACL written, applied to gi3, saved to startup-config
- Verified by function across five destinations
- VLAN 20 isolation is now explicit written policy, not absence of routing

**Also resolved:**
- Apartment wall jacks working — live WhiteSky drop available at a wall jack
- WhiteSky support appointment cancelled

**Queued next, in dependency order:**
1. **Move Proxmox from Beryl's eth1 to SG300 gi2 (VLAN 10)**, statically
   addressed. This frees Beryl's second Ethernet port.
2. **Beryl wired WAN** — now sourced from a live wall jack instead of the
   ceiling Juniper port, which was confirmed single-MAC-locked in a previous
   session. Requires step 1 to free the port.

**Still open, lower priority:**
- SG300 NTP unconfigured (timestamps won't correlate with other logs)
- Reboot durability test on the full current config
- VLAN 10 has no ACL yet (nothing is on it yet — do it when Proxmox moves)
