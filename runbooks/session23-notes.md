# Session 23 — Deploying the SG300-10: First VLAN Segmentation

Date: 2026-09-01

The Cisco SG300-10 was bought back in session11 specifically to serve the
Network+ / SOC learning track, then sat undeployed. This session put it in
the path, built two VLANs, and proved isolation actually works — while
deliberately stopping short of the step that would have caused a lockout.

---

## PART 1 — GETTING IN: A 2010-ERA SWITCH MEETS A MODERN SSH CLIENT

Mint's OpenSSH refused to connect at all:
```
no matching key exchange method found
```
The switch only offers `diffie-hellman-group-exchange-sha1` and
`diffie-hellman-group1-sha1` — both disabled by default in modern OpenSSH
because SHA1 is broken. This isn't a misconfiguration on either side; it's a
15-year gap in security defaults showing up as a connection failure.

One-off connection:
```bash
ssh \
  -o KexAlgorithms=+diffie-hellman-group1-sha1,diffie-hellman-group-exchange-sha1 \
  -o HostKeyAlgorithms=+ssh-rsa \
  -o PubkeyAcceptedAlgorithms=+ssh-rsa \
  -o Ciphers=+aes128-cbc,3des-cbc,aes256-cbc \
  cisco@192.168.8.165
```
Made permanent as a scoped `~/.ssh/config` entry, so the weakened crypto
applies to **this host only** and never leaks to any other SSH target:
```
Host sg300
    HostName 192.168.8.165
    User cisco
    KexAlgorithms +diffie-hellman-group1-sha1,diffie-hellman-group-exchange-sha1
    HostKeyAlgorithms +ssh-rsa
    PubkeyAcceptedAlgorithms +ssh-rsa
    Ciphers +aes128-cbc,3des-cbc,aes256-cbc
```
- The `+` prefix means "add to the default list," not "replace it."
- **WHY scoping matters:** a global `KexAlgorithms +...` would re-enable SHA1
  key exchange for every SSH connection from this laptop, including the
  honeypot VPS. Restricting it to a `Host` block keeps the weakening on a
  LAN-only device that can't be reached from the internet.

Login is the default `cisco` account at privilege 15. Custom users created in
the GUI didn't work until the password was actually saved to startup-config —
and `enable` is unnecessary when the prompt already ends in `#`.

## PART 2 — THE SAVE COMMAND THAT ISN'T `write memory`

A pasted config block failed because `write memory` — the command muscle
memory from most Cisco IOS material — **is invalid on this switch's image**.
Worse, mid-paste output showed `Cold Startup`, meaning an unsaved reboot was
a real risk.

The command that actually works here:
```
copy running-config startup-config
```
(Confirm `Y` at the overwrite prompt.)

**Lesson: "Cisco" is not one command set.** The SG300 runs a different image
from mainline IOS, and a command that's universal in tutorials can simply not
exist. Verify the save command on the actual hardware before assuming a
config survived.

## PART 3 — THE VLANS THAT STUCK

```
vlan database
vlan 10,20
exit
hostname sg300-apt
ip ssh server
interface gigabitethernet2
 switchport mode access
 switchport access vlan 10
exit
interface gigabitethernet3
 switchport mode access
 switchport access vlan 20
```

Resulting port layout:
```
gi1      VLAN 1 access    Beryl uplink — KEEP
gi2      VLAN 10 access   reserved for pve-node1 (NOT moved yet)
gi3      VLAN 20 access   lab / isolated
gi4-10   VLAN 1 access    spare / management
```
- `switchport mode access` means the port carries exactly one untagged VLAN —
  the right mode for an end device (a laptop, a server), as opposed to a trunk
  port that carries multiple tagged VLANs to another switch or router.
- Clock set manually (`clock set 11:42:00 1 Sep 2026`); it reports UTC
  regardless of intent and there's no NTP configured. Fine for now — but worth
  knowing when correlating switch logs against anything else.

## PART 4 — PROVING ISOLATION (the test that's easy to fake)

```bash
nmcli radio wifi off      # ESSENTIAL — see below
# plug Mint into gi3 (VLAN 20) only
ip -br a
ping -c 2 192.168.8.1
```
Result: the Ethernet interface sat at "looking for an IP" and never got one.
**That's a pass** — VLAN 20 has no DHCP server, because the Beryl only serves
the flat 192.168.8.0/24 on VLAN 1 and nothing bridges the two.

**WHY `nmcli radio wifi off` is non-negotiable here:** with WiFi still on, the
laptop stays reachable over VLAN 1 the whole time, so a successful ping proves
nothing about the copper connection. The test would silently pass while
actually measuring the wrong path. **Same category of mistake as verifying a
service by "systemctl says running" — confirm you're testing the path you
think you're testing.**

## PART 5 — THE STEP DELIBERATELY NOT TAKEN

Attempted to give VLAN 20 a gateway so it could route:
```
configure terminal
interface vlan 20
ip address 192.168.20.1 255.255.255.0
```
The switch responded:
```
Would you like to apply this new configuration? (Y/N)[N]
```
**Answered N — correctly.** The SG300 here is operating as a **Layer 2**
switch with exactly one management interface. Assigning an IP to VLAN 20
wouldn't add a second address; it would **move management** to VLAN 20, and
since the admin laptop sits on VLAN 1, that's an instant lockout requiring
physical console access to recover.

Confirmed the safe state afterward:
```
show ip interface
# Gateway 192.168.8.1   dhcp
# 192.168.8.165/24      vlan 1   DHCP   Valid
```

**Lesson: a confirmation prompt defaulting to `[N]` is often the hardware
warning you about exactly this.** Read what the change actually does to the
path you're connected over before accepting.

## PART 6 — WHAT'S NEXT (and why it's a separate session)

The path to making VLANs 10/20 actually routable runs through the Beryl, not
the switch: configure an 802.1Q trunk on gi1, then serve DHCP for VLANs 10/20
from the Beryl itself. **High lockout risk** — it changes the link everything
currently depends on.

Rules set for that future session:
- Back up the Beryl config first.
- **Do NOT move pve-node1 onto gi2 while the Beryl is still flat** — the
  Proxmox host would land on a VLAN with no DHCP and no gateway, taking both
  VMs off the network.

## LESSONS

- **Scope weakened crypto to the one host that needs it.** A `Host` block in
  `~/.ssh/config` keeps SHA1 key exchange on a LAN-only switch instead of
  globally re-enabling it for every SSH connection including internet-facing
  boxes.
- **"Cisco" isn't one command set** — `write memory` doesn't exist on this
  image; it's `copy running-config startup-config`. Verify the save command on
  the actual device.
- **Turn off the other path before testing isolation.** Leaving WiFi on during
  a VLAN isolation test produces a confident, meaningless pass.
- **On a Layer 2 switch, adding an IP to a VLAN moves management rather than
  adding an interface** — that prompt defaulting to `[N]` was the device
  warning about a lockout.
- **Deploy the thing you bought.** The switch sat unused from session11 to
  now; the actual VLAN work turned out to be a single evening, and the
  learning (802.1Q, access vs. trunk, L2 vs. L3, management-plane risk) is
  directly Network+ material.

---
END. See `hardware/sg300-10.md` for the current-state device reference.
