# Homelab Session 2 — Docker VM + OpenClaw Migration

Date: 2026-07-16 → 2026-07-18
Covers: creating the first VM, installing Docker, migrating OpenClaw
(12 agents, 6.4 GB of state) off AWS onto the home server.

---

## PART 1 — WHAT EXISTS NOW (current state of the lab)

### pve-node1 (the Proxmox host)
- Lenovo ThinkCentre M720q Tiny, i5-8400T (6 cores), 2x8 GB DDR4-2666,
  256 GB NVMe (Samsung PM981), empty 2.5" SATA bay
- Proxmox VE 9.2 (Debian 13 Trixie base)
- Static IP 192.168.8.2, web UI https://192.168.8.2:8006
- Login: root / password in Apple Passwords

### Network
- GL.iNet Beryl AX (GL-MT3000) in repeater / WiFi-as-WAN mode
- Uplink: WhiteSky-Prose building WiFi (managed, client-isolated)
- Private LAN: 192.168.8.x, router admin http://192.168.8.1
- Apartment ethernet wall jack is DEAD (NO-CARRIER, confirmed 2 cables)

### VM 100 — docker-agents
- Debian 13.6 (Trixie)
- 4 vCPU (type: host), 8 GB RAM, 32 GB disk on local-lvm
- MAC bc:24:11:8e:4f:e2 — DHCP (should get a reservation; IP has moved once)
- qemu-guest-agent installed and running
- Users:
  - root — password in Apple Passwords
  - rich (UID 1000) — groups: sudo, docker — password in Apple Passwords
  - ubuntu (UID 1001) — runs OpenClaw — password in Apple Passwords
- Software: Docker CE, Node.js v22, npm, OpenClaw, rsync, curl, git

### OpenClaw (migrated from AWS, LIVE on home server)
- Installed as npm global in /home/ubuntu/.npm-global/bin/openclaw
- All state in /home/ubuntu/.openclaw (6.4 GB, 34 subfolders)
- 12 agents: tuck (primary/overseer, xai/grok-4.3), anchor, bastion,
  branch, brief, cipher, designer, forge, ink, pitch, programmer,
  researcher, protocol-enforcer
- 13 plugins, 23 skills
- Gateway runs as a SYSTEM service: /etc/systemd/system/openclaw-gateway.service
  (NOTE: on AWS it was a --user unit; here it needs `sudo systemctl`)
- Gateway port: 127.0.0.1:18789

### AWS (stopped, kept as fallback)
- OpenClaw box: 54.197.26.209 (ubuntu@, key openclaw-key.pem)
- Gateway stopped and disabled — instance intact, NOT terminated
- Keep as fallback for several days before decommissioning
- Still on AWS: Hermes (separate host), ASL Translator runtime,
  website host 35.169.141.91, reverse SSH tunnels

---

## PART 2 — COMMANDS USED, AND WHY

### Creating and inspecting VMs (run on the Proxmox HOST only)

```
qm list
```
- `qm` = **q**emu **m**anager, Proxmox's VM control command.
- Lists VMs with VMID, name, status, allocated RAM, disk, and PID.
- Only exists on the Proxmox host — inside a VM it's "command not found."

```
qm config 100 | grep -E "cores|sockets|cpu"
```
- Shows the full config of VM 100, filtered to the CPU lines.
- Useful to confirm what a VM is *actually* allocated vs. what you remember.

### VM resource choices (and the reasoning)

- **CPU type `host`** — passes the real CPU's features through to the guest
  for best performance. The default (`kvm64`) is a slow generic
  compatibility mode. Correct choice for a single-node homelab.
- **Cores 4** — bumped from 3 because sub-agent deployment is CPU-bursty.
  Left 2 cores of the 6 for Proxmox + the future Hermes VM.
- **RAM 8 GB** — OpenClaw uses ~1.3–2.1 GB in practice. 8 GB is ~4x that.
- **Disk 32 GB** — currently 6.4 GB used. Resize deferred (see swap note).
- **Bridge vmbr0 / VirtIO** — the fast paravirtualized network default.

KEY PRINCIPLE — **RAM cannot be safely oversubscribed; CPU can.**
RAM assigned to a VM is physically reserved and unavailable to anything
else. Over-assigning risks host OOM (Linux starts killing processes).
vCPUs are time-shared, so over-assigning causes contention/slowness, not
failure. Never commit more than ~75-80% of RAM across all VMs.

### Debian install choices
- Text installer ("Install", not graphical) — lighter, reliable.
- Partitioning: "Guided - use entire disk" → "All files in one partition".
  Only touches the VM's virtual disk, never the real NVMe.
- Software selection: **UNcheck** desktop environment (servers don't need
  a GUI), **CHECK** SSH server + standard system utilities.
- GRUB: install to /dev/sda (the only disk).

### Users and permissions

```
su -
```
- **s**witch **u**ser; with no name, switches to root. The `-` gives root's
  full environment (correct PATH etc.). Always use `su -`, not bare `su`.

```
sudo usermod -aG sudo,docker rich
```
- `usermod` modifies an account. `-aG` = **a**ppend to **G**roup(s).
- The `-a` is critical: without it you *replace* the user's group list
  and can lock them out of groups they need.
- Group changes take effect on the user's NEXT LOGIN.

```
sudo useradd -m -s /bin/bash ubuntu
sudo passwd ubuntu
```
- `-m` creates the home directory, `-s` sets the login shell.
- Created to match the AWS username (see the /home/ubuntu lesson below).

```
sudo chown -R ubuntu:ubuntu /home/ubuntu/.openclaw
```
- `chown` = change **own**er. `-R` = recursive (everything inside too).
- Format is `user:group`. Needed because the files arrived owned by rich.

```
chmod 600 ~/aws-key.pem
```
- Owner read/write only, nobody else. SSH REFUSES to use a key file that
  others can read — this isn't optional.
- `chmod 0755 -d` (as used for /etc/apt/keyrings) = create a directory
  with owner-write, everyone-read/execute.

### Reading /etc/passwd

Format: `name:x:UID:GID:GECOS:home:shell`
- Field 2 `x` = "password hash lives in /etc/shadow" (which is root-only).
  An actual hash sitting in /etc/passwd is a red flag.
- **UID 0 = root.** Privilege comes from the NUMBER, not the name. A
  second account with UID 0 is a classic backdoor.
- UID < 1000 = system accounts; 1000+ = human users.
- Shell `/usr/sbin/nologin` on system accounts blocks interactive login.
  A system account that has quietly gained /bin/bash is a persistence
  technique worth auditing for.

Find any UID-0 accounts:
```
awk -F: '$3 == 0 {print $1}' /etc/passwd
```
- `-F:` splits fields on `:`; `$3` is the third field (UID); `$1` is the name.
- A healthy system returns exactly one line: root.

### Installing Docker (official repo method)

```
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
chmod a+r /etc/apt/keyrings/docker.asc
```
- Adds Docker's GPG key so apt trusts their packages.
- `curl -fsSL`: **f**ail silently on errors, **s**ilent, **S**how errors
  anyway, **L**ocation (follow redirects). `-o` writes to a file.

```
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | tee /etc/apt/sources.list.d/docker.list > /dev/null
```
- Writes the Docker repo definition.
- `$(...)` runs a command and substitutes its output — here it auto-detects
  the architecture (amd64) and the Debian codename (trixie).
- `tee` writes to a file AND to stdout; `> /dev/null` discards the stdout
  copy so it doesn't spam the terminal.

```
apt update
apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
docker run hello-world
```

### Node.js + OpenClaw (per-user global install, matching AWS)

```
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt install -y nodejs
```
- `sudo -E` preserves the environment when running the setup script.

```
mkdir -p ~/.npm-global
npm config set prefix ~/.npm-global
echo 'export PATH=~/.npm-global/bin:$PATH' >> ~/.bashrc
source ~/.bashrc
npm install -g openclaw      # NO sudo — installs into the user's home
```
- Setting the npm `prefix` makes "global" mean "global for this user,"
  landing in ~/.npm-global/bin exactly like the AWS layout.
- `>>` appends to a file (`>` would overwrite it).
- `source ~/.bashrc` reloads the shell config so PATH applies immediately.

### The migration transfer

```
scp C:\Users\rgibb\.ssh\openclaw-key.pem rich@192.168.8.139:/home/rich/aws-key.pem
```
- `scp` = **s**ecure **c**o**p**y, file transfer over SSH. Run from the
  laptop; authenticates to the VM with rich's password.

```
ssh -i ~/aws-key.pem ubuntu@54.197.26.209 "echo connected; hostname; df -h ~"
```
- `-i` selects an **i**dentity file (private key).
- Putting a command in quotes runs it remotely and exits — good for tests.

```
rsync -avz --progress -e "ssh -i ~/aws-key.pem" ubuntu@54.197.26.209:~/.openclaw/ ~/.openclaw/
```
- `-a` archive mode: recursive AND preserves permissions, ownership,
  timestamps, symlinks. Essential — the secret folders are drwx------
  and must stay that way.
- `-z` compress in transit (faster over slow WiFi).
- `--progress` show a progress bar.
- `-e` specify the remote shell command (here, ssh with a specific key).
- **Trailing slashes matter**: `src/` means "the contents of src".
- rsync only transfers what's different, so re-running resumes an
  interrupted copy and makes a fast final delta sync.

WHY PULL, NOT PUSH: AWS is public and can be reached from anywhere. The
home VM is behind NAT on apartment WiFi with no inbound access. So the
VM must INITIATE the connection (outbound works) and pull the data down.

### Service control

```
sudo systemctl stop openclaw-gateway.service
sudo systemctl status openclaw-gateway.service
systemctl is-enabled docker
```
- `systemctl` manages services. `enable` = start at boot,
  `start` = start now, `enable --now` = both.
- **System units** (/etc/systemd/system/) need root/sudo and run at boot
  regardless of login. **User units** (`systemctl --user`) are tied to a
  login session. OpenClaw is a system unit here but was a user unit on
  AWS — so use `sudo systemctl`, not `systemctl --user`.
- Press `q` to exit a `status` view.

### Diagnostics

```
free -h            # RAM: total / used / free / available (-h = human readable)
nproc              # number of CPU cores available
top                # live CPU/RAM by process; q to quit
df -h /            # disk space on the root filesystem
lsblk              # block devices and partitions
du -sh <dir>       # total size of a directory (-s summary, -h human)
du -sh dir/* | sort -h | tail -15    # the 15 biggest items in a directory
ip addr show       # IP addresses per interface
ip link            # interface link state (UP / DOWN / NO-CARRIER)
```

```
grep -rl "/home/ubuntu" ~/.openclaw/ | wc -l
```
- `grep -r` recursive, `-l` list only the filenames that match.
- `wc -l` = **w**ord **c**ount, `-l` for lines → counts matching files.
- This is how we found 20,062 files with hardcoded paths.

---

## PART 3 — LESSONS LEARNED (this session)

### 12. Docker's apt repo must be added BEFORE `apt update`
Symptom: "Package docker-ce has no installation candidate."
Cause: the repo-add step (the long echo|tee line) silently didn't run, so
apt had no idea where to find Docker. The `apt update` output showed only
the three Debian repos — no download.docker.com line.
Fix: re-run the repo add, then `cat /etc/apt/sources.list.d/docker.list`
to VERIFY before updating.
Takeaway: after adding a repo, always confirm it appears in `apt update`
output. Long one-liner pastes are the usual culprit when they don't.

### 13. Minimal Debian is missing common tools
sudo, rsync, curl were all absent on a fresh minimal install. That's by
design (smaller = more secure). Install what you need as you hit it.
Corollary: as a plain user you can't `apt install` — you need root, and
if sudo isn't installed yet, use `su -`.

### 14. The Proxmox noVNC console is for emergencies, not daily work
It won't resize, can't scroll, and mangles pasted text. Use it for the OS
install and for recovery when the network is down. For everything else,
SSH in — bigger window, scrollback, working copy/paste.

### 15. MIGRATION GOTCHA: hardcoded /home/ubuntu paths (the big one)
Symptom: `openclaw doctor` → "EACCES: permission denied, mkdir /home/ubuntu"
Cause: AWS ran as user `ubuntu`; the VM's user is `rich`. The migrated
config had absolute paths baked in. `grep -rl "/home/ubuntu"` found
**20,062 files** — including agent memory, dream logs, and sqlite DBs.
Fix (Option A, chosen): recreate the environment the app expects —
create an `ubuntu` user, move .openclaw to /home/ubuntu/, chown -R to
ubuntu, install OpenClaw as ubuntu into ~/.npm-global. All 20k paths then
resolve untouched.
Rejected (Option B): mass find-and-replace across 20k files. Miss one
path and you get subtle breakage later, possibly in agent memory.
Takeaway: when migrating between machines with different usernames,
recreating the original username is usually safer than rewriting paths.

### 16. "Permission denied" can be proof that security is WORKING
After chowning .openclaw to ubuntu, running `du -sh` as rich failed.
That's correct — drwx------ means owner-only. The earlier `sudo ls`
worked because sudo runs as root. Not every red error is a problem.

### 17. EADDRINUSE on startup = it's ALREADY running
"another gateway instance is already listening on 18789" meant the
migrated systemd unit had auto-started the gateway on boot. The migration
had already succeeded; the manual start was redundant.

### 18. System units vs. user units
On AWS the gateway was a `--user` unit (tied to the login session). On
the VM it's a system unit in /etc/systemd/system/ (starts at boot, needs
sudo). The error message told us exactly this. System units are the more
robust choice for an always-on server.

### 19. DHCP IPs move; servers need reservations
After the shutdown/restart for the CPU change, the VM's IP changed and
SSH stopped working. Recovery: Proxmox Console → `ip addr show` to find
the new address. Permanent fix: DHCP reservation on the router bound to
the VM's MAC (bc:24:11:8e:4f:e2), or a static IP inside Debian.

### 20. An agent speccing its own machine will over-ask
OpenClaw requested 6 vCPU / 24 GB RAM / 100 GB disk. It ran in a 3.7 GB
AWS box using ~2.1 GB. 24 GB would have reserved ~22 GB of idle memory
and starved Proxmox + the planned Hermes VM.
BUT it was RIGHT about CPU — sub-agent deployment is genuinely
CPU-bursty, which was the actual bottleneck. And it caught a real error
in the plan (see #21).
Takeaway: agents optimize for their own unconstrained operation; they
can't see the other tenants. Allocation judgment is the infra owner's
job. Take the diagnosis seriously, size it yourself.

### 21. DON'T blindly `growpart` — check partition order first
Planned disk resize from 32→64 GB. New space is always added at the END
of a disk, and a partition can only grow into free space DIRECTLY after
it. The layout here is /dev/sda1 (root) then /dev/sda5 (swap), so swap
sits between root and the new space. A blind growpart would fail or do
damage.
Correct sequence (deferred, not yet done): snapshot the VM first →
grow the virtual disk in Proxmox → swapoff → remove the swap partition →
growpart + resize2fs on root → replace swap with a 4 GB SWAPFILE.
A swapfile lives inside the root filesystem, so it never blocks future
disk growth. This permanently removes the partition-ordering constraint.

---

## PART 4 — OPEN ITEMS / NEXT STEPS

Immediate:
- [ ] DHCP reservation for the VM (MAC bc:24:11:8e:4f:e2) — IP has moved once
- [ ] Measure sub-agent deploy performance at 4 vCPU; escalate to 6 if it
      still stalls (watch `top` during a deploy)
- [x] Delete aws-key.pem from the VM (done)

Deferred (needs a careful, snapshotted session):
- [ ] Disk resize 32 → 64 GB + convert swap partition to a 4 GB swapfile
      with low swappiness. NOT urgent: 6.4 GB used of 32 GB.

Phase 2 — Hermes:
- [ ] Build a Rocky Linux VM (RHEL-compatible, dnf, free)
- [ ] Inventory the AWS Hermes host the same way we did OpenClaw
- [ ] Migrate, reconnect to OpenClaw on the home network

Phase 3 — ASL runtime + website + tunnels:
- [ ] Most entangled piece; replace the fragile reverse-SSH tunnel
      (to 35.169.141.91:22022) with Cloudflare Tunnel
- [ ] Then decommission AWS entirely (deadline: November)

Known pre-existing issue (NOT caused by the migration):
- Voyage embeddings broken — missing API key. Was already broken on AWS.
