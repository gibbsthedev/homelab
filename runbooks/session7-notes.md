# Session 7 — The "Cable Problem" That Was Never a Cable

Date: 2026-07-24

A week of daily network outages, blamed on a loose ethernet cable, turned
out to be a driver bug. This one is worth reading carefully — the wrong
diagnosis persisted for days because the evidence genuinely looked like a
physical fault.

---

## PART 1 — WHAT ACTUALLY HAPPENED

### The symptom
Roughly once a day, the Proxmox host would stop passing network traffic.
- Could not ping the router (192.168.8.1)
- Could not reach the internet
- The VM at 192.168.8.10 became unreachable too
- `git pull` failed with DNS errors
- The web UI was unreachable

Every time, **unplugging and reseating the ethernet cable fixed it.**

That's why it looked physical. Reseating a cable fixing a network problem
is about as clear a signal as you get.

### The wrong conclusion (twice)
First occurrence: assumed a marginal connection, reseated, moved on.
Second occurrence: concluded the cable was FAILING — "a fault that recurs
after a reseat is a failing cable, not a loose one." Recommended replacing
it. Rich pointed out the cable was brand new. That should have prompted a
rethink sooner than it did.

### The actual cause
```
dmesg -T | grep -iE "eth|link|nic0|enp0s31f6" | tail -20

[Fri Jul 24 15:23:37 2026] e1000e 0000:00:1f.6 nic0: Detected Hardware Unit Hang:
[Fri Jul 24 15:23:39 2026] e1000e 0000:00:1f.6 nic0: Detected Hardware Unit Hang:
[Fri Jul 24 15:23:41 2026] e1000e 0000:00:1f.6 nic0: Detected Hardware Unit Hang:
... repeating every 2 seconds ...
```

**"Detected Hardware Unit Hang"** is a well-known bug in Intel's `e1000e`
network driver. The NIC's transmit queue stalls. The hardware stops
sending packets, but the physical link stays electrically up.

### Why this looked exactly like a cable fault

| Observation | Cable explanation | Actual explanation |
|---|---|---|
| `LOWER_UP` present, nothing passing | Partially-seated connector | Link is up; TX queue is stalled |
| Reseating fixes it | Restores contact | Forces a link-down/up event, which RESETS the adapter |
| Recurs daily | Connector working loose | Hang recurs under sustained traffic |
| New cable didn't help | Bad new cable | Cable was never involved |

**The reseat wasn't fixing a connector. It was power-cycling the NIC.**
Any action that resets the link clears the stalled queue — unplugging,
`ip link down/up`, a reboot. They're indistinguishable from the outside.

### What should have been done on day one
```
dmesg -T | grep -i "hang\|link\|eth"
```
The kernel had been logging the real cause the entire time, with
timestamps. Interface state (`ip link`) tells you what the link looks like
NOW. `dmesg` tells you what the kernel has been complaining about, and it
is the only place a driver-level fault shows up.

---

## PART 2 — CONCEPTS

### What "hardware offload" means
Modern network cards can do work the CPU would otherwise do. The main ones
here:

- **TSO — TCP Segmentation Offload.** Normally the kernel chops a large
  chunk of data into individual packets sized to fit the network (~1500
  bytes). With TSO, the kernel hands the NIC one big buffer and says "you
  split it." Saves CPU.
- **GSO — Generic Segmentation Offload.** A software cousin: defer the
  splitting as late as possible in the kernel's own stack.
- **GRO — Generic Receive Offload.** The reverse, on the receive side:
  merge many small incoming packets into fewer large ones before handing
  them up the stack. Fewer trips through the network code.

All three are performance optimizations. All three are also where the
`e1000e` hang bug lives — the segmentation logic mismanages the descriptor
ring under certain traffic patterns and the queue wedges.

**Turning them off trades a little CPU for stability.** The kernel does
the work the NIC was doing. On a 6-core i5 sitting at 0.6% load, that cost
is invisible.

### Why the link stays "up" while nothing passes
Ethernet link state is negotiated at the physical layer — two devices
agreeing there's a working electrical connection. That negotiation happens
in one part of the chip.

Actually moving packets happens elsewhere: the driver writes packet
descriptors into a ring buffer, the NIC reads them and transmits. When
that ring wedges, the physical negotiation is entirely unaffected. The
link light stays on. `ip link` reports `LOWER_UP`. And nothing moves.

**This is the second time this lab has produced the same lesson:
`LOWER_UP` means "carrier detected," not "the connection works."** First
time it was a genuinely loose connector; this time a driver stall. Same
misleading indicator, different root cause.

### Why the VM went down too
The VM's virtual NIC attaches to `vmbr0`, a software bridge on the host.
That bridge's path to the outside world is `nic0` — the physical port.
When `nic0` wedged, everything behind the bridge lost external
connectivity.

Notably, host-to-VM traffic kept working during earlier episodes, because
that stays inside the bridge and never touches the physical NIC. That
asymmetry was a real clue and it was there the whole time.

Each of these outages also risked tripping OpenClaw's crash-loop breaker
again — the gateway restarting with no network is exactly what produced
the eleven unclean boots that killed Telegram in session 3.

---

## PART 3 — THE FIX

### 1. Clear the current hang
```
ip link set nic0 down
ip link set nic0 up
```
Software equivalent of unplugging the cable. Resets the adapter and clears
the stalled transmit queue.
(`modprobe -r e1000e && modprobe e1000e` unloads and reloads the driver
entirely if a link bounce isn't enough. A reboot always works.)

### 2. Disable the offloads
```
ethtool -K nic0 tso off
ethtool -K nic0 gso off
ethtool -K nic0 gro off
```
**Note: these had to be run SEPARATELY.** The combined form
`ethtool -K nic0 tso off gso off gro off` failed with "could not change
device features" — ethtool rejects the whole batch if any part of it is
awkward, rather than applying what it can.

`-K` (capital) SETS features. `-k` (lowercase) SHOWS them.

Verify:
```
ethtool -k nic0
```
Should show:
```
tcp-segmentation-offload: off
generic-segmentation-offload: off
generic-receive-offload: off
```
Entries marked `[fixed]` cannot be changed — that's the hardware saying
the feature is permanently on or off.

### 3. Make it survive reboot
`ethtool` settings are runtime-only. They vanish on restart.

Added to `/etc/network/interfaces`, under the **vmbr0** stanza:
```
auto vmbr0
iface vmbr0 inet static
        address 192.168.8.2/24
        gateway 192.168.8.1
        bridge-ports nic0
        bridge-stp off
        bridge-fd 0
        post-up /sbin/ethtool -K nic0 tso off
        post-up /sbin/ethtool -K nic0 gso off
        post-up /sbin/ethtool -K nic0 gro off
```

**Why under vmbr0 and not nic0:** the `iface nic0 inet manual` stanza has
no `auto` line, so nic0 is brought up as a side effect of vmbr0 claiming
it via `bridge-ports`. Whether its `post-up` hooks fire reliably is
uncertain. vmbr0 definitely runs its own hooks, and by the time they run,
nic0 exists.

`post-up` = run this command after the interface comes up. Full path to
ethtool because the boot environment has a minimal PATH.

### 4. Verify after the next reboot
```
ethtool -k nic0 | grep -E "^tcp-segmentation-offload|^generic-segmentation|^generic-receive"
```
All three should read `off`. If they don't, the hook didn't fire and the
next option is a systemd unit instead — more reliable, more setup.

### If hangs continue anyway
Next lever is interrupt moderation:
```
ethtool -C nic0 rx-usecs 0
```
After that it's genuinely firmware or hardware. But disabling offloads
resolves the large majority of e1000e hang reports.

---

## PART 4 — COMMANDS

```
dmesg -T
dmesg -T | grep -i hang
dmesg -T | grep -iE "eth|link|nic0"
```
Kernel ring buffer — messages from the kernel and its drivers. `-T` prints
human-readable timestamps instead of seconds-since-boot.
**This is where hardware and driver faults appear, and nowhere else.**
Reach for it early on any hardware-adjacent problem.

```
ethtool -k <iface>          # SHOW features (lowercase k)
ethtool -K <iface> tso off  # SET a feature (capital K)
ethtool -C <iface> rx-usecs 0   # interrupt coalescing
ethtool <iface>             # link status, speed, duplex
```
Set features one at a time — batched changes can fail wholesale.

```
ip link set nic0 down
ip link set nic0 up
```
Bounce an interface. Clears driver-level stalls without a reboot.

```
modprobe -r e1000e && modprobe e1000e
```
Unload and reload a kernel module. `-r` = remove. Heavier than a link
bounce; only do it from the console, since it drops the interface.

### Interface naming on this host
```
2: nic0: ... 
   altname enp0s31f6
   altname enx98fa9b4c6520
```
- `nic0` — Proxmox's assigned name
- `enp0s31f6` — systemd's "predictable" name, derived from PCI location
  (**en**thernet, **p**ci bus 0, **s**lot 31, **f**unction 6)
- `enx98fa9b4c6520` — derived from the MAC address

All three refer to the same hardware. Tools may want different ones;
`ethtool` accepted `nic0` here.

### nano
- **Ctrl+O** — write out (save). Nano shows the filename pre-filled;
  press **Enter** to confirm.
- **Ctrl+X** — exit. If there are unsaved changes it prompts
  "Save modified buffer?" → `Y` → Enter to confirm the name.
- The "save under a different name?" prompt is just the filename
  confirmation. Press Enter with the path unchanged.

---

## PART 5 — LESSONS

### 48. `dmesg` first for anything hardware-adjacent
The kernel had been logging "Detected Hardware Unit Hang" every two
seconds, with timestamps, for days. Nobody looked. `ip link` describes
the interface's state NOW; `dmesg` describes what the kernel has been
complaining about over time. For intermittent faults, the log is worth
more than the current state.

### 49. "Reseating fixed it" does NOT prove a physical fault
Unplugging a cable forces a link-down/link-up event, which resets the
network adapter. That clears driver-level stalls just as effectively as it
fixes a loose connector. **Both look identical from the outside.**
If reseating fixes something, that narrows it to "something the link reset
cleared" — which includes driver bugs, not just connectors.

### 50. A brand-new part failing should trigger a rethink, not a replacement
When Rich said the cable was new, the working theory should have been
re-examined immediately. Instead the recommendation was still "replace the
cable." New parts do fail, but "the new part is also broken" is a much
weaker hypothesis than "the diagnosis is wrong."

### 51. Same misleading indicator, two different root causes
`LOWER_UP` meaning "carrier detected, not necessarily working" has now
produced two separate multi-hour debugging sessions in this lab — once for
a genuinely loose connector, once for a driver stall. When an indicator
has fooled you before, distrust it faster the second time.

### 52. `ethtool -K` may need one feature per command
The batched form failed with "could not change device features." Split
into separate invocations and all three applied. Worth remembering
generally: a tool rejecting a combined command doesn't mean the individual
operations are unsupported.

### 53. Runtime network tuning doesn't survive reboot
`ethtool` changes live only in the running kernel. Persisting them means
`post-up` hooks in `/etc/network/interfaces` (Debian/Proxmox), and
attaching them to a stanza that actually runs — the bridge, not the
bridge-member interface.

---

## PART 6 — OPEN ITEMS

Verify:
- [ ] No new "Hardware Unit Hang" entries over the next few days:
      `dmesg -T | grep -i hang | tail -5`
- [ ] After the next reboot, confirm the offloads stuck (see Part 3 step 4)
- [ ] Confirm the VM and OpenClaw gateway recovered from this outage:
      `sudo systemctl status openclaw-gateway.service --no-pager`
- [ ] The 2am dump should have dropped from 6.17 GB toward ~4.8 GB after
      the backup-disk change; delete the `pre-backup-disk` snapshot once
      confirmed

Still not done:
- [ ] SSH hardening — key auth, then `PermitRootLogin prohibit-password`,
      then fail2ban
- [ ] Decide whether to encrypt backups before upload
- [ ] Clean up /root/openclaw-telegram-repair-20260719T091122Z
- [ ] Identify the May 2026 "Cloudflare Agent Token"
- [ ] `disable_checksum = true` in rclone.conf
- [ ] Remove the leftover Debian ISO from VM 100's virtual CD drive
- [ ] Watch VM memory — 78% (6.27 of 8 GiB) after 4 days uptime

Hermes migration — inventory done, decisions pending:
- Real payload is only 3–4 GB. Most of the 26 GB on that box is cache,
  duplicates (two copies of the ASL project, two of torch), and an
  OpenClaw sweep that's now redundant since OpenClaw lives at home and is
  backed up.
- **That box runs four different things**, not just Hermes: Hermes itself,
  Apache serving *.richgibbs.dev, the live ASL app in /srv, and a trading
  bot touching Polymarket in /opt and /var/lib.
- Open questions: does the trading bot run unattended with real positions
  (if so, apartment WiFi is the wrong home for it — a $5/mo VPS is more
  appropriate), and does anything on *.richgibbs.dev need uptime?
- Also: OpenClaw's secrets were copied to that box during the pre-migration
  sweep. That's credential sprawl on an internet-facing host running a
  public web server. Worth cleaning up on AWS regardless of migration
  timing.
- RHEL 10.2 there maps cleanly to Rocky Linux 10 here — same family, same
  dnf, so the migration is much less fraught than crossing distro families.
