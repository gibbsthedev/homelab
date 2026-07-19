# Homelab Session 3 — Network Faults, DHCP Reservation, Cross-Host Access

Date: 2026-07-18 → 2026-07-19
Covers: a marginal ethernet connection that produced misleading symptoms,
pinning the VM's IP, OpenClaw's restart-loop breaker, and giving Hermes
(still on AWS) access to the home VM via Tailscale.

---

## PART 1 — STATE CHANGES THIS SESSION

- VM 100 (docker-agents) given a DHCP reservation on the Beryl AX:
  MAC bc:24:11:8e:4f:e2 → 192.168.8.10
  (IP had drifted .139 → .138 across reboots before this)
- Tailscale installed on the docker-agents VM and on the AWS Hermes host,
  joining both to the same tailnet so Hermes can reach the home VM
  despite NAT.
- Proxmox host DNS/connectivity restored after reseating the ethernet cable.

Still open:
- Disk resize 32→64 GB + swap-partition→swapfile (still deferred)

---

## PART 2 — COMMANDS USED

### Network diagnosis

```
ping -c 3 192.168.8.1     # the router, by IP — no DNS involved
ping -c 3 1.1.1.1         # internet, by IP — proves routing works
ping -c 3 github.com      # by name — only this one needs DNS
```
- `-c 3` = send exactly 3 packets and stop (otherwise ping runs forever).
- Pinging by IP vs. by name is how you separate a ROUTING problem from a
  DNS problem. If 1.1.1.1 answers but github.com doesn't, DNS is the fault.
- "Destination Host Unreachable" from your OWN address means no ARP reply
  came back — the target didn't answer at layer 2 at all.

```
ip addr show   # addresses per interface
ip link        # link state: UP / DOWN / NO-CARRIER
ip route       # routing table; look for the "default via" line
cat /etc/resolv.conf   # which DNS servers this machine is configured to use
```

```
systemctl status pve-cluster pvedaemon pveproxy --no-pager
```
- `--no-pager` prints straight to the terminal instead of opening a
  scrollable pager you have to `q` out of. Useful when you want to
  copy/photograph the output.
- pve-cluster provides the /etc/pve filesystem. If it's down, the web UI
  still loads (static files from pveproxy) but every API call fails —
  which shows up as an EMPTY REALM DROPDOWN on the login page.

### Service and log inspection

```
sudo systemctl status openclaw-gateway.service --no-pager
sudo journalctl -u openclaw-gateway -n 50 --no-pager
sudo journalctl -u openclaw-gateway --since "1 hour ago" --no-pager
sudo journalctl -u openclaw-gateway -f
```
- `journalctl -u <unit>` shows the logs for one specific service.
- `-n 50` = last 50 lines. `-f` = follow live (Ctrl+C to stop).
- `--since "1 hour ago"` accepts plain-English time expressions.

### Tailscale (cross-network access)

```
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
tailscale status
```
- Builds an encrypted mesh network between machines. Each node gets a
  stable 100.x address that works regardless of what physical network it's
  on — no port forwarding, no reverse tunnels, no dependence on the
  apartment router.
- `tailscale up` prints a URL you open in ANY browser to authenticate;
  it doesn't have to be a browser on that machine.
- Both machines must join the SAME account/tailnet.
- Free tier covers 100 devices.
- Trade-off worth knowing: Tailscale operates the coordination server that
  helps nodes find each other. Traffic is end-to-end encrypted and usually
  peer-to-peer, but you are trusting a third party with the control plane.

### The reverse-tunnel alternative (not used, but worth knowing)

```
ssh -i ~/aws-key.pem -R 2222:localhost:22 ubuntu@54.197.26.209 -N
```
- `-R 2222:localhost:22` = "open port 2222 on the REMOTE host and forward
  anything arriving there back to port 22 on me."
- Works through NAT because the machine behind NAT makes the OUTBOUND
  connection; the remote side then rides that established channel back in.
- `-N` = don't run a command, just hold the tunnel open.
- This is the pattern AWS was using before (tunnels to 35.169.141.91:22022).
  Tailscale replaces it more robustly.

---

## PART 3 — LESSONS LEARNED

### 22. LOWER_UP means "carrier detected", NOT "the connection works"
Symptom: Proxmox host couldn't ping the router (192.168.8.1) or anything
beyond it. `git pull` failed with "Temporary failure in name resolution".
But `ip link` showed the NIC UP with LOWER_UP, `vmbr0` had the correct
192.168.8.2/24, and `ip route` had a valid default gateway. Everything in
software looked perfect.
Cause: the ethernet cable was seated just well enough to negotiate link
but not well enough to pass traffic reliably.
Fix: unplug and firmly reseat the cable at both ends.
Takeaway: the physical layer can lie. When the software stack is
demonstrably correct and nothing passes, suspect the wire BEFORE the
config. This also explains an earlier "it fixed itself" episode — it was
the same marginal connection failing intermittently, not a router blip.

### 23. Symptoms cascade — chase the lowest layer first
The visible failures were, in order: git pull fails → DNS resolution
fails → can't ping the router → can't reach Proxmox web UI → Realm
dropdown empty. Every one of those was a downstream symptom of ONE loose
cable. Diagnosing top-down (git, then DNS, then services) wasted time.
Bottom-up (link → routing → DNS → application) would have found it faster.

### 24. Test both directions to localize a fault
The host could SSH to the VM at .138 but couldn't reach the router at .1.
That asymmetry was the key clue: VM traffic stays inside the Proxmox
bridge and never touches the physical NIC, so it working proved the
software stack was fine while saying nothing about the cable.
Generalizable test: unplug the suspect cable from the server and plug it
into a laptop. If the laptop gets an address, the cable and switch port
are good and the server is at fault. If not, it's the cable or port.
That single test splits the problem in half.

### 25. docker0 showing NO-CARRIER is normal
Docker's virtual bridge sits DOWN with NO-CARRIER whenever no containers
are running — nothing is attached to it. Same flag, completely different
meaning than on a physical NIC. Don't chase it.

### 26. GL.iNet showing a static-IP device as "offline" is not diagnostic
The router marks clients offline based on DHCP lease activity. The
Proxmox host uses a STATIC IP and never talks to the router's DHCP
server, so it can be fully functional and still show as offline. Same
quirk as the earlier "0 LAN clients" confusion.

### 27. DHCP reservations: pick an address below the pool
Set on the router: Network → LAN → Address Reservation, bind MAC to IP.
Observed leases were landing in the .130–.226 range, so .10 sits safely
clear of the pool.
The reservation does NOT take effect until the client renews its lease —
reboot the VM (`qm reboot 100`) and confirm with `ip addr show`.
Worth recording the pool's start/end in the network docs; it constrains
every future static or reserved assignment.

### 28. OpenClaw's restart-loop breaker (why Telegram went silent)
Symptom: `systemctl status openclaw-gateway` showed active (running), the
VM had working internet (pings to 1.1.1.1 and api.telegram.org both
succeeded), but Telegram stopped responding.
Log line:
  "restart-loop breaker tripped: 11 unclean boot(s) within 300000ms;
   suppressing channel/provider account auto-start."
Cause: while the ethernet was faulty, the gateway repeatedly started with
no network, crashed, and got restarted by systemd — 11 times in 5 minutes.
OpenClaw's safety mechanism then DELIBERATELY refused to start the
messaging channels.
Key insight: the gateway process comes up and serves HTTP (so systemctl
looks healthy) while the channel layer is intentionally suppressed. A
"running" service is not necessarily a working one.
Clearing it: stop the service, wait out the 5-minute window with zero
restarts, then start it fresh. `restart` may not reset a time-based
counter; `stop` → wait → `start` does.

### 29. A user account is authorization, not reachability
Wanting to give Hermes (on AWS) access to the home VM, the instinct was
"create a user, grant sudo, revoke it later." The account part is fine and
reversible. But it doesn't help on its own: the VM is behind NAT on
apartment WiFi with no inbound access, so AWS has no route to it
regardless of credentials. That gap needs Tailscale, a reverse tunnel, or
port forwarding — and port forwarding isn't available on this network.
Takeaway: separate "can it connect" from "is it allowed" when debugging
access problems. They fail differently and need different fixes.

### 30. Don't reach for a big tool before the cheap checks
The gateway token mismatch appeared right after a session where the shell
was sitting at `ubuntu@docker-agents:/home/rich` — the ubuntu user in
rich's home directory. A config lookup keyed to $HOME or the working
directory would produce exactly that error. Two free checks (`cd ~`, and
`openclaw gateway --help` for a token flag) were worth trying before
standing up cross-network access to let an agent fix it.

### 31. Two service SCOPES can start the same daemon (root cause of #28)
Symptom: Telegram dead, gateway crash-looping, breaker tripped, and a
"gateway token mismatch" from the CLI.
Root cause: TWO OpenClaw gateways were competing for port 18789.
  - An accidental USER-scope unit under `rich` owned the port.
  - The intended SYSTEM-scope unit running as `ubuntu` kept failing with
    EADDRINUSE.
  - 11 failed starts tripped the crash-loop breaker → Telegram suppressed.
The tell was there earlier and got misread: we saw EADDRINUSE once during
the original cutover and read it as "the migrated unit auto-started it,
all good." It was actually the first sign of a second start path.
How this happened: the user-scope unit came over in the migration from
AWS (where it was a `--user` unit), and we then created/enabled a
system-scope unit here. Both auto-started.

Enumerate EVERY way a service can start:
```
systemctl list-units --all | grep -i <name>
systemctl --user list-units --all | grep -i <name>
sudo -u <user> XDG_RUNTIME_DIR=/run/user/<uid> systemctl --user list-units --all | grep -i <name>
loginctl show-user <user> | grep -i linger
```
- Checking `systemctl --user` as YOURSELF only shows YOUR user's units.
  You have to check each user on the box separately.
- **Lingering** is what allows a user-scope unit to run without that user
  being logged in. `loginctl disable-linger <user>` stops that.

Find who actually owns a port:
```
sudo ss -ltnp | grep <port>
```
- `ss` lists sockets: `-l` listening, `-t` TCP, `-n` numeric, `-p` show
  the owning process (PID and user). This is the fastest way to answer
  "what is already using this port and who runs it."

Fix applied (by Hermes): backed up both service definitions first, then
DISABLED (not just stopped) the duplicate `rich` gateway, restarted the
intended `ubuntu` system gateway, cleared the failed state, and manually
restarted Telegram via `channels.start`. Verified with an actual
round-trip message, not just "service is running."

Takeaway: `disable` vs `stop` matters — stop is until the next boot,
disable is permanent. And when verifying a fix, test the FUNCTION
(send a message) not just the STATUS (systemctl says active).

FOLLOW-UP (2026-07-19): checked and closed the underlying mechanism.
```
sudo -u rich XDG_RUNTIME_DIR=/run/user/1000 systemctl --user is-enabled openclaw-gateway
  → disabled
loginctl show-user rich | grep -i linger
  → Linger=yes          ← this was the real enabler
sudo loginctl disable-linger rich
loginctl show-user rich | grep -i linger
  → Linger=no
```
Why lingering mattered: normally a user's systemd instance starts at
login and dies at logout, so `--user` units only run while that user is
actually on the box. LINGERING overrides that — it keeps the user manager
alive from boot, which is how a user-scope service ends up running on a
headless server with nobody logged in. This setting almost certainly came
across from the AWS host, where the gateway legitimately WAS a user unit
and needed lingering to survive.
Checking `is-enabled` alone would have missed this. The unit was already
disabled, but with Linger=yes the door stays open if anything re-enables
it. Disabling linger is belt-and-braces.

Remaining proof: after the next reboot, `sudo ss -ltnp | grep 18789`
should show exactly ONE process, owned by ubuntu.

---

## PART 4 — OPEN ITEMS

Verify soon:
- [x] Telegram repaired — duplicate `rich` gateway disabled, `ubuntu`
      system gateway restarted, round-trip message confirmed (msg 7045/7048)
- [x] VM confirmed at 192.168.8.10 after the reservation + reboot
- [x] Lingering disabled for `rich` (was Linger=yes — the mechanism that
      let a user-scope unit run without a login)
- [ ] Confirm the duplicate stays gone across a REBOOT:
      `sudo ss -ltnp | grep 18789` should show exactly one process

Security hygiene:
- [x] Tailscale SSH disabled on the VM; Hermes host logged out of the
      tailnet (no Tailscale IP, reports NeedsLogin)
- [ ] Eventually clean up the repair backup at
      /root/openclaw-telegram-repair-20260719T091122Z

Still deferred:
- [ ] Disk 32 → 64 GB, snapshot FIRST, replace /dev/sda5 swap partition
      with a 4 GB swapfile (see lesson #21)
- [ ] Backups — still none. Highest-value outstanding item.
- [ ] Hermes migration proper (inventory first, then size the hardware)

Watch for:
- Whether the ethernet fault recurs. If the link drops again, replace the
  cable rather than reseating it.
