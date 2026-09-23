# SESSION 39 — Beryl Wired WAN + Failover Proven — 2026-09-23

**Status: COMPLETE.** The Beryl's internet connection moved from the WiFi
repeater to a wired wall jack. Download went from **52 to 332 Mbps**. The
WiFi repeater was kept as an automatic backup, and failover was tested by
pulling the cable: **zero packets lost, both directions.**

**Continuity:** the last item in `SESSION-36`'s follow-ups ("Next: Beryl wired
WAN"). The speed investigation that justified it is in
`whitesky-ethernet-uplink-investigation.md`; the wall jacks coming back is in
`SESSION-34`.

---

## 0. WHY THIS WAS WORTH DOING

Before tonight, every packet leaving the apartment went through a **WiFi
repeater hop**: the Beryl received WhiteSky's WiFi on one radio and
retransmitted it. That hop splits airtime and rides the same crowded spectrum
as every other device in the building.

The measurement that proved it was the bottleneck (from the Juniper
investigation, repeated here):

| Path | Down | Up | Ping |
|---|---|---|---|
| FBI WiFi, through Beryl's repeater WAN | 52.36 | 94.37 | 14 ms |
| WhiteSky-Prose direct (phone) | 203.47 | 323.93 | 13 ms |

If WhiteSky itself were rationing bandwidth, the direct test would have been
capped too. It wasn't. So the limit was the Beryl's WiFi-as-WAN link
specifically — and a cable removes it.

---

## 1. THE PLAN WAS WRONG ABOUT WHICH PORT

The plan (`PLAN-beryl-wired-wan-migration.md`) said: use the port Proxmox
vacated (eth1) as the new WAN, leave the switch uplink alone.

**The GL.iNet firmware doesn't allow that.** Looking at the admin panel before
clicking anything showed it:

- **Port Management** has a WAN/LAN toggle **only on the WAN-labeled port.**
  The LAN tab has no such option.
- In **Multi-WAN**, "Ethernet" means that one port specifically.

The WAN-labeled port was also the one currently carrying the switch uplink —
visible as `1000 Mbps full duplex` on the WAN tab while it was set to
"As LAN Port."

> **Lesson:** a plan written from general knowledge of a device is a guess
> until you've seen the actual screen. Screenshot first, click second. This is
> the same discipline that found `service-acl` on the SG300 when the docs said
> it lived somewhere else.

### The corrected sequence

The fix was a cable swap before the setting change, **in this order:**

1. Move the **switch uplink** from the WAN port → the LAN port (free since
   Proxmox moved to the switch in SESSION-36)
2. Flip the WAN port from LAN back to **WAN** in Port Management → Apply
3. **Only then** plug the wall jack into the WAN port

**Why the order matters:** if you flip the port to WAN while the switch is
still plugged into it, the switch lands on the *internet* side of the router.
The whole internal network — Proxmox, both VMs, all three VLANs, the public
sites — gets cut off.

**Managed from FBI WiFi, not wired Mint.** Mint reaches the Beryl through the
switch, so it loses the admin panel the moment the switch cable moves.

### Upside of the correction

The end state is the standard, fully supported configuration, and the WAN is
now on the port rated for 2.5 Gbps.

```
BEFORE                                  AFTER
WAN port (2.5G) -> switch (as LAN)      WAN port (2.5G) -> WALL JACK (WAN)
LAN port (1G)   -> Proxmox              LAN port (1G)   -> switch uplink
Internet via WiFi repeater              Internet via cable, repeater = backup
```

Moving the switch uplink to the 1G port costs nothing — the SG300 is a
gigabit switch.

---

## 2. MULTI-WAN WAS ALREADY CONFIGURED CORRECTLY

Before changing anything, checked the priority order:

```
Mode: Failover
  1  Ethernet     <- the wall jack
  2  Repeater     <- the WiFi link
  3  Tethering
  4  Cellular
```

**Failover** means one link at a time, switching automatically if the active
one dies. **Load Balance** uses several links at once and spreads connections
across them. Failover is the right choice here: the wire is simply better, so
there's no reason to push traffic over the slower link.

The Apply button was faded — no pending changes. This was the firmware
default. Nothing to click.

> A priority list only *says* which link is preferred. It doesn't prove the
> link is up, or that failover works. That needed testing (§4).

### MAC Mode left at default

When the port switched to WAN mode, a **MAC Mode** option appeared, showing
the Beryl's own WAN MAC (`94:83:c4:94:04:03`). Left at default.

Kept in reserve: the Juniper port turned out to accept only the first device
it ever saw (see the WhiteSky investigation), and Mint was the first device on
the wall jack too. If the Beryl had failed to get an address, cloning Mint's
MAC (`74:86:7a:3f:58:f7`) was the first thing to try. It wasn't needed.

---

## 3. RESULT

Measured from Windows on FBI WiFi — **the same path as the 52/94 baseline**,
so the only thing that changed between tests is the WAN link. One variable.

| | Down | Up | Ping |
|---|---|---|---|
| Before — WiFi repeater WAN | 52.36 | 94.37 | 14 ms |
| **After — wired WAN** | **332.57** | **318.15** | **13 ms** |
| WhiteSky-Prose direct (reference) | 203.47 | 323.93 | 13 ms |

**6.4× faster download, 3.4× faster upload.**

It also beat the direct-to-WhiteSky WiFi test by ~130 Mbps on download. That
one was measured over a phone's WiFi, so the wire removed two bottlenecks at
once. What limits this test now is most likely the WiFi hop between the
testing device and the Beryl, not the WAN.

---

## 4. FAILOVER — TESTED, NOT ASSUMED

"Failover: enabled" on a settings page is a claim. Pulling the cable makes it
a fact.

### Test 1 — pull the WAN cable (wall jack out of the Beryl)

Continuous ping from wired Mint:

```bash
ping -O 1.1.1.1
```

`-O` makes Linux print `no answer yet for icmp_seq=N` for every lost packet,
instead of silently skipping sequence numbers.

**Why wired Mint is a valid test here:** Mint's path is
Mint → gi5 → switch → Beryl → WAN. The cable being pulled is *past* the point
where Mint's traffic enters the Beryl. Mint's WiFi also lands on the Beryl, so
both of Mint's paths hit the same WAN link — there's no route around the thing
being tested. (Normally WiFi-on ruins a test by providing an alternate path.
Here both paths converge before the failure point, so it doesn't matter.)

| Sequence | Latency | Path |
|---|---|---|
| 1–11 | 7.24–7.43 ms | Wired WAN |
| 12 | 12.4 ms | ← cable pulled |
| 12–38 | 7.87–15.0 ms | WiFi repeater |
| 39–57 | 7.09–7.48 ms | Wired WAN ← cable back in |

**57 sent, 57 received, 0% packet loss.** Two switchovers, no drops.

That's better than expected. Pulling a cable drops the link instantly, so the
router doesn't have to wait for its health checks to fail — it sees the link
die and swaps routes before a packet is lost.

### Reading the result: spread matters more than average

| | Range | Spread |
|---|---|---|
| Wired | 7.24 – 7.43 ms | **0.2 ms** |
| Repeater | 7.87 – 15.0 ms | **7 ms** |

That's the signature of radio. A cable delivers every packet in almost exactly
the same time. WiFi shares the air and retransmits after collisions, so timing
wanders. That variation is **jitter**, and it's what makes video calls stutter
and games lag — often more noticeable than raw speed.

**TTL stayed at 58 the whole time.** TTL counts *routers* crossed, not cables
or radio hops. Both paths cross the same routers (Beryl, then WhiteSky), so
hop count is identical. Only the medium changed. That's why latency and jitter
were the evidence here, not TTL.

### Test 2 — pull Mint's own cable

Different test: this failed Mint's link to the switch instead of the Beryl's
link to the internet.

```
seq 1-4    ~7.3 ms   wired
seq 5-10   LOST      (6 packets)
             From 192.168.8.142 icmp_seq=5 Destination Host Unreachable
             From 192.168.8.142 icmp_seq=6 Destination Host Unreachable
             From 192.168.8.142 icmp_seq=7 Destination Host Unreachable
seq 11-31  ~7.5 ms   via Mint's WiFi
31 sent, 25 received, 19% loss
```

Mint **did** fail over — to its own WiFi connection to the Beryl — but it took
about **6 seconds**, versus zero for the Beryl.

**Why the difference:** Mint's WiFi was already connected (`.186`) alongside
wired (`.142`). The WiFi route just had lower priority — metric 600 versus 100
for wired. When the cable came out, Mint kept trying the dead wired path first.
Once NetworkManager noticed and withdrew the wired route, WiFi was the only
default left and traffic resumed. The Beryl did the same thing; it's just
faster at noticing a dead link.

**Read who's reporting the error.** `192.168.8.142` is Mint's *own* address.
Mint reported the failure to itself — it tried to reach the gateway over a
dead link, got nothing, and gave up locally. Those packets never left the
laptop.

### Failback confirmed

After plugging Mint back in:
```
ip route get 1.1.1.1
1.1.1.1 via 192.168.8.1 dev enp1s0 src 192.168.8.142
```
`dev enp1s0` = wired. `src .142` = the wired address. Mint returned to the
better path on its own.

**`ip route get <destination>`** is the one-line answer to "which path is this
machine actually using right now" — better than watching for notifications.

---

## 5. THREE FAILURE SIGNATURES, NOW ALL SEEN

This session completed a set that's been building across the project:

| What you see | What it means | Where it was seen |
|---|---|---|
| **Destination Host Unreachable from your own IP** | Your device can't reach the next hop — dead link, ARP failing | Test 2 tonight; the Juniper port test |
| **Silent timeouts, no error at all** | Something downstream dropped it on purpose | The VLAN 20 ACL (SESSION-34) |
| **0% loss despite a pulled cable** | Redundancy caught it | Test 1 tonight |

Knowing which one you're looking at tells you *where* to look before you run
another command.

---

## 6. LESSONS

**Screenshot the real screen before following a plan.** The plan was wrong
about which port could be WAN. It would have been discovered mid-change, with
the network half-reconfigured.

**Order of operations can be the whole risk.** Every individual step here was
simple. Doing step 2 before step 1 would have cut off the entire internal
network.

**A setting is a claim; a pulled cable is proof.** Failover was configured by
default the whole time. Until the cable came out, nobody knew if it worked.

**Redundancy protects one link, not the chain.** The Beryl's WAN had a backup.
Mint's cable had a slower backup. The switch, and the single cable between the
switch and the Beryl, still have none. That's fine for a homelab — but it's
worth knowing exactly where the single points of failure are.

**The WiFi second path is the same behavior, two ways.** It ruined the early
VLAN isolation tests by hiding failures. Tonight it was the backup. Same
mechanism; whether it helps or hurts depends on what you're testing.

---

## 7. STATE AT END OF SESSION

```
WhiteSky (CGNAT)
  |
  wall jack --cable--> Beryl WAN port (2.5G)   PRIMARY
  WhiteSky-Prose WiFi -> Beryl repeater        BACKUP (failover, 0 loss)
  |
  Beryl AX  192.168.8.1
  |  LAN port (1G) -> SG300 gi1
  |  WiFi SSIDs: FBI Surveillance Van #4 (2.4G) / #5 (5G)
  |
  SG300 sg300-apt  192.168.8.165   (Router mode)
     gi1   VLAN 1   Beryl uplink
     gi2   VLAN 10  pve-node1 192.168.10.2 (+ VMs .10.10, .10.206)
     gi3   VLAN 20  lab, vlan20-policy ACL
     gi5   VLAN 1   Mint
```

## Follow-ups

- [ ] **Fresh Beryl config backup** (LuCI → System → Backup / Flash Firmware →
      Generate archive). The one taken before this session has the OLD port
      layout — restoring it would undo this change. Name it distinctly, e.g.
      `beryl-wired-wan-2026-09-23.tar.gz`.
- [x] Update SSH configs / bookmarks still pointing at `192.168.8.2`, `.8.10`,
      `.8.206` (carried over from SESSION-36) — done.
- [ ] Optional: check the WAN tab's negotiated speed. The WAN port is 2.5G, but
      the link rate depends on the building's switch at the other end of the
      wall jack.
