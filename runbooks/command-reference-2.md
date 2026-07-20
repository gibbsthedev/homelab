# Command Reference — Part 2

Everything used since the first reference doc (which covered hardware
inspection, the USB installer, apt/repo fixes, and initial Git setup).

This part covers: network troubleshooting, service and session management,
systemd scopes, Tailscale, rclone/Cloudflare R2, and Proxmox backups.

Organized by topic rather than chronologically, so it reads as a reference
rather than a diary.

---

## 1 — NETWORK TROUBLESHOOTING

The single most useful principle in this whole section: **diagnose
bottom-up.** Link → routing → DNS → application. One loose cable produced
five different-looking failures (git pull broken, DNS resolution broken,
router unreachable, web UI unreachable, an empty login dropdown). Starting
at the application layer wasted an hour; starting at the wire would have
found it in two minutes.

```
ping -c 3 192.168.8.1        # the router, by IP
ping -c 3 1.1.1.1            # the internet, by IP
ping -c 3 github.com         # by NAME
```
- `-c 3` sends exactly 3 packets and stops. Without it ping runs forever.
- WHY three separate targets: this is how you split a ROUTING problem from
  a DNS problem. If 1.1.1.1 answers but github.com doesn't, the network is
  fine and only name resolution is broken. If neither answers but the
  router does, you have internet routing trouble. If the router doesn't
  answer, the problem is local.
- "Destination Host Unreachable" coming FROM YOUR OWN ADDRESS means no ARP
  reply came back — the target never answered at layer 2 at all.

```
ip addr show      # addresses assigned to each interface
ip link           # LINK STATE only: UP / DOWN / NO-CARRIER
ip route          # routing table — look for the "default via" line
cat /etc/resolv.conf   # which DNS servers this machine is configured to use
```
- WHY `ip link` matters separately from `ip addr`: an interface can have a
  perfectly correct IP address while the physical link is dead. These are
  different layers and they fail independently.
- **CRITICAL GOTCHA:** `LOWER_UP` means a carrier signal was detected. It
  does NOT mean the connection works. A partially-seated RJ45 can hold
  enough contact to negotiate link while dropping most frames. The physical
  layer can lie. When the software stack is demonstrably correct and
  nothing passes, suspect the wire.
- `NO-CARRIER` on a PHYSICAL nic = nothing on the other end (dead jack,
  unplugged cable). `NO-CARRIER` on `docker0` or another VIRTUAL bridge is
  NORMAL — it just means no containers are attached. Same flag, completely
  different meaning. Don't chase it.

### Localizing a physical fault
Two tests that split the problem in half:

1. **Compare paths that use different hardware.** The host could SSH to
   the VM but couldn't reach the router. VM traffic stays inside the
   Proxmox bridge and never touches the physical NIC — so that working
   proved the software stack was fine while saying nothing about the cable.
2. **Swap the endpoint.** Unplug the suspect cable from the server and put
   it in a laptop. If the laptop gets an address, the cable and switch port
   are good and the server is at fault. If not, it's the cable or the port.

### Proxmox-specific: an empty Realm dropdown at login
```
systemctl status pve-cluster pvedaemon pveproxy --no-pager
ls /etc/pve
df -h /
```
- `--no-pager` prints straight to the terminal instead of opening a
  scrollable pager you have to `q` out of. Use it whenever you want to
  copy or photograph output.
- WHY these three services: `pve-cluster` provides the /etc/pve filesystem
  (pmxcfs) where realm definitions live. If it's down, the web UI still
  loads — those are static files served by `pveproxy` — but every API call
  behind it fails. An empty Realm dropdown is the classic symptom.
- `ls /etc/pve` should list storage.cfg, user.cfg, nodes/. Empty means
  pmxcfs isn't mounted.
- `df -h /` because a FULL root filesystem breaks pmxcfs in exactly this way.

---

## 2 — SERVICES, LOGS, AND PORTS

```
sudo systemctl status <unit> --no-pager
sudo systemctl start|stop|restart <unit>
sudo systemctl enable|disable <unit>
sudo systemctl enable --now <unit>
systemctl is-enabled <unit>
```
- `start`/`stop` act NOW. `enable`/`disable` control whether it runs AT
  BOOT. `enable --now` does both.
- **`disable` vs `stop` matters.** A stopped service comes back on the next
  reboot. A disabled one doesn't. When removing something permanently,
  disable it.

```
sudo journalctl -u <unit> -n 50 --no-pager
sudo journalctl -u <unit> --since "1 hour ago" --no-pager
sudo journalctl -u <unit> -f
sudo journalctl -u <unit> | grep -iE "error|crash|fatal"
```
- `-u` scopes to one unit. `-n 50` = last 50 lines. `-f` = follow live
  (Ctrl+C to stop). `--since` accepts plain English like "1 hour ago",
  "yesterday", "2026-07-19".
- `grep -i` = case-insensitive, `-E` = extended regex so `|` means OR.

```
sudo ss -ltnp | grep 18789
```
- `ss` lists sockets. `-l` listening, `-t` TCP, `-n` numeric ports (don't
  resolve to service names), `-p` show the owning process.
- WHY this is the fastest way to answer "what is already using this port
  and who runs it."
- **Reading the output:** two lines showing `127.0.0.1:18789` and
  `[::1]:18789` with the SAME pid is normal — one process binding both the
  IPv4 and IPv6 loopback stacks. Two DIFFERENT pids on the same port is the
  broken case.

```
ps -eo pid,user,cmd | grep -i openclaw | grep -v grep
ps -o pid,user,cmd -p 6817
ps -ft pts/1
```
- `-e` every process, `-o` choose which columns to print.
- `grep -v grep` excludes the grep command itself from its own results —
  otherwise you always see one phantom match.
- `-p <pid>` inspects one process. `-ft pts/N` shows what owns a terminal.

### A running service is not necessarily a working one
`systemctl status` showed `active (running)` while OpenClaw had
deliberately suppressed its Telegram channel. The gateway process was up
and serving HTTP; the messaging layer was intentionally off. Always verify
by FUNCTION (send a message, read a file back) rather than by STATUS.

---

## 3 — SYSTEMD SCOPES AND LINGERING

This caused a genuinely subtle outage and is worth understanding properly.

Systemd has **system-scope** units and **user-scope** units:
- System scope lives in `/etc/systemd/system/`, needs root/sudo to control,
  and starts at boot regardless of whether anyone logs in.
- User scope is per-user, controlled with `systemctl --user`, and normally
  starts at login and dies at logout.

Two gateways were competing for port 18789: a user-scope unit under `rich`
owned the port, and the intended system-scope unit under `ubuntu` kept
failing with EADDRINUSE. Eleven failed starts in five minutes tripped
OpenClaw's crash-loop breaker, which then suppressed Telegram.

The user unit came across in the migration from AWS (where it legitimately
WAS a user unit); a system unit was then created here. Both auto-started.

```
systemctl list-units --all | grep -i <name>
systemctl --user list-units --all | grep -i <name>
sudo -u rich XDG_RUNTIME_DIR=/run/user/1000 systemctl --user list-units --all | grep -i <name>
```
- **`systemctl --user` as yourself only shows YOUR OWN user's units.** To
  check another user you must run it as them, and set `XDG_RUNTIME_DIR` to
  `/run/user/<their-uid>` so systemd can find their session bus. UIDs come
  from `/etc/passwd` — 1000 is typically the first human user.

```
loginctl show-user rich | grep -i linger
sudo loginctl disable-linger rich
```
- **Lingering** is what lets a user-scope unit run on a headless server
  with nobody logged in. It keeps the user's systemd manager alive from
  boot. This was the actual enabling mechanism, and almost certainly came
  from the AWS host where it was needed.
- Checking `is-enabled` alone would have MISSED this. The unit was already
  disabled; with `Linger=yes` the door stays open if anything re-enables it.

**Takeaway:** when enumerating what can start a service, check system
scope, user scope FOR EVERY USER ON THE BOX, and lingering.

---

## 4 — LOGIN SESSIONS

```
who
who -u
w
tty
```
- `w` is the most informative. Reading it:
  - **FROM** column shows the source IP. A `-` means it is NOT an SSH
    session — it's a local console (Proxmox web shell or a physically
    attached screen).
  - **WHAT** shows what that session is running. The session running `w`
    is you.
  - **IDLE** shows how long it's been untouched.
- `tty` prints which PTS you are currently on — so you don't kill yourself.

```
sudo pkill -t pts/1          # SIGTERM — graceful, the default, preferred
sudo pkill -9 -t pts/1       # SIGKILL — immediate, no cleanup
loginctl terminate-user root # kills ALL sessions for that user
```
- WHY prefer plain `pkill`: SIGTERM asks a process to exit and lets it
  clean up. `-9` (SIGKILL) cannot be caught or ignored — the process is
  destroyed mid-whatever. Reach for `-9` only when something ignores the
  polite request, not as a default.
- **`loginctl terminate-user root` will disconnect YOU** if you're logged
  in as root. Expect to be dropped.

### Auditing accounts — /etc/passwd
Format: `name:x:UID:GID:GECOS:home:shell`

```
grep -E ":/home/" /etc/passwd
awk -F: '$3 == 0 {print $1}' /etc/passwd
```
- `awk -F:` splits each line on `:`. `$3` is the third field (UID), `$1` is
  the name. So this prints the name of every account with UID 0.
- **UID 0 IS root.** Privilege comes from the NUMBER, not the name. A
  healthy system returns exactly one line. Anything else is a finding — an
  innocuously-named account with UID 0 is a classic backdoor.
- Field 2 being `x` means the password hash lives in `/etc/shadow`, which
  is root-only. An actual hash sitting in world-readable /etc/passwd is a
  red flag.
- A shell of `/usr/sbin/nologin` on system accounts blocks interactive
  login. A dormant system account that has quietly gained `/bin/bash` is a
  persistence technique worth auditing for.

---

## 5 — TAILSCALE (cross-network access)

The problem it solves: AWS is public and reachable from anywhere. The home
VM sits behind NAT on apartment WiFi with no inbound access and no ability
to port-forward. So AWS cannot open a connection to it, regardless of
credentials.

**A user account is AUTHORIZATION. This was a REACHABILITY problem.** Those
fail differently and need different fixes. Creating a user with sudo
wouldn't have helped at all.

```
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
tailscale status
```
- Builds an encrypted mesh between machines. Each node gets a stable 100.x
  address that works regardless of the physical network it's on. No port
  forwarding, no reverse tunnels, no dependence on the router.
- `tailscale up` prints a URL to authenticate. It can be opened in ANY
  browser — it doesn't have to be on that machine. Useful when the machine
  is headless.
- Both machines must join the SAME account/tailnet.
- **Trade-off worth knowing:** Tailscale operates the coordination server
  that helps nodes find each other. Traffic is end-to-end encrypted and
  usually peer-to-peer, but you are trusting a third party with the
  control plane.

### The reverse-tunnel alternative (not used, but understand it)
```
ssh -i ~/aws-key.pem -R 2222:localhost:22 ubuntu@<public-host> -N
```
- `-R 2222:localhost:22` = "open port 2222 on the REMOTE host, and forward
  anything arriving there back to port 22 on me."
- WHY it defeats NAT: the machine BEHIND the NAT makes the OUTBOUND
  connection. The remote side then rides that already-established channel
  back in. Outbound works; inbound doesn't; so you invert the direction.
- `-N` = don't run a command, just hold the tunnel open. Add `-f` to
  background it, or run it in tmux so it survives your session.
- This is the pattern the AWS setup was already using. Tailscale replaces
  it more robustly.

---

## 6 — RCLONE AND CLOUDFLARE R2

### Install and configure — on the PROXMOX HOST, as root
```
apt install -y rclone
rclone config
```
- **WHY the host and not the VM:** vzdump runs on the hypervisor, because
  that's what has access to the VM's disk image. The backup files land on
  the host, so the uploader lives there too. The guest doesn't know it's
  being backed up.
- **WHY root:** `rclone config` as root writes to
  `/root/.config/rclone/rclone.conf`, which a root cron job can read.
  Configured as another user, the scheduled job wouldn't find the creds.

Wizard answers and reasoning:
- Storage: **Amazon S3** — R2 speaks the S3 API. S3 became the de facto
  standard, so most object stores implement it. That's why an "Amazon"
  backend works against Cloudflare.
- Provider: **Cloudflare** — same API, small dialect differences; naming
  the provider lets rclone adjust.
- `env_auth: false` — `true` means "find credentials in environment
  variables or an attached IAM role," which is for cloud VMs with machine
  identities. We're entering keys directly.
- `region:` blank — R2 auto-distributes across Cloudflare's network. There
  is no region to pin, unlike AWS.
- `endpoint: https://<accountid>.r2.cloudflarestorage.com` —
  **ACCOUNT-level, with NO bucket name appended.** rclone builds paths as
  endpoint + bucket + object. The URL shown on the bucket's settings page
  already includes `/bucketname`; using that doubles it and breaks paths.

```
rclone config show r2
```
- Verify what actually got saved. The interactive edit flow is easy to
  click through without a value landing.

### Testing
```
rclone lsd r2:                      # lists ALL BUCKETS — account-level op
rclone ls r2:homelab-backups/       # lists objects IN one bucket
rclone -vv ls r2:homelab-backups/   # -vv shows the actual API calls
rclone copy <src> r2:bucket/
rclone cat r2:bucket/file
rclone delete r2:bucket/file
rclone delete r2:bucket/ --min-age 8d
```
- **`lsd` will fail with 403 on a bucket-scoped token, and that's correct.**
  Listing all buckets is an account-level operation. The refusal is least
  privilege working. It was simply the wrong first test.
- `ls` on the specific bucket is the right test. **Empty output with no
  error = success** (the bucket is empty).
- `-vv` = very verbose. Shows which operation is being denied rather than
  just "access denied."
- **`cat` is the important one.** It reads the file BACK from Cloudflare.
  Upload alone proves nothing. A backup you cannot retrieve is not a backup.

```
rclone copy --s3-no-check-bucket <src> r2:bucket/
```
- **WHY:** before writing, rclone calls `HeadBucket` to confirm the bucket
  exists (and may attempt `CreateBucket`). Those are BUCKET-level
  operations. A token with Object Read & Write scoped to one bucket cannot
  perform them, so the copy fails with 403 even though the credentials are
  completely correct.
- Permanent fix — add to the `[r2]` section of rclone.conf:
  ```
  no_check_bucket = true
  ```
- **Do NOT widen the token to make the error go away. Find the flag.**

### The debugging method worth remembering
Three separate 403s, three different causes. The breakthrough was noticing
that `ls` SUCCEEDED while `copy` FAILED. That asymmetry proved the
endpoint, account, bucket name and credentials were all correct — leaving
only the permission model in question.

**Narrow a fault by what DOES work, not by staring at what doesn't.**

### Known oddity
```
ERROR : Failed to copy: NotImplemented: 501
ERROR : Attempt 2/3 succeeded
```
Older rclone (Debian ships v1.60.1-DEV) sends an S3 feature R2 doesn't
implement. First attempt fails, retry succeeds. Harmless but noisy.
Usually traced to checksum handling — try `disable_checksum = true` in the
`[r2]` section.

### Cloudflare-side notes
- Use an **Account API token**, not a User API token. Account tokens stay
  valid independent of your user; user tokens go inactive if account
  membership changes. Unattended jobs need account tokens.
- Scope the token to ONE bucket with Object Read & Write. Least privilege.
- Storage class **Standard**, not Infrequent Access. IA is cheaper per GB
  but adds a retrieval fee and a 30-DAY MINIMUM storage duration — you pay
  30 days even if you delete after 2. Rotating nightly snapshots is exactly
  the wrong shape for it. IA suits write-once-keep-for-years archives.
- The secret access key displays ONCE. Save it immediately.
- **If a secret is ever exposed: delete the old token FIRST, then create
  the replacement.** That order leaves no window where a leaked key is
  still live.

---

## 7 — PROXMOX BACKUPS (vzdump)

Created in the UI: Datacenter → Backup → Add.

| Setting | Value | Why |
|---|---|---|
| Storage | local | lands in /var/lib/vz/dump/ |
| Schedule | 02:00 | **dropdown DEFAULTS TO "yearly"** — must be changed |
| Mode | Snapshot | backs up while the VM RUNS — no downtime |
| Compression | ZSTD | fast, good ratio |
| Keep Last | 3 | local history |

The Schedule field accepts presets or systemd calendar expressions.
Test with **Run now** rather than waiting for the schedule.

Backup modes:
- **Snapshot** — VM keeps running. Small risk of catching a database
  mid-write, fine for most workloads.
- **Suspend** — pauses the VM briefly. More consistent, brief downtime.
- **Stop** — full shutdown. Most consistent, most disruptive.

```
ls -lh /var/lib/vz/dump/
```
- `-l` long format, `-h` human-readable sizes.
- Three files per run: `.vma.zst` (the image), `.log` (what happened),
  `.vma.zst.notes` (note template output).
- Result here: **4.8 GB compressed from a 32 GB disk**, about a minute to
  create. At 1–2 MiB/s over WiFi-as-WAN that's 40–80 minutes to upload.

### The offsite push script
```bash
#!/bin/bash
LATEST=$(ls -t /var/lib/vz/dump/*.vma.zst 2>/dev/null | head -1)
[ -z "$LATEST" ] && exit 1
rclone copy "$LATEST" r2:homelab-backups/ \
  --log-file /var/log/rclone-backup.log --log-level INFO
rclone delete r2:homelab-backups/ --min-age 8d \
  --log-file /var/log/rclone-backup.log --log-level INFO
```
- `ls -t` sorts by modification time, newest first; `head -1` takes it.
  **WHY only the newest:** R2's free tier is 10 GB and a snapshot is
  4.8 GB. Only one fits. Local keeps the history; offsite keeps the most
  recent copy for disaster recovery.
- `2>/dev/null` discards the error if the glob matches nothing — otherwise
  a missing-file error lands in the log on every run.
- `[ -z "$LATEST" ] && exit 1` — `-z` tests for an empty string. Bail out
  rather than running rclone against an empty path.
- Quotes around `"$LATEST"` — filenames containing spaces would otherwise
  split into multiple arguments.
- `--log-file` + `--log-level INFO` — scheduled jobs run with nobody
  watching. Without a log, a silent failure is invisible until you need
  the backup.

```
chmod +x /root/backup-to-r2.sh
```
- Makes it executable. Without this, cron can't run it.

### Cron
```
crontab -e
30 3 * * * /root/backup-to-r2.sh
```
- Format: `minute hour day-of-month month day-of-week command`.
  Asterisk means "every".
- `30 3 * * *` = 03:30 daily.
- **WHY 03:30:** vzdump runs at 02:00 and takes about a minute. This leaves
  ample margin so the upload never starts before the file exists.
- **WHY cron rather than running it by hand:** a foreground job dies when
  the SSH session drops. Cron jobs aren't tied to a session.

### Watching a long transfer
```
tail -f /var/log/rclone-backup.log
watch -n 10 tail -5 /var/log/rclone-backup.log
```
- `tail -f` follows a file, printing new lines as they're written. It looks
  static because rclone only writes a progress block every 60 seconds at
  INFO level.
- `watch` re-runs a command on an interval and redraws the screen.
  `-n 10` = every 10 seconds.
- **Ctrl+C on `tail` or `watch` is safe** — it only stops watching.
  **Ctrl+C on rclone itself KILLS the upload.**
- For long foreground transfers, run inside `tmux`/`screen` or with
  `nohup` so they survive a dropped SSH connection or a closed laptop.

---

## 8 — PROXMOX VM MANAGEMENT (host only)

```
qm list
qm config 100 | grep -E "cores|sockets|cpu"
qm reboot 100
```
- `qm` = QEMU Manager. **Only exists on the Proxmox host.** Inside a VM it
  is "command not found" — a useful reminder of which machine you're on.
- `qm config <vmid>` shows what a VM is ACTUALLY allocated, versus what you
  remember allocating.

### Resource allocation principle
**RAM cannot be safely oversubscribed. CPU can.**
- RAM assigned to a VM is physically reserved and unavailable to anything
  else. Over-assigning risks host OOM, where Linux starts killing
  processes to survive.
- vCPUs are time-shared — the hypervisor switches physical cores between
  VMs thousands of times per second. Over-assigning causes contention and
  sluggishness, not failure.
- Rule of thumb: never commit more than ~75–80% of RAM across all VMs.

---

## 9 — SSH HARDENING (planned, not yet applied)

Proxmox genuinely needs root — the web UI authenticates as `root@pam`. So
the goal is NOT eliminating root; it's stopping PASSWORD-based root login
over SSH while keeping key-based access.

Target: `PermitRootLogin prohibit-password`. Do NOT use `no` — that breaks
Proxmox workflows.

**ORDER MATTERS. Set up and TEST key auth before changing anything.**
```
ssh-keygen -t ed25519
type $env:USERPROFILE\.ssh\id_ed25519.pub | ssh root@192.168.8.2 "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
```
- Windows has no `ssh-copy-id`, hence the `type | ssh` construction.

Then open a SECOND terminal and confirm passwordless login works. **Do not
close the working session until verified** — that session plus the
physically-attached console are the safety lines.

Then in `/etc/ssh/sshd_config`:
```
PermitRootLogin prohibit-password
PasswordAuthentication no
```
`systemctl restart sshd`, then test from a NEW terminal while keeping the
old one open.
