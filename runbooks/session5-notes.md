# Homelab Session 5 — Backups Working (rclone + R2 + vzdump)

Date: 2026-07-20
Covers: configuring rclone against Cloudflare R2, debugging a series of
403s, creating the Proxmox backup job, and automating the offsite push.

---

## PART 1 — RCLONE CONFIG (on the PROXMOX HOST, as root)

```
apt install -y rclone
rclone config
```

Wizard answers used:
- `n` new remote, name: `r2`
- Storage: **5** — Amazon S3 (R2 speaks the S3 API; that's why an
  S3 backend works against Cloudflare)
- Provider: **5** — Cloudflare R2 Storage
- env_auth: `false` (entering keys directly rather than pulling from
  environment variables or an IAM role)
- access_key_id / secret_access_key: from the R2 Account API token
- region: left blank (R2 auto-distributes)
- endpoint: `https://<accountid>.r2.cloudflarestorage.com`
  — ACCOUNT-level, with NO bucket name on the end. The URL shown on the
  bucket's settings page DOES include /bucketname; using that breaks
  path resolution.

Config lives at `/root/.config/rclone/rclone.conf`.

Useful commands:
```
rclone config show r2          # print the saved config for one remote
rclone -vv ls r2:bucket/       # -vv = very verbose, shows the actual calls
```

---

## PART 2 — THE 403 DEBUGGING SAGA (three separate causes)

Worth writing out in full because each 403 had a different cause and the
progression is a good example of narrowing a fault.

### 403 #1 — `rclone lsd r2:` — NOT A BUG
`lsd` lists ALL BUCKETS in the account. That is an account-level
operation. The token is deliberately scoped to one bucket, so R2
correctly refuses.
This was a bad first test command. The right one is:
```
rclone ls r2:homelab-backups/
```
which is a pure object operation inside the bucket.
**Empty output with no error = success** (the bucket was empty).

### 403 #2 — reads worked, writes failed
`ls` succeeded but `copy` returned 403. That pattern is diagnostic on its
own: endpoint, account, bucket name and credentials must ALL be correct,
or the read would have failed too. Only the permission level was in
question.
First theory was a read-only token — but the dashboard confirmed
Object Read & Write on the right bucket.

### 403 #3 — the actual cause: rclone's bucket existence check
Before writing, rclone performs a `HeadBucket` (and may attempt
`CreateBucket`). Those are BUCKET-level operations, not object-level.
A token scoped to Object Read & Write on one bucket cannot do them, and
rclone surfaces the denial as a 403 on the copy.

Fix:
```
rclone copy --s3-no-check-bucket /tmp/test.txt r2:homelab-backups/
```
Made permanent by adding to the `[r2]` section of rclone.conf:
```
no_check_bucket = true
```

### Bonus oddity — 501 Not Implemented on attempt 1, success on attempt 2
```
ERROR : test.txt: Failed to copy: NotImplemented: 501
ERROR : Attempt 2/3 succeeded
```
Older rclone (Debian ships v1.60.1-DEV) sends an S3 feature R2 doesn't
implement; the first attempt fails, the retry succeeds. Harmless but
noisy in logs. Usually traced to checksum handling — try adding
`disable_checksum = true` to the `[r2]` config section.

### Verification — always test the ROUND TRIP
```
echo "backup test $(date)" > /tmp/test.txt
rclone copy /tmp/test.txt r2:homelab-backups/
rclone ls r2:homelab-backups/
rclone cat r2:homelab-backups/test.txt   # reads it BACK from Cloudflare
rclone delete r2:homelab-backups/test.txt
```
Upload alone proves nothing. A backup you cannot retrieve is not a backup.

---

## PART 3 — PROXMOX BACKUP JOB (vzdump)

Created via Datacenter → Backup → Add.

| Setting | Value | Why |
|---|---|---|
| Node | pve-node1 | only node |
| Storage | local | lands in /var/lib/vz/dump/ |
| Schedule | 02:00 | **the dropdown DEFAULTS TO "yearly"** — must be changed |
| Selection mode | Include selected VMs → 100 | |
| Compression | ZSTD | fast and good ratio |
| Mode | **Snapshot** | backs up while the VM RUNS — no downtime |
| Keep Last | 3 | local history |

Schedule field accepts presets or systemd calendar expressions (`02:00`).

Test immediately with **Run now** rather than waiting for the schedule.

Result:
```
ls -lh /var/lib/vz/dump/
4.8G  vzdump-qemu-100-2026_07_20-13_08_33.vma.zst
3.8K  vzdump-qemu-100-2026_07_20-13_08_33.log
 13   vzdump-qemu-100-2026_07_20-13_08_33.vma.zst.notes
```
4.8 GB compressed from a 32 GB disk, completed in about a minute.

Backup modes worth knowing:
- **Snapshot** — VM keeps running. Slight risk of an inconsistent moment
  for databases mid-write, but fine for most workloads.
- **Suspend** — pauses the VM briefly. More consistent, brief downtime.
- **Stop** — full shutdown. Most consistent, most disruptive.

---

## PART 4 — OFFSITE PUSH SCRIPT

Constraint: R2's free tier is 10 GB and a snapshot is 4.8 GB, so only
ONE fits comfortably offsite. Two would be 9.6 GB with no margin — and
during an upload you'd briefly hold both, exceeding the tier.

Decision: keep **3 locally** (fast restores, recent history) and
**1 offsite** (disaster recovery — fire, theft, drive failure).
Beyond the free tier R2 is ~$0.015/GB/month, so keeping 3 offsite would
cost roughly a nickel a month if deeper history is wanted later.

```
nano /root/backup-to-r2.sh
```
```bash
#!/bin/bash
LATEST=$(ls -t /var/lib/vz/dump/*.vma.zst 2>/dev/null | head -1)
[ -z "$LATEST" ] && exit 1
rclone copy "$LATEST" r2:homelab-backups/ --log-file /var/log/rclone-backup.log --log-level INFO
# prune anything in R2 older than 8 days
rclone delete r2:homelab-backups/ --min-age 8d --log-file /var/log/rclone-backup.log --log-level INFO
```
```
chmod +x /root/backup-to-r2.sh
```

Line by line:
- `ls -t` sorts by modification time, newest first; `head -1` takes the
  newest. So only the latest snapshot is uploaded.
- `2>/dev/null` discards the error if the glob matches nothing.
- `[ -z "$LATEST" ] && exit 1` — bail out if no file was found rather than
  running rclone against an empty string.
- `--log-file` + `--log-level INFO` writes a record instead of dumping to
  a terminal nobody is watching. Essential for scheduled jobs.
- `--min-age 8d` prunes old objects so R2 doesn't accumulate.

Watch a long upload from another session:
```
tail -f /var/log/rclone-backup.log
```

Schedule it AFTER vzdump finishes:
```
crontab -e
30 3 * * * /root/backup-to-r2.sh
```
Cron format: `minute hour day-of-month month day-of-week command`
So `30 3 * * *` = 03:30 every day. vzdump runs at 02:00 and takes about a
minute, so 03:30 leaves plenty of margin.

First upload of 4.8 GB over WiFi-as-WAN: roughly 38 minutes.

---

## PART 5 — LESSONS LEARNED

### 32. Narrow a fault by what DOES work
Three different 403s, three different causes. The breakthrough was
noticing that `ls` succeeded while `copy` failed. That single asymmetry
proved the endpoint, account, bucket name and credentials were all
correct — leaving only the permission model in question. Staring at the
failure told us less than comparing it against the success.

### 33. Least privilege will break things that "should" work
The token was correctly scoped to Object Read & Write on ONE bucket.
That is the right security posture, and it is exactly why `lsd` and the
pre-write bucket check both failed. Expect tightly-scoped credentials to
require extra flags. Do not loosen the token to make an error go away —
find the flag instead (`--s3-no-check-bucket` here).

### 34. Don't paste secrets into a chat window
A secret access key was pasted verbatim while sharing terminal output.
Rotated immediately: deleted the old token FIRST (so there is no window
where a leaked key is still live), then created a replacement and updated
rclone. Habit to build: redact anything after `secret_access_key`,
`password`, `token`, or `api key` before sharing output.

### 35. R2 storage class: Standard, not Infrequent Access
IA is cheaper per GB but adds a retrieval fee and a 30-DAY MINIMUM
storage duration — you pay 30 days even if the object is deleted after 2.
Rotating nightly snapshots is exactly the wrong shape for it. IA suits
write-once-keep-for-years archives.

### 36. Cloudflare Account tokens vs. User tokens
Account API tokens stay valid independent of your user; User tokens go
inactive if account membership changes. For an unattended backup job,
use an ACCOUNT token.

### 37. The Proxmox backup schedule dropdown defaults to "yearly"
Easy to click straight past. Set it explicitly.

### 38. vzdump runs on the HYPERVISOR, not in the guest
The host is what has access to the VM's disk image, so that is where
vzdump runs, where the files land, and therefore where rclone must be
installed and configured. Configuring rclone as a different user would
leave a root cron job unable to find the credentials.

---

## PART 6 — OPEN ITEMS

Immediate:
- [ ] Confirm the first R2 upload completed (~38 min)
- [ ] Add the cron entry for /root/backup-to-r2.sh
- [ ] **TEST A RESTORE** — restore the backup to VMID 101 and boot it.
      Proxmox can restore to a different VMID without touching VM 100, so
      this can be verified while production keeps running. Not yet done.

Open decision — encrypt before upload?
The vzdump image contains the ENTIRE VM, including ~/.openclaw/secrets.env,
credentials/, identity/, and API keys for seven providers.
Cloudflare encrypts at rest, so this is not about their carelessness — it
is about who holds the keys. An rclone `crypt` remote layered on top would
mean Cloudflare holds ciphertext they cannot read.
Trade-off: **lose the passphrase and the backups are permanently
unrecoverable.** No support ticket fixes that.
Setting it up later means re-uploading everything, so decide before the
next cycle rather than in a month.

Hardening (not started):
- [ ] SSH key auth for root, then `PermitRootLogin prohibit-password`
- [ ] fail2ban

Deferred:
- [ ] Disk 32 → 64 GB: snapshot FIRST, swapoff, remove /dev/sda5,
      growpart + resize2fs, replace with a 4 GB swapfile
- [ ] Re-price the 32 GB RAM kit given the 2026 shortage (~172% DRAM rise)
- [ ] Hermes migration — inventory the AWS host first, then size hardware
      to measured need

Housekeeping:
- [ ] Clean up /root/openclaw-telegram-repair-20260719T091122Z
- [ ] Identify the "Cloudflare Agent Token" (May 2026, all buckets,
      admin read-only) — know what created it and whether it is needed
- [ ] Consider `disable_checksum = true` to stop the 501-then-retry noise
