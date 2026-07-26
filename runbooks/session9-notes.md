# Session 9 — Why Deleting Files Didn't Shrink the Backups (TRIM)

Date: 2026-07-26

Last session we deleted ~4 GB of junk from the VM. The backups stayed at
7.5 GB anyway. This session explains why deleting files doesn't shrink a
VM backup until you do something called TRIM — and fixes the R2 upload
script so the cloud bucket stops accumulating. Written to explain the
concepts, because "I deleted files but the backup didn't shrink" makes no
sense until you understand how disks actually free space.

---

## PART 1 — THE PUZZLE

After session 8's cleanup, the situation was:
- The VM's actual files: **11 GB** (checked with `du`)
- The nightly backup: still **7.5 GB compressed**

That doesn't add up. 11 GB of real files, well-compressed, should produce
a backup far under 7.5 GB. And we'd just deleted ~4 GB — the backup should
have shrunk and didn't.

There was also a false alarm along the way: the dump directory showed
"23 GB," which looked like the VM had ballooned again. It hadn't — that
was **three** nightly backups at ~7.5 GB each sitting in
`/var/lib/vz/dump/` (the keep-last-3 retention). Retention working, not a
leak. Worth internalizing: `23G` in the dump folder ÷ 3 kept copies ≈ the
size of one dump. Always check whether a big number is one thing or
several.

---

## PART 2 — THE KEY CONCEPT: DELETING A FILE DOESN'T ERASE IT

This is the crux, and it's genuinely counterintuitive.

When you delete a file, the operating system does NOT go scrub the actual
data off the disk. That would be slow and pointless. Instead it just marks
those blocks as "available" in the filesystem's index — like removing a
book's entry from the library catalog while leaving the book on the shelf.
The catalog says the space is free; the old data physically remains until
something else writes over it.

For normal use this is invisible and fine. The space shows as free, new
files land in it, life goes on.

### Why it breaks VM backups

A Proxmox backup (vzdump) does NOT copy your files. It copies the **entire
virtual disk image** — all 32 GB of the virtual drive, block by block,
compressing as it goes. It reads the raw disk, not the filesystem's view
of it.

So vzdump reads and compresses **the leftover data from every file you
ever deleted**, because those bytes are still physically on the disk. Your
deleted Chrome copy, the stale OpenClaw directory, months of old caches —
all still there at the block level, all still getting read and compressed
into every single nightly backup.

That's why deleting files did nothing. The filesystem forgot about them.
The disk image didn't.

### Compression makes this vivid

Compression shrinks patterns and repetition. Zeroed-out space (genuinely
empty blocks, all 0x00) compresses to almost nothing — a huge run of
identical bytes. But leftover deleted data is real, varied data. It
compresses like real data, i.e. poorly. So a disk "full" of deleted-file
ghosts produces a big backup, while a disk with the same free space
properly zeroed produces a tiny one.

---

## PART 3 — THE FIX: TRIM

TRIM is the operation that closes the gap. It tells the underlying storage
"these blocks are genuinely free — you can forget their contents." Once
told, the storage layer can zero them, and zeros compress to nearly
nothing.

```
sudo fstrim -av
```
- `fstrim` walks mounted filesystems and issues TRIM for every free block.
- `-a` = all mounted filesystems that support it.
- `-v` = verbose — report how much was trimmed per filesystem.

What it reported:
```
/home/ubuntu/.openclaw/backups/local: 13.2 GiB trimmed on /dev/sdb1
/: 19.3 GiB trimmed on /dev/sda1
```

**19.3 GB of ghost data on the root disk.** That's every file deleted
since the VM was created, finally being released at the storage layer. The
next backup reads those as empty blocks and compresses them away.

### Why this works here specifically

TRIM only passes through to the storage if the virtual disk was created
with **discard** enabled. Ours was — the config showed `discard=on` on
both disks (we checked it back when adding the backup disk). `discard` is
the setting that lets a guest's TRIM commands reach the hypervisor's
storage and actually free the space. Without it, `fstrim` would run but
accomplish nothing.

### Making it automatic
```
sudo systemctl enable --now fstrim.timer
```
- Runs `fstrim` on a schedule (weekly by default) so ghost data gets
  released regularly without you remembering.
- `enable` = turn it on at boot, `--now` = also start it immediately.
- Standard practice for any VM or SSD-backed system.

We still ran it by hand today because we'd just done a lot of deleting and
didn't want to wait a week for the timer.

### The takeaway
**On a VM, deleting files does not shrink backups. You must TRIM.** The
filesystem frees blocks logically; TRIM frees them physically. vzdump sees
the physical disk, so only TRIM affects backup size. This one command did
more for the dump size than all of last session's manual deletion.

---

## PART 4 — THE OTHER FIX: R2 KEEPING TOO MANY COPIES

Separate problem, same session. The Cloudflare bucket had accumulated
FIVE dumps (~31 GB total), far over the 10 GB free tier.

### Why it accumulated
The old upload script pruned by AGE:
```
rclone delete r2:homelab-backups/ --min-age 8d
```
That only removes objects older than 8 days. But a new ~8 GB object lands
every night, and none reach 8 days old before several have piled up. So
the bucket grew nightly and never self-corrected. Age-based pruning is the
wrong tool when you only want to keep ONE of something.

### The new script
```bash
#!/bin/bash
LATEST=$(ls -t /var/lib/vz/dump/*.vma.zst 2>/dev/null | head -1)
[ -z "$LATEST" ] && exit 1
BASENAME=$(basename "$LATEST")

# upload the newest local dump
rclone copy "$LATEST" r2:homelab-backups/ --log-file /var/log/rclone-backup.log --log-level INFO

# only prune if the upload actually landed in the bucket
if rclone lsf r2:homelab-backups/ | grep -q "^$BASENAME$"; then
  rclone delete r2:homelab-backups/ --exclude "$BASENAME" --log-file /var/log/rclone-backup.log --log-level INFO
fi
```

### Line by line
- `LATEST=$(ls -t ... | head -1)` — newest dump file. `ls -t` sorts
  newest-first; `head -1` takes it.
- `[ -z "$LATEST" ] && exit 1` — `-z` is true if the string is empty. Bail
  if no dump was found rather than run rclone against nothing.
- `BASENAME=$(basename "$LATEST")` — strips the folder path, leaving just
  the filename. R2 stores objects by name, so this is what we match on.
- `rclone copy` — upload the newest dump.
- `rclone lsf` — list the bucket's filenames (`lsf` = list, filenames
  only, no sizes/dates).
- `grep -q "^$BASENAME$"` — quietly (`-q`, no output) check whether our
  filename is in that list. `^...$` anchors mean the WHOLE line must match
  exactly, so it can't accidentally match a partial or similar name.
- The `if` runs the delete ONLY when the upload is confirmed present.
- `rclone delete ... --exclude "$BASENAME"` — delete everything in the
  bucket EXCEPT the file we just uploaded.

### Why the safety check matters
Order is upload-then-delete, but with a guard. Consider the NIC-hang
nights: the upload FAILED (DNS timeouts, network wedged). Without the
guard, the script would then delete "everything except today's file" —
but today's file never uploaded, so it would delete the previous good
copy and leave the bucket EMPTY. The `if rclone lsf | grep` check prevents
that: no confirmed upload, no deletion. A failed night simply leaves the
prior copy intact.

### Result
After every SUCCESSFUL run the bucket holds exactly ONE object (the
newest). A failed run changes nothing. It can neither accumulate nor
accidentally empty itself. Ran it by hand to clean up the five stragglers;
`rclone ls r2:homelab-backups/` now shows one file, comfortably inside the
free tier.

---

## PART 5 — df vs du, ONE MORE TIME (because it keeps mattering)

- **df** asks the filesystem "how full are you?" — fast, and it counts the
  ghost blocks as free (they ARE free to the filesystem).
- **du** adds up actual files in a tree — it never sees ghost data,
  because deleted files aren't files anymore.
- **vzdump** reads the raw disk image — it DOES see ghost data, because
  the bytes are physically there.

That three-way difference is the whole story of this session:
`du` said 11 GB, `df` agreed there was free space, but the backup was
7.5 GB because it read the physical disk including the ghosts. TRIM
reconciles them by actually releasing the free blocks.

---

## PART 6 — LESSONS

### 61. Deleting files does not erase them; it de-indexes them
The data stays on disk until overwritten. Normal use never notices. Raw
disk-image backups notice, because they read the physical disk, not the
filesystem's catalog.

### 62. On a VM, TRIM is what actually shrinks backups
`sudo fstrim -av` releases freed-but-not-reclaimed blocks so they compress
to nothing. Deleting files without TRIM does nothing for backup size.
Requires `discard=on` on the virtual disk (we had it). Enable
`fstrim.timer` to automate it weekly.

### 63. A big number might be several things, not one problem
"23 GB" in the dump folder looked like the VM ballooned. It was three kept
backups at ~7.5 GB each — retention, not a leak. Divide before you panic.

### 64. Prune-by-age is wrong when you want to keep exactly N
`--min-age 8d` let nightly ~8 GB objects pile up because none aged out
fast enough. To keep exactly one, delete everything EXCEPT the newest,
guarded by a check that the newest actually uploaded.

### 65. Guard destructive automation against the failure case
The prune deletes "everything except today's file." If today's upload
failed, that logic would wipe the last good copy. The `if rclone lsf |
grep` guard makes deletion conditional on a confirmed upload. Always ask
of an automated delete: "what does this do on the night the previous step
failed?"

---

## PART 7 — OPEN ITEMS

Confirm:
- [ ] Tonight's 2am dump should drop toward ~4–5 GB (or lower) now that
      TRIM released 19.3 GB of ghost blocks. THIS is the payoff to verify.
- [ ] After it shrinks, the three local kept copies will shrink over the
      next three nights as each is replaced.

Carried, still not done:
- [ ] SSH hardening — key auth, then PermitRootLogin prohibit-password,
      then fail2ban
- [ ] Encrypt backups before upload? (dump has secrets.env, credentials/,
      identity/, seven providers' API keys)
- [ ] Clean up /root/openclaw-telegram-repair-20260719T091122Z
- [ ] Identify the May 2026 "Cloudflare Agent Token"
- [ ] disable_checksum = true in rclone.conf (stops 501 retry noise)
- [ ] Remove leftover Debian ISO from VM 100's virtual CD drive
- [ ] Watch VM memory (was 78% after 4 days uptime)

NIC hang (session 7):
- [ ] Confirm no new "Hardware Unit Hang" in dmesg over coming days
- [ ] The two failed R2 uploads (Jul 23/24) were caused by the hang;
      uploads resumed on their own once the offload fix took effect
      (Jul 25/26 present in the bucket)

Hermes migration — SIZING NOW MEASURED:
- Hermes agent itself uses only ~647 MB resident. The whole AWS box
  (Hermes + both web apps + ASL + trading bot + MCP servers) uses just
  2.5 GB total. Earlier intuition that "Hermes needs way more RAM than
  OpenClaw" did not survive measurement — they're comparable.
- Plan: Rocky Linux 10 VM (RHEL 10.2 on AWS maps cleanly to it),
  8 GB RAM, 4 vCPU (6 available if bursts demand). Fits alongside
  OpenClaw's VM on 16 GB without the RAM upgrade.
- DECISIONS MADE:
  - Trading bot STAYS in the cloud on its own cheap VPS ($5/mo class) —
    it runs unattended with real money on Polymarket, and this home
    network hangs daily. Wrong place for it.
  - Everything else comes home: Hermes, the ASL app, and the
    *.richgibbs.dev websites. No customers, so downtime is acceptable.
  - Web-facing pieces need Cloudflare Tunnel (outbound from home, so no
    port forwarding / no exposed home IP) — this replaces the fragile
    reverse-SSH tunnels entirely.
- Migration order: (1) trading bot → VPS, (2) build Rocky VM,
  (3) migrate Hermes, (4) migrate ASL app, (5) Cloudflare Tunnel for
  ASL + sites, (6) cut over DNS, verify, decommission AWS.
- STILL TO ANSWER before starting: do you have a VPS already or is
  standing one up part of this, and how well do you understand the
  trading bot's own code (it has real money on it — move slowest there).
- Cleanup opportunity on the AWS box first: real payload is 3–4 GB under
  ~22 GB of cache/duplicates (uv cache 2.2 GB, two ASL copies, two torch
  libs, redundant OpenClaw sweep). Also: OpenClaw's secrets were copied
  onto that internet-facing box during the pre-migration sweep — worth
  removing regardless of migration timing.
