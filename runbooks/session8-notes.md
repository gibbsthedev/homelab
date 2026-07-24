# Session 8 — Why the Backups Kept Growing (a disk-space investigation)

Date: 2026-07-24

The nightly backup grew 4.8 → 6.2 → 7.3 GB over three days, which pushed
us over Cloudflare's free tier and made uploads take over an hour. This
session was a hunt for WHERE the space was going. Written to explain the
reasoning, because the commands are only useful if you understand what
question each one answers.

---

## PART 1 — THE PROBLEM AND WHY IT MATTERED

The Proxmox backup (vzdump) copies the whole VM disk into one compressed
file every night. That file was growing:

- Jul 22: 4.8 GB
- Jul 23: 6.2 GB
- Jul 24: 7.3 GB

Two reasons that's a problem:
1. **Cloudflare R2's free tier is 10 GB.** At 7.3 GB per snapshot, you
   can't keep even two without paying.
2. **Upload time.** 4.8 GB took 34 minutes over your connection. 7.3 GB
   takes well over an hour, every night, on a link that keeps dropping.

So the goal was: find what's taking the space, and remove what doesn't
need to be there.

---

## PART 2 — THE CORE SKILL: FOLLOWING DISK USAGE DOWN A TREE

This whole session is one technique repeated: **start at the top of the
filesystem, find the biggest thing, go into it, repeat.** You drill down
until you find the specific directories eating space.

The tool is `du` — **d**isk **u**sage. It adds up the size of everything
in a directory.

```
du -sh /some/path
```
- `-s` = **s**ummary: give me ONE total for the whole path, not a line for
  every file inside.
- `-h` = **h**uman-readable: "2.5G" instead of "2621440".

To compare several things and see which is biggest:
```
du -xsh /home/* | sort -h
```
- `/home/*` = every item directly inside /home.
- `sort -h` = sort by human-readable size, smallest to largest, so the
  biggest lands at the bottom where you'll see it.

### The two flags that mattered most

**`-x` — stay on one filesystem.**
Remember from last session: we mounted a separate 16 GB disk at
`~/.openclaw/backups/local`. Without `-x`, `du` walks INTO that mounted
disk and counts its contents too — which inflates the numbers and hides
what's actually on the main disk. `-x` tells `du` "don't cross into other
filesystems," so you measure only the disk you care about.

This is exactly why `du` on the backups folder showed 2.5G one way and
370M another. Pointed directly at the mount, it measured the mounted disk
(2.5G of archives). With `-x` from above, it measured only what's on the
root disk (370M of other files). Both were correct — they were answering
different questions.

**`.[!.]*` — catch hidden files.**
On Linux, anything starting with a dot is "hidden" — `.cache`, `.npm`,
`.local`, `.openclaw`. A plain `*` does NOT match these. So
`du -sh /home/ubuntu/*` would completely miss `.openclaw`, which is 6.4 GB.

`.[!.]*` is a glob that means "starts with a dot, but the second character
is NOT a dot" — which catches `.cache` and `.openclaw` while avoiding `.`
(the current directory) and `..` (the parent), which would cause `du` to
walk the whole system.

Simpler alternative that avoids the cryptic glob entirely:
```
du -xh --max-depth=1 /home/rich | sort -h
```
- `--max-depth=1` = show totals for immediate children only, don't recurse
  deeper. Lists everything, hidden or not, one level down.
- This is the command that finally revealed `/home/rich/.openclaw` at
  2.5 GB, after the glob version had returned nothing.

---

## PART 3 — WHAT WE ACTUALLY FOUND

The hunt went: whole disk → /home → each user → each user's directories.

### Finding 1 — /home was 12 GB but the obvious folders only explained 6
That gap is what told us something unexpected was hiding. **Data doesn't
vanish and space doesn't appear from nowhere.** When the total doesn't
match the sum of the parts you can see, keep drilling — the difference is
real and it's somewhere.

### Finding 2 — /home/rich had a whole stale copy of OpenClaw (2.5 GB)
This was the big one, and it's a migration leftover.

Recall the timeline: during the OpenClaw migration we first rsync'd
everything into `/home/rich/.openclaw`, THEN moved a copy to
`/home/ubuntu/.openclaw` (because the app expected the `ubuntu` user). The
original in `/home/rich` was never deleted.

Worse: the duplicate `rich` gateway service we killed back in session 3
had been running against THIS copy. So it wasn't just dead space — it was
the source of the duplicate-gateway problem too.

**How we proved it was safe to delete:**
```
sudo stat /home/rich/.openclaw
```
`stat` shows a file/directory's timestamps. The key line:
```
Modify: 2026-07-17 14:36:...
```
Last modified July 17 — a week ago, and before we disabled the duplicate
gateway on the 19th. Nothing had written to it since.

Then we confirmed the LIVE copy was current:
```
sudo stat /home/ubuntu/.openclaw | grep Modify
Modify: 2026-07-24 10:29:...
```
Modified today. So the ubuntu copy is the real, active one, and the rich
copy has nothing it lacks.

**stat's three timestamps, since they're confusing:**
- **Access** — last time it was READ (even `ls` updates this; nearly
  useless for "is this stale")
- **Modify** — last time the CONTENTS changed (this is the one you want)
- **Change** — last time metadata like permissions changed
- **Birth** — when it was created

### Finding 3 — leftover pre-repair copies
- `/home/ubuntu/.local/opt/google-chrome.pre-repair-...` (831 MB) — a
  backup copy of Chrome made before some repair on July 18, sitting next
  to the live 416 MB install. Chrome is reinstallable; a saved copy of it
  never needed backing up.
- `/home/rich/.npm` and `.npm-global` (665 MB) — the duplicate gateway's
  Node install.
- Old one-off repair snapshots under `.openclaw/backups` (~370 MB) — safety
  copies from specific past operations, left for OpenClaw to confirm.

### The pattern worth noticing
Almost none of the bloat was precious data. It was **duplicates and
reinstallable tooling** left behind by the migration and various repairs.
The genuinely irreplaceable stuff — agents, memory, workspace, state,
secrets — is about 5.5 GB. Everything above that was cruft nobody cleaned
up.

---

## PART 4 — THE CLEANUP

Snapshot first — you're deleting 2.5 GB that was, a week ago, your entire
agent brain. Proxmox UI → VM 100 → Snapshots → Take Snapshot (`pre-cleanup`).

```
sudo rm -rf /home/rich/.openclaw
sudo rm -rf /home/rich/.npm-global
sudo rm -rf /home/rich/.npm
sudo rm -rf /home/ubuntu/.local/opt/google-chrome.pre-repair-20260718T0121Z
```
- `rm` = remove. `-r` = recursive (delete a directory and everything in
  it). `-f` = force (don't prompt for each file).
- **`rm -rf` is the most dangerous command in common use.** It deletes
  immediately, permanently, no trash can, no undo. The only safety is
  reading the path three times before pressing Enter. The snapshot is the
  real safety net here.

Check the result:
```
df -h /
```
- `df` = **d**isk **f**ree — how full each filesystem is (as opposed to
  `du`, which sums up files). Dropped from 15G used to ~11G.

Then confirm nothing broke — OpenClaw still answering on Telegram, gateway
healthy — and delete the `pre-cleanup` snapshot so it doesn't consume
space itself.

### df vs du — the distinction is worth keeping straight
- **df** asks the FILESYSTEM how much space is used. Fast, authoritative
  for "how full is the disk."
- **du** ADDS UP files in a directory tree. Slower, tells you WHERE the
  space went.
- When they disagree, something interesting is happening — deleted-but-
  still-open files, or (as in session 6) files hidden under a mount.

---

## PART 5 — THE OTHER THING WE CONFIRMED: R2 IS OVER THE FREE TIER

The offsite side had its own problem. From the Proxmox host:
```
tail -20 /var/log/rclone-backup.log
rclone ls r2:homelab-backups/
```

Two findings:
1. **The last two nights' uploads FAILED** — the log showed DNS timeouts
   (`i/o timeout ... 1.1.1.1:53`) at 03:56 and 04:23. The cron job fired at
   03:30 while the NIC was hung (session 7), so it never reached Cloudflare.
   Not an rclone fault — a casualty of the driver bug.
2. **R2 holds three snapshots (~15.4 GB), over the 10 GB free tier.** The
   prune only removes objects older than 8 days, so nothing has aged out
   yet. This is now a small monthly charge unless we cut to one snapshot.

Decision still to make: keep one offsite (free) or keep three (a few cents
a month). Either is fine — it just needs to be chosen, not left to drift.

---

## PART 6 — LESSONS

### 54. Follow disk usage DOWN the tree
Finding where space went is one move repeated: `du -xsh <dir>/* | sort -h`,
find the biggest, go into it, repeat. You don't guess — you measure at
each level and let the numbers point you deeper.

### 55. `du -x` stays on one filesystem; without it, mounts skew everything
The same folder read 2.5G or 370M depending on whether `du` crossed into
the mounted backup disk. Both numbers were correct. When a disk has other
filesystems mounted inside it, `-x` is how you measure just the one you
mean.

### 56. `*` misses hidden files; the big stuff is usually hidden
`.openclaw`, `.cache`, `.local`, `.npm` are all dot-directories that a
plain `*` glob skips entirely — and they're exactly where the gigabytes
live. Use `--max-depth=1` (lists everything, hidden included) or the
`.[!.]*` glob.

### 57. When the total doesn't match the visible parts, keep drilling
/home was 12 GB but the folders we could see added to 6. That 6 GB gap was
the whole investigation. A mismatch between the total and the sum of parts
is a signal, not a rounding error.

### 58. `stat` proves whether something is stale
`Modify` time is the one that matters — when the contents last changed.
The stale OpenClaw copy hadn't been modified since July 17; the live one
was modified today. That comparison is what made deletion safe rather than
a guess. (`Access` time updates on mere reads, so it's nearly useless for
this.)

### 59. Migrations and repairs leave cruft; clean up afterward
Nearly all the bloat was duplicates and pre-repair copies left behind — a
whole second OpenClaw in /home/rich, a backup copy of Chrome, the
duplicate gateway's npm install. None of it was ever cleaned up. Budget a
cleanup pass after any migration or major repair.

### 60. df vs du answer different questions
`df` = how full is the disk (ask the filesystem). `du` = where did the
space go (add up the files). Reach for df to see IF there's a problem, du
to find WHERE it is.

---

## PART 7 — OPEN ITEMS

Immediate:
- [ ] Tonight's dump should land under 5 GB after the cleanup — confirm
- [ ] Delete the `pre-cleanup` snapshot once OpenClaw is confirmed healthy
- [ ] Decide R2 retention: one snapshot (free) or three (~$0.05/mo). The
      failed uploads mean R2 currently holds the 20th/21st/22nd only.
- [ ] Ask OpenClaw before deleting the ~370 MB of old repair snapshots
      under ~/.openclaw/backups (it created them; it knows what's referenced)

Carried over:
- [ ] SSH hardening — key auth, then PermitRootLogin prohibit-password,
      then fail2ban
- [ ] Encrypt backups before upload? (dump contains secrets.env,
      credentials/, identity/, seven providers' API keys)
- [ ] Clean up /root/openclaw-telegram-repair-20260719T091122Z
- [ ] Identify the May 2026 "Cloudflare Agent Token"
- [ ] disable_checksum = true in rclone.conf (stops the 501 retry noise)
- [ ] Remove the leftover Debian ISO from VM 100's virtual CD drive
- [ ] Watch VM memory (was 78% after 4 days uptime)

NIC hang (session 7) — verify the fix held:
- [ ] No new "Hardware Unit Hang" in dmesg over the coming days
- [ ] After any reboot, confirm the offloads stuck
- [ ] Note: the failed R2 uploads were caused by the NIC hang. Once the
      hang fix is confirmed, uploads should resume on their own.

Longer-term idea surfaced this session:
- Most of the VM is reinstallable tooling, not precious state. If dumps
  keep growing, consider backing up ONLY ~/.openclaw (the ~5.5 GB that
  actually matters) instead of the whole VM. Smaller and faster, but a
  more manual restore (rebuild OS + tools by hand).

Hermes migration — inventory done, decisions still pending (see session 7).
