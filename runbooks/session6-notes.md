# Session 6 — Adding a Dedicated Backup Disk (explained from scratch)

Date: 2026-07-23

This one is written to EXPLAIN, not just record. If you don't understand
what we did, this document should fix that. Read Part 1 first — the
commands in Part 4 won't mean much without it.

---

## PART 1 — WHY WE DID THIS AT ALL

### The problem, in plain terms

Two different things were making backups:

1. **Proxmox (vzdump)** takes a snapshot of the ENTIRE virtual machine
   every night at 2am — the whole 32 GB disk, compressed down to a single
   file. This is your disaster-recovery copy.

2. **OpenClaw** makes its own backups INSIDE the VM — `.tar.gz` archives
   of its own data, stored at `~/.openclaw/backups/local/`. These are for
   quick "undo" situations.

The trouble: OpenClaw's archives live inside the VM, and Proxmox backs up
the whole VM. So every OpenClaw archive was getting swept into the Proxmox
dump too.

### Why that made the dumps grow so fast

The Proxmox dumps went 4.77 GB → 4.79 GB → **6.17 GB** in three days.

The reason is a compression fact worth understanding: **you can't compress
the same data twice.**

Compression works by finding repetition and patterns and encoding them
more efficiently. Once that's been done, the data looks essentially random
— there are no patterns left to find. So when Proxmox's compressor (zstd)
hits an 820 MB `.tar.gz` file that OpenClaw already compressed, it can't
shrink it at all. That 820 MB lands in the dump as a full 820 MB.

Two OpenClaw archives = about 1.6 GB added to every nightly dump, forever,
and growing with each new archive (OpenClaw was keeping 7).

### Why that actually mattered

- **Cloudflare R2's free tier is 10 GB.** At 6.17 GB you're close; a
  couple more archives and every nightly upload starts costing money.
- **Upload time.** 4.8 GB took 34 minutes over your connection. 10 GB
  would take over an hour, nightly, on a link that has already failed
  twice.
- **It's redundancy you don't actually get.** OpenClaw's archives live
  INSIDE the thing Proxmox is backing up. If the VM's disk dies, both die
  together. You were paying full price in space and bandwidth for a second
  copy that shares the same single point of failure.

### Why the obvious fix didn't work

The intuitive answer is "tell Proxmox to skip that folder." I suggested
exactly that, and I was wrong. OpenClaw caught it.

**vzdump backs up DISK IMAGES, not files.** When it backs up a VM it
copies the whole virtual hard drive as one blob — it doesn't look inside
at individual files, so it can't skip any. (File-level exclusions do exist
in Proxmox, but only for LXC containers, which work differently.)

### The fix that does work

Proxmox lets you mark an individual **virtual disk** as "don't back this
up." So:

1. Give the VM a SECOND virtual disk
2. Mark that disk `backup=0`
3. Put OpenClaw's archives on it

Now the archives are on a disk vzdump ignores entirely. OpenClaw keeps its
fast local backups; Proxmox stops duplicating them.

---

## PART 2 — CONCEPTS YOU NEED (the stuff nobody explains)

### What a "virtual disk" actually is

Your M720q has one real 256 GB NVMe drive. A virtual disk is just a chunk
of that real drive that Proxmox hands to a VM and says "pretend this is a
whole hard drive."

The VM has no idea. Inside Debian, it looks exactly like a physical disk
you plugged in. That's the entire illusion of virtualization.

So "adding a 16 GB disk" means: Proxmox carved 16 GB out of the NVMe and
presented it to the VM as a new drive.

### Disk names: sda, sdb, sr0

Linux names storage devices in the order it finds them:
- `sda` — first disk ("**s**torage **d**evice **a**")
- `sdb` — second disk
- `sr0` — first CD/DVD drive

Partitions get numbers appended: `sda1`, `sda2`, `sda5`.

From your `lsblk` output before we started:
```
sda      32G          ← the original disk
├─sda1  30.3G  /      ← partition 1, holds the whole system, mounted at /
├─sda2     1K         ← "extended partition" — just a container, ignore it
└─sda5   1.7G  [SWAP] ← partition 5, swap space
sdb       16G         ← the NEW disk, completely blank
sr0      755M  rom    ← the virtual CD drive (leftover Debian ISO)
```

### The three steps to make a disk usable

A brand-new disk is a blank slab. Three things have to happen, in order,
and they're genuinely different operations:

**1. PARTITION** — divide the disk into labelled regions.
   Even if you only want one region covering the whole disk, you still
   have to declare that. It's like drawing property lines on empty land.
   The script did this: created a GPT partition table and one partition
   `/dev/sdb1` covering all 16 GB.

   *GPT* is the modern partition table format. The older one (MBR) had
   limits — 2 TB max, only 4 primary partitions. GPT has neither.

**2. FORMAT** — install a filesystem in that region.
   A partition is just "these sectors are reserved." A filesystem is the
   organizational system that tracks where files start and end, what
   they're named, who owns them, and when they changed. Without it, the
   disk can hold bytes but has no idea what a "file" is.

   We used **ext4**, the standard Linux filesystem. Formatting is what
   destroys data — this is the step to be careful about.

   *That's why the script refuses to format a disk that already has a
   filesystem unless you explicitly set `FORCE_REFORMAT=1`. It's a
   guardrail against pointing it at the wrong disk.*

**3. MOUNT** — attach the filesystem to a folder in your directory tree.
   This is the concept that's most alien coming from Windows.

### Mounting, and why it's not like Windows

Windows gives each drive its own letter: C:, D:, E:. Separate trees.

Linux has ONE tree, starting at `/`. Additional disks get **attached to a
folder** somewhere in that tree. That folder is called a *mount point*.

So when we mounted `/dev/sdb1` at
`/home/ubuntu/.openclaw/backups/local`, we said: "from now on, anything
written to that path goes onto the new disk instead of the old one."

The path looks identical. Programs don't need to change. But the bytes
land somewhere completely different. OpenClaw keeps writing to the same
folder it always used, unaware that folder is now a different piece of
hardware.

### THE BUG WE HIT: mounting over a non-empty folder

This is the thing that went wrong, and it's worth really understanding.

When you mount a disk onto a folder that **already has files in it**, the
existing files don't get deleted — they get **hidden**. The new
filesystem is laid over the top like a sheet over furniture. The furniture
is still there; you just can't see it.

That's exactly what happened. `~/.openclaw/backups/local` already held
1.7 GB of archives on `sda1`. The script mounted the new empty disk on top
of that folder. Result:

- `ls` showed an almost-empty folder (you were seeing the new disk)
- `df` said 2.1 MB used (the new disk really was nearly empty)
- The 1.7 GB of archives were still on `sda1`, invisible

**The tell:** the folder appeared to have lost 1.7 GB of files, but the
old disk's free space hadn't gone up. Data doesn't vanish silently. If
files "disappear" but the space isn't freed, something is hiding them.

**How to see underneath a mount:** unmount it, or bind-mount the root
filesystem somewhere else and look there. The files were fine the whole
time.

### fstab — making a mount permanent

Mounting by hand lasts until reboot. To make it automatic, you add a line
to `/etc/fstab` ("**f**ile **s**ystems **tab**le"), which the system reads
at every boot.

Three details in that line matter:

**UUID instead of /dev/sdb1.** Device names are assigned in discovery
order and can change — add a disk, and yesterday's `sdb` might come up as
`sdc`. A **UUID** (Universally Unique Identifier) is a long random string
baked into the filesystem when it's formatted. Yours is
`beb93537-d011-47fd-b2fb-f71c2de09a21`. It never changes, no matter what
order things are detected in. Always mount by UUID.

**`nofail`.** By default, if a disk listed in fstab is missing at boot,
Linux considers the boot failed and drops you into an emergency shell.
For a non-essential disk that's terrible — a dead backup disk shouldn't
stop the VM from starting. `nofail` means "if this isn't here, carry on."

**`x-systemd.device-timeout=10`.** Wait at most 10 seconds for the disk
before giving up, instead of hanging.

**A bad fstab entry only bites at boot.** Everything looks fine until you
reboot, and then the machine won't come up. That's why we took a snapshot
first, and why rebooting to test was mandatory rather than optional.

### Checksums — how you know a copy is a real copy

When the recovery script moved the archives, it verified them with
`sha256sum -c`.

A **checksum** is a fixed-length fingerprint calculated from a file's
contents. Change one byte anywhere and the fingerprint changes completely.
So you record the fingerprint when you create a file, then recalculate it
later — if they match, the file is bit-for-bit intact.

Those `.sha256` files sitting next to each archive are the recorded
fingerprints. `sha256sum -c` recalculates and compares.

This matters because **a file can copy "successfully" and still be
corrupt.** Comparing sizes catches truncation; only a checksum catches
silent corruption. It's the same reasoning behind `rclone check` on the
R2 upload.

### Ownership — why chown kept coming up

Every file has an owner and a group. When a script runs under `sudo`, it
runs as root, so anything it creates is owned by root.

OpenClaw runs as the `ubuntu` user. If its backup folder ends up
root-owned, OpenClaw can't write to it — and the failure appears later, at
the next backup, not at setup time.

That's why the recovery script explicitly ran `chown ubuntu:ubuntu`. It
left `lost+found` root-owned, which is correct — that's a filesystem
recovery directory that belongs to the system, not to you.

---

## PART 3 — WHAT ACTUALLY HAPPENED, IN ORDER

1. **Took a snapshot** (`pre-backup-disk`). Rollback point. Cheap
   insurance before touching partitions and fstab.

2. **Added a 16 GB SCSI disk in Proxmox** with the **Backup checkbox
   UNCHECKED**. That checkbox is the entire point of the exercise.

3. **Verified** `qm config 100 | grep scsi` showed `backup=0` on the new
   disk. If that flag were missing, we'd have added 16 GB that still got
   backed up — worse than doing nothing.

4. **Confirmed the guest saw it** — `lsblk` showed `sdb`, 16 GB, blank,
   exactly one candidate.

5. **Ran the setup script**, which partitioned (GPT), formatted (ext4),
   wrote the fstab entry, and mounted it.

6. **Discovered the archives hadn't moved** — `df` showed 2.1 MB used on
   a disk that should have held 1.7 GB. The files were hidden under the
   mount.

7. **Ran a recovery script** that unmounted, found the originals, copied
   them onto the new disk, verified with sha256, fixed ownership, and
   kept the originals in a staging folder rather than deleting them.

8. **Rebooted** — the real test of fstab. `findmnt` confirmed the mount
   came back automatically.

9. **Deleted the staging copy** (it was still on `sda1`, still counting
   toward the dump size — the exact problem we were solving).

Result:
```
/dev/sdb1  16G  1.7G  14G  11%  /home/ubuntu/.openclaw/backups/local
```

---

## PART 4 — THE COMMANDS

```
lsblk
```
List **bl**ock devices — every disk and partition, with sizes and where
each is mounted. The first thing to run when dealing with storage.

```
qm config 100 | grep -E 'scsi[0-9]+'
```
Show VM 100's config, filtered to its disks. Run on the PROXMOX HOST —
`qm` doesn't exist inside a VM. You're looking for `backup=0`.
(A stray `+` at the start of a grep pattern causes a warning — patterns
can't begin with a repetition operator.)

```
sudo bash -c 'echo "- - -" > /sys/class/scsi_host/host0/scan'
```
Force the kernel to rescan for new SCSI devices, if a hotplugged disk
doesn't appear. The three dashes mean "any controller, any channel, any
device" — scan everything. Rebooting also works.

```
findmnt /path
```
Show what's mounted at a path, and from which device. Clearer than
digging through `mount` output.

```
df -h /path
```
**D**isk **f**ree. `-h` = human-readable. Shows which filesystem backs a
path and how full it is. **This is what revealed the bug** — 2.1 MB used
where 1.7 GB should have been.

```
du -sh /path
```
**D**isk **u**sage. `-s` summary (one total, not per-file), `-h` human.
`df` asks the filesystem how full it is; `du` adds up actual files. When
they disagree, something interesting is happening.

```
grep openclaw /etc/fstab
```
Check the persistent mount entry. Verify it uses `UUID=` and includes
`nofail`.

```
sudo systemctl daemon-reload
```
Tell systemd to re-read fstab. systemd caches it, so edits aren't seen
until you reload — the original script's output warned about exactly this.

```
sudo mount /path
```
Mount using the fstab entry for that path. Tests the entry without a
reboot.

```
sudo umount /path
```
Unmount. Note the spelling — no "n". Unmounting reveals whatever was
hidden underneath.

```
sudo rsync -av <src>/ <dest>/
```
Copy preserving permissions, ownership, timestamps (`-a`), verbosely
(`-v`). **Trailing slashes matter:** `src/` means "the contents of src".
Preferred over `cp` because it preserves metadata and can resume.

```
sha256sum -c file.sha256
```
Recalculate a file's fingerprint and compare it to the recorded one.
Proves a copy is bit-for-bit identical, not merely the right size.

```
sudo chown -R ubuntu:ubuntu /path
```
Change **own**er recursively. Format is `user:group`. Needed because
files created under sudo belong to root, and OpenClaw runs as ubuntu.

```
sudo reboot
```
The only real test of an fstab change.

---

## PART 5 — LESSONS

### 39. You cannot compress compressed data
An 820 MB `.tar.gz` inside a VM adds a full 820 MB to the VM's compressed
dump. Compression finds patterns; once removed, there are none left. This
is why nested backups are expensive in a way that isn't obvious.

### 40. vzdump backs up DISK IMAGES, not files
There is no file-level exclusion for VMs — that only exists for LXC
containers. To exclude data from a VM backup, it has to live on a separate
virtual disk marked `backup=0`. I got this wrong and OpenClaw corrected
it.

### 41. Mounting over a non-empty folder HIDES its contents
The files aren't deleted, they're covered. `ls` shows the new filesystem;
the old files sit underneath, invisible but still consuming space on the
original disk.
**The tell:** files appear to vanish but the space isn't freed. Data
doesn't disappear silently. To see underneath: unmount, or bind-mount the
root filesystem elsewhere.

### 42. Always mount by UUID, never by device name
`/dev/sdb` is assigned in discovery order and can change. A UUID is baked
into the filesystem at format time and never changes.

### 43. `nofail` prevents a missing disk from blocking boot
Without it, an absent disk drops the system into an emergency shell. For
a non-essential disk that's a terrible trade.

### 44. A bad fstab entry only fails at BOOT
Everything looks fine until the reboot, and then the machine won't come
up. This is why the snapshot came first and the reboot test was mandatory.

### 45. Verify a copy with checksums, not just size
A file can copy "successfully" and still be corrupt. Size catches
truncation; only a checksum catches silent corruption. Same reasoning as
`rclone check` on the R2 upload.

### 46. Scripts running under sudo create root-owned files
And the failure shows up later, when the application (running as a
different user) can't write. Fix ownership explicitly after any
sudo-driven setup.

### 47. Verify the safety flag BEFORE doing the work
`backup=0` had to be confirmed before running the setup script. Without
it, the whole exercise would have added 16 GB of still-backed-up disk —
strictly worse than doing nothing. Check that the thing which makes the
work worthwhile is actually in place first.

---

## PART 6 — OPEN ITEMS

Confirm tomorrow:
- [ ] The 2am dump should drop from 6.17 GB back toward ~4.8 GB
- [ ] Delete the `pre-backup-disk` snapshot once confirmed (snapshots
      consume space and slightly slow the VM while they exist)

Watch:
- [ ] VM memory was at 78% (6.27 of 8 GiB) after 4 days uptime, up from
      1.3 GiB after the resize. Some is disk cache and normal. Worth
      checking whether OpenClaw's resident set is climbing:
      `ps -eo pid,user,%mem,rss,cmd --sort=-rss | head -10`
- [ ] The intermittent network fault. Two outages, both fixed by
      reseating a NEW cable. Next occurrence, note whether the VM is also
      unreachable and whether the router admin still loads over WiFi —
      that narrows it to cable vs. router port vs. NIC.

Still not done:
- [ ] SSH hardening: key auth for root, then
      `PermitRootLogin prohibit-password`, then fail2ban
- [ ] Decide on encrypting backups before upload (the dump contains
      secrets.env, credentials/, identity/, and seven providers' API keys)
- [ ] Clean up /root/openclaw-telegram-repair-20260719T091122Z
- [ ] Identify the May 2026 "Cloudflare Agent Token" (all buckets,
      admin read-only)
- [ ] `disable_checksum = true` in rclone.conf to stop the 501 retry noise
- [ ] Remove the leftover Debian ISO from the VM's virtual CD drive

Deferred:
- [ ] Disk 32 → 64 GB (snapshot, swapoff, remove /dev/sda5, growpart +
      resize2fs, replace with a swapfile)
- [ ] Re-price the 32 GB RAM kit given the 2026 shortage
- [ ] Hermes migration — inventory first, then size hardware to measured
      need
