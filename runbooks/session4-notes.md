# Homelab Session 4 — Backups, Hardware Market, Hardening Plan

Date: 2026-07-19
Covers: verifying the duplicate-gateway fix, session cleanup, choosing a
backup strategy, Cloudflare R2 setup, the 2026 storage/memory shortage,
hardware sourcing, and a Network+ study plan.

---

## PART 1 — VERIFICATION COMPLETED

### Duplicate gateway — CLOSED
```
sudo ss -ltnp | grep 18789
LISTEN 0 511  127.0.0.1:18789  users:(("openclaw-gatewa",pid=6817,fd=34))
LISTEN 0 511      [::1]:18789  users:(("openclaw-gatewa",pid=6817,fd=33))
```
TWO lines but ONE pid — a single process binding both the IPv4 (127.0.0.1)
and IPv6 ([::1]) loopback stacks. That is normal for a daemon listening on
both. The broken state would have shown two DIFFERENT pids.
Combined with `rich`'s unit disabled and Linger=no, this is fully resolved.

### Lingering — CLOSED
```
sudo -u rich XDG_RUNTIME_DIR=/run/user/1000 systemctl --user is-enabled openclaw-gateway
  → disabled
loginctl show-user rich | grep -i linger  → Linger=yes
sudo loginctl disable-linger rich
loginctl show-user rich | grep -i linger  → Linger=no
```

---

## PART 2 — SESSION MANAGEMENT (stray logins)

```
who
w
who -u
tty          # which PTS am I currently on?
ps -ft pts/1 # what process owns that PTS
```
Reading `w` output:
- The FROM column shows the source IP. A `-` means it is NOT an SSH
  session — it is a local console (Proxmox web shell or the TV console).
- The WHAT column shows what that session is running. The session running
  `w` is you.
- IDLE tells you how long it has been sitting untouched.

Closing sessions:
```
sudo pkill -t pts/1                  # SIGTERM — graceful, preferred
sudo pkill -9 -t pts/1               # SIGKILL — no cleanup, use only if
                                     #   the process ignores SIGTERM
loginctl terminate-user root         # kills ALL sessions for that user
```
NOTE: `loginctl terminate-user root` will disconnect YOU too if you are
logged in as root. Expect to be dropped.

Observation worth acting on later: everything here logs in directly as
root over SSH. Normal for Proxmox, fine on a home LAN, but a hardening
target (see Part 5).

---

## PART 3 — BACKUP STRATEGY

### The decision
Chose **Cloudflare R2** for offsite backups instead of buying a drive.

Reasoning:
- Actual data footprint is tiny: 32 GB VM, ~6.4 GB of real data.
  Compressed vzdump snapshots are a few GB each.
- R2's free tier covers 10 GB of storage with NO egress fees — likely
  covers the whole need indefinitely.
- Sidesteps the 2026 storage shortage entirely (see Part 4).
- Already had a Cloudflare account, which is also needed later for
  Cloudflare Tunnel (replacing the fragile reverse-SSH tunnels).
- The 64 GB flash drive stays as the Proxmox INSTALLER — worth keeping
  bootable recovery media now that live workloads are running.

### R2 bucket
- Name: `homelab-backups`
- Location: Western North America (WNAM)
- Storage class: **Standard**, not Infrequent Access.
  Infrequent Access is cheaper per GB but adds a retrieval fee and a
  30-DAY MINIMUM storage duration — you pay 30 days even if you delete
  after 2. Rotating nightly snapshots is exactly the wrong shape for it.
  IA suits write-once-keep-for-years archives.
- Public Development URL: DISABLED. Custom domain: none.
  Backups must never be publicly reachable.
- Account ID: 4de361f8f306c9f8482cb842f0b9a1a6
- S3 endpoint (account-level, no bucket on the end):
  https://4de361f8f306c9f8482cb842f0b9a1a6.r2.cloudflarestorage.com

### API token
- Used **Account API token**, not User API token. Cloudflare's own note:
  account tokens stay active independent of your user, which is what an
  unattended backup job needs. A user token breaks if account membership
  changes.
- Permission: Object Read & Write
- Scope: the `homelab-backups` bucket ONLY (least privilege — a leaked
  token cannot touch anything else)
- Access Key ID + Secret stored in Apple Passwords.
  The secret is displayed ONCE and never again.
- Never goes in the Git repo.

### rclone setup — runs on the PROXMOX HOST as root
WHY the host and not the VM: `vzdump` runs on the hypervisor because
that is what has access to the VM's disk image. The backup files land on
the host, so the uploader lives there too. The VM doesn't know it's being
backed up.
WHY root: `rclone config` as root writes to
/root/.config/rclone/rclone.conf, which a root cron job or systemd timer
can read. Configured as another user, the scheduled job won't find the
credentials.

```
apt install -y rclone
rclone config
```
Wizard answers:
- n (new remote), name: `r2`
- Storage type: **Amazon S3** (R2 speaks the S3 API — that's why this works)
- Provider: Cloudflare (or "Other")
- env_auth: false
- access_key_id / secret_access_key: from the R2 token
- region: blank or `auto`
- endpoint: the account-level URL above

Verify — a full ROUND TRIP, not just an upload:
```
rclone lsd r2:                              # should list homelab-backups
echo "backup test $(date)" > /tmp/test.txt
rclone copy /tmp/test.txt r2:homelab-backups/
rclone ls r2:homelab-backups/
rclone cat r2:homelab-backups/test.txt      # reads it BACK from Cloudflare
rclone delete r2:homelab-backups/test.txt
```
A backup you cannot retrieve is not a backup. Always test the restore.

### Open decision
The VM backups will contain ~/.openclaw, which holds secrets.env,
credentials/, identity/, and API keys. Decide whether to layer an rclone
`crypt` remote on top so the data is encrypted before it leaves the LAN.

---

## PART 4 — THE 2026 STORAGE & MEMORY SHORTAGE (important context)

Earlier price estimates in these notes were based on pre-shortage figures
and were WRONG. Corrected with current sources:

- HDD prices rose ~46% between Sept 2025 and Jan 2026; individual models
  up 23–66%. Tracking since Sept 2025 shows HDDs +50.8%, SSDs +86.6%.
- Western Digital's production capacity is SOLD OUT for all of calendar
  2026, with datacenter orders locked in through 2027 and 2028.
- Seagate can fill only ~50–66% of near-term demand — a genuine shortage,
  not just a price rise.
- Consumer 1 TB and 2 TB HDDs specifically are hard to obtain.
- DRAM up ~172%. NAND wafer costs reportedly up 246%.
- Dell and Lenovo have raised PC prices 15–20%.
- Cause: AI datacenter buildout. Not a normal chip shortage that clears in
  six months — the contracts are booked years out. Tight through at least
  end of 2026.

Consequences for this lab:
- The 32 GB RAM upgrade will cost far more than the $35–55 estimated
  earlier. Re-price before budgeting. Prices are more likely to rise than
  fall, so "wait until we hit a limit" now has a real cost attached.
- The Dell T5810 / any second machine will also cost more.
- Refurb/used is now the recommended route for price per TB, which
  strengthens rather than weakens the ITAD sourcing strategy.
- Cloud storage is unaffected by the shortage — another point for R2.

---

## PART 5 — HARDWARE SOURCING (ITAD)

The term for the channel: **ITAD — IT Asset Disposition.** Companies that
take decommissioned corporate/datacenter hardware, wipe it, test it, and
resell it. Sage Sustainable Electronics (where the M720q came from) is one.

Sources, roughly best to worst:
- **ITAD sellers on eBay** — Sage, Discount Electronics, PC Server and
  Parts, TechMikeNY, Bargain Hardware. Good ones post actual SMART
  screenshots, stated power-on hours, and photos of the real unit rather
  than stock images.
- **ServerPartDeals** — specializes in recertified enterprise drives.
- **Government / university surplus** — GovDeals, PublicSurplus, state
  university auctions. Cheapest, because sellers just need assets gone.
  Usually as-is, often lots, frequently local pickup only.
- **Local business liquidation** — Craigslist / FB Marketplace. Search
  "server", "rack", "office closing", "IT equipment".
- **r/homelabsales** — people who describe gear accurately.

Search technique: search for what COMPANIES DECOMMISSION, not what
consumers buy. Corporate refresh cycles are 3–5 years, so target roughly
2019–2022 gear. Sort by "newly listed"; good stock moves fast.

### GOTCHA: SAS is not SATA
Enterprise listings are full of SAS drives at tempting prices because
consumer demand is nil. SAS and SATA are different interfaces and the
connector is keyed so a SAS drive will NOT seat in a SATA port.
It works one direction only: a SATA drive WILL plug into a SAS backplane,
never the reverse.
The M720q has a plain 2.5" SATA bay → filter listings to **SATA**, 2.5",
and **7mm height** (15mm drives do not fit).

---

## PART 6 — HARDENING PLAN (not yet done)

### SSH root login
Proxmox genuinely needs root — the web UI authenticates as root@pam and
much of the admin surface expects it. So the goal is NOT eliminating root,
it is stopping PASSWORD-based root login over SSH while keeping key-based
access working.

Target: `PermitRootLogin prohibit-password` (allows key, refuses password).
Do NOT set `no` — that breaks Proxmox workflows.

ORDER MATTERS. Set up and TEST key auth before changing anything:
```
ssh-keygen -t ed25519          # on Windows, if no key exists yet
type $env:USERPROFILE\.ssh\id_ed25519.pub | ssh root@192.168.8.2 "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
```
Then open a SECOND terminal and confirm passwordless login works.
Do not close the working session until verified — that session plus the
TV console are the safety lines.

Then in /etc/ssh/sshd_config:
```
PermitRootLogin prohibit-password
PasswordAuthentication no
```
`systemctl restart sshd`, then test from a NEW terminal while keeping the
old one open.

### Other hardening, by value
- `fail2ban` — watches auth logs, temp-bans repeated failures.
- Reassess exposure before the Cloudflare Tunnel work. Right now nothing
  is reachable from the internet (behind NAT, no port forwarding), so the
  real attack surface is small. Tunnel work is the point where that
  changes — harden BEFORE that, not after.

---

## PART 7 — NETWORK+ (N10-009) STUDY PLAN

Current exam: **N10-009** (released June 2024). 90 questions / 90 minutes.
Passing score 720 of 900. US voucher $390, or $439 with a retake included.
Most candidates spend 60–100 hours preparing.

Make sure any materials are N10-009, NOT N10-008. The newer version adds
cloud networking, network automation fundamentals, Wi-Fi 6/6E/7, SDN,
zero-trust, and IPv6 scenarios.

Highest-leverage study:
- **Subnetting.** ~15–20% of the exam traces back to it. Drill until you
  can produce network address, broadcast, and usable host range for a /27,
  /28, /29 without a calculator. Ten minutes a day.
- **OSI model as a TROUBLESHOOTING ORDER**, not a list to recite. The
  cable fault in session 3 was a layer-1 problem presenting as layers 3–7.
  Diagnose bottom-up.
- **PBQs** (performance-based questions) are interactive scenarios —
  placing a firewall on a diagram, configuring a router. They assume
  hands-on experience.

Materials commonly recommended: CompTIA's free objectives PDF (start
here, use as a checklist), Mike Meyers' All-In-One guide, Jason Dion
practice exams, Professor Messer's free videos, SubnettingPractice.com.

### Honest note on lab-vs-study
The lab gives EXPOSURE, which makes concepts land faster later. It does
not substitute for deliberate study. Much of what has been built so far
was done by following supplied commands — that is a real head start on
vocabulary and context, but it is not the same as being able to diagnose
an unfamiliar network cold.

Approach for closing that gap: when a problem appears, state the symptom
and a hypothesis FIRST, and ask "what should I be looking at" rather than
"what do I run." Slower, but it builds the reasoning the exam and the job
actually test.

Idea: as lab work touches a networking topic, note which N10-009
objective it maps to. Turns the build log into a study aid.

---

## PART 8 — OPEN ITEMS

Next up:
- [ ] Finish rclone config on the Proxmox host + round-trip test
- [ ] Decide on rclone `crypt` (encrypt backups before they leave the LAN)
- [ ] Schedule vzdump for VM 100, then push to R2
- [ ] TEST A RESTORE. An untested backup is not a backup.

Hardening:
- [ ] SSH key auth for root, then PermitRootLogin prohibit-password
- [ ] fail2ban

Deferred:
- [ ] Disk 32 → 64 GB: snapshot FIRST, then swapoff, remove /dev/sda5,
      growpart + resize2fs, replace with a 4 GB swapfile
- [ ] Re-price the 32 GB RAM kit given the shortage
- [ ] Hermes migration — inventory the AWS host first, then size hardware
      to measured need (OpenClaw asked for 24 GB and uses ~1.3 GB)

Housekeeping:
- [ ] Clean up /root/openclaw-telegram-repair-20260719T091122Z once stable
- [ ] Identify the "Cloudflare Agent Token" (May 2026, all buckets, admin
      read-only) — know what created it and whether it is still needed
