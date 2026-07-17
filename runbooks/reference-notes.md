# Homelab Reference Notes — Why We Did What We Did

A personal reference covering the naming decisions, every command, what each
flag means, and the fixes/lessons from building pve-node1 (Lenovo M720q).

Written so future-me understands the *reasoning*, not just the steps.

---

## PART 1 — NAMING DECISIONS (why we called things what we called them)

### `pve-node1` (the Proxmox hostname)
- **pve** = **P**roxmox **V**irtual **E**nvironment. Standard convention; anyone
  in the homelab/enterprise world sees "pve" and knows it's a Proxmox host.
- **node1** = this is the *first node*. Proxmox is built around the idea of a
  *cluster* of nodes. Even though I have one machine now, naming it `node1`
  leaves room for `node2`, `node3` later without renaming anything. Renaming a
  Proxmox host after the fact is painful, so we planned ahead.
- **Lesson:** name for the future you expect, not just today. Cheap to plan,
  expensive to rename.

### `homelab` (the Git repo)
- Simple, obvious, describes exactly what it holds. A repo name should tell you
  what's inside at a glance.

### `.local` suffix (e.g. `pve-node1.local`)
- `.local` is a reserved domain for local networks (mDNS). It signals "this name
  only means something on my own network," never routes to the public internet.
  Safe choice for a private box.

### The static IP `192.168.8.2`
- **192.168.8.x** is the network range the GL.iNet router creates by default.
- **.1** is always the router itself (the gateway).
- **.2** = the first "real" device address after the router. Low, memorable,
  easy to type. Servers get low static numbers by convention; phones/laptops get
  high numbers from DHCP. Keeps mental order.

### SSH key comment `pve-node1`
- The `-C "pve-node1"` on the key just labels *which machine this key belongs to*
  when you look at your list of keys on GitHub. If I have keys from 5 machines,
  the comment tells them apart. Purely for human readability.

---

## PART 2 — EVERY COMMAND, EXPLAINED

### Hardware inspection

```
dmidecode -t memory | grep -E "Size|Locator"
```
- `dmidecode` = **d**esktop **m**anagement **i**nterface decode. Reads hardware
  info straight from the system firmware (BIOS/UEFI tables). The authoritative
  source of truth for what hardware is physically present.
- `-t memory` = **t**ype filter, only show the *memory* section (not CPU, BIOS,
  etc.). Without it you'd get pages of everything.
- `|` = the "pipe." Takes the output of the command on the left and feeds it as
  input to the command on the right. One of the most important concepts in Linux.
- `grep` = search/filter text. Only prints lines that match.
- `-E` = use **E**xtended regular expressions (lets us use `|` inside the pattern
  to mean "OR").
- `"Size|Locator"` = the search pattern: show lines containing "Size" OR
  "Locator". This gave us each RAM stick's size and which slot it's in.
- **Why we ran it:** to settle whether the 16GB was 1x16 or 2x8 *definitively*,
  instead of guessing from a blurry photo. Software truth beats eyeballing.
- **What it revealed:** 2x 8GB, both slots full → to reach 32GB, must replace
  both with a 2x16GB kit (no free slot to just add one).

```
free -h
```
- `free` = show memory usage (RAM + swap).
- `-h` = **h**uman-readable (shows "15Gi" instead of "16302588 kB").
- Confirmed ~15GB usable RAM, dual-channel.

```
nproc
```
- Prints the **n**umber of **proc**essing units (CPU cores/threads available).
- Returned `6` → confirms the i5-8400T's 6 cores.

```
date
```
- Shows the system clock. We used it to rule out a wrong-clock cause for the
  SSL errors (wrong clocks break certificate validation). Clock was fine (UTC).

---

### The USB installer

```
lsblk
```
- **l**i**s**t **bl**ock devices — shows all drives and partitions.
- Used to *identify the USB stick by its size* before flashing, so we didn't
  accidentally erase the laptop's own drive. THE critical safety check before
  writing to any disk.

```
sudo dd if=<iso> of=/dev/sdX bs=4M status=progress oflag=sync
```
(The `dd` method — an alternative to the GUI flasher.)
- `sudo` = run as **su**peruser (root/admin). Writing directly to a disk device
  requires admin rights.
- `dd` = **d**ata **d**uplicator. Copies raw bytes from one place to another.
  Powerful and unforgiving — it does exactly what you say, including wiping the
  wrong disk if you point it there. "disk destroyer" is the nickname.
- `if=` = **i**nput **f**ile (the ISO you're writing).
- `of=` = **o**utput **f**ile (the USB device — `/dev/sdX`, NOT a partition like
  `/dev/sdX1`).
- `bs=4M` = **b**lock **s**ize; copy 4 megabytes at a time. Bigger blocks = faster.
- `status=progress` = show a live progress readout (otherwise dd is silent).
- `oflag=sync` = force each write to physically complete before continuing, so
  "done" really means done. Directly related to our USB lesson below.

```
sudo umount /dev/sdX*
```
- `umount` = **un**mount (note: no "n" — it's literally "umount"). Detaches a
  drive so it's safe to write to or remove.
- `/dev/sdX*` = the `*` is a wildcard meaning "all partitions on this device"
  (sdX1, sdX2, etc.).

---

### Networking diagnostics

```
ip route
```
- Shows the routing table — where network traffic goes. The `default via X`
  line is your gateway (router). We used this to discover the real network.

```
ip route | grep default
```
- Same, but filtered to just the `default` line (the gateway).

```
ip addr show   (or: ip addr)
```
- Shows every network interface and its IP address(es). `inet 192.168.x.x` is an
  IPv4 address. Used to find what IP a machine actually has.

```
ip link
```
- Shows network interfaces and their *physical link state*.
- `state UP` = interface enabled. `state DOWN` = disabled.
- `NO-CARRIER` = enabled but nothing's plugged in / no signal on the cable.
- **This is how we proved the apartment wall jack was dead:** the port was `UP`
  (enabled) but `NO-CARRIER` (no signal), with two different cables. Not the
  cable, not the laptop — the jack itself had no live connection.

```
ipconfig        (Windows equivalent)
ipconfig /all   (Windows, more detail)
```
- Windows version of checking network config. `/all` adds DHCP, DNS, MAC info.
- Confirmed the subnet mask `255.255.255.192` = a **/26** network (only ~62
  usable addresses), and the DNS suffix `wtsky.net` = WhiteSky managed WiFi.

**CIDR / netmask note (important concept):**
- `/24` = `255.255.255.0` = 254 usable addresses (the common home default).
- `/26` = `255.255.255.192` = 62 usable addresses.
- The number after the slash = how many bits are "locked" as the network part.
  Higher number = smaller network. Getting this wrong means devices can't talk.

---

### Repository / apt fixes (the big troubleshooting saga)

```
apt update
```
- Refreshes the list of *available* packages from all configured repositories.
  Does NOT install anything — just updates the catalog. Always run before
  installing or upgrading.

```
apt upgrade -y
```
- Actually installs the newer versions of packages you already have.
- `-y` = auto-answer **y**es to prompts (don't stop to ask "are you sure?").

```
apt install -y git
```
- Installs a package (here, git). `-y` again to skip the confirmation prompt.

```
sed -i 's/OLD/NEW/' <file>
```
- `sed` = **s**tream **ed**itor. Finds and replaces text in files.
- `-i` = edit the file **i**n place (change the actual file, not just print).
- `s/OLD/NEW/` = **s**ubstitute: replace OLD with NEW. The `/` are separators.
- **We used a `|` as the separator instead of `/`** in some commands
  (`s|OLD|NEW|`) because the text being replaced contained `/` characters (URLs).
  Using a different separator avoids confusing sed. Clever, legal, and common.
- **Lesson learned:** sed surgery on config files is error-prone. When a config
  is messy, it's often cleaner to DELETE it and write a fresh correct one than
  to patch it with more sed. (This is exactly what fixed our repo mess.)

```
rm -f <file>
```
- `rm` = **r**e**m**ove (delete) a file.
- `-f` = **f**orce: don't complain or prompt if the file is missing or
  write-protected. Safe here because we WANTED them gone regardless.
- **Caution:** `rm -f` deletes without asking. Always double-check the path.

```
cat > <file> << 'EOF'
...contents...
EOF
```
- `cat` normally prints a file. `cat >` *writes* to a file instead.
- `> <file>` = redirect output INTO this file (overwriting it completely).
- `<< 'EOF'` = a "heredoc" — everything you type until a line that says exactly
  `EOF` becomes the file's contents. `EOF` = "end of file" (just a marker word;
  could be any word, EOF is convention).
- The quotes around `'EOF'` mean "don't interpret $variables inside" — take the
  text literally. Important for config files with special characters.
- **Lesson learned:** pasting multi-line heredocs into a web shell can get
  mangled if the paste is interrupted. If the shell hangs at a `>` prompt, the
  closing `EOF` didn't land — type `EOF` on its own line to finish it. And
  ALWAYS `cat` the file afterward to verify it's correct before relying on it.

```
grep -H URIs /etc/apt/sources.list.d/*.sources
```
- `grep` = search. `-H` = print the filename (**H** for header) with each match,
  so you know which file each line came from.
- `*.sources` = wildcard: check every file ending in `.sources`.
- Used to see all repo URLs at once and confirm none still pointed at the paid
  `enterprise.proxmox.com`.

```
cat <file1> <file2>
```
- Prints multiple files back-to-back. We used it to review both repo files at
  once and verify correct spelling before `apt update`.

```
ls /usr/share/keyrings/ | grep proxmox
```
- `ls` = list directory contents. Piped to `grep proxmox` to filter to just the
  Proxmox-related files. Used to confirm the EXACT keyring filename
  (`proxmox-archive-keyring.gpg`, singular "keyring") after we'd typo'd it.

**The SSL/certificate lesson (conceptual, important):**
- `download.proxmox.com` shares a server/IP with `enterprise.proxmox.com`, and
  presented a certificate for the *enterprise* name. When we connected via
  `https://` to the download name, the cert didn't match → "certificate verify
  failed."
- **The fix:** use `http://` (not https) for these repos. Safe because Proxmox
  repos are cryptographically **GPG-signed** (the `Signed-By:` line points to a
  keyring). The signature proves the packages are authentic regardless of the
  transport encryption. This is exactly how Proxmox's own default config works.
- **Takeaway:** package authenticity (GPG signing) and transport encryption
  (TLS/https) are two SEPARATE protections. For signed repos, http is fine.

```
curl -v https://download.proxmox.com 2>&1 | grep -i "issuer\|subject\|SSL certificate"
```
- `curl` = fetch a URL from the command line. `-v` = **v**erbose (show the
  connection details including the certificate).
- `2>&1` = redirect "standard error" (stream 2) into "standard output"
  (stream 1) so grep can filter ALL the output, not just the normal part.
  (Error messages go to a separate stream by default; this merges them.)
- `grep -i` = **i**gnore case (match "SSL" or "ssl").
- `"issuer\|subject\|..."` = the `\|` means OR in basic grep (needs the
  backslash without `-E`).
- **This command diagnosed the whole SSL problem** — it showed the cert said
  `CN=enterprise.proxmox.com`, revealing the name mismatch.

---

### Git setup (server side)

```
git config --global user.name "Rich"
git config --global user.email "you@email.com"
```
- Sets your identity for commits. `--global` = apply to ALL repos for this user
  (stored in `~/.gitconfig`), not just one repo. Every commit is stamped with
  this name/email so history shows who did what.

```
ssh-keygen -t ed25519 -C "pve-node1"
```
- `ssh-keygen` = generate an SSH key pair (a private + public key).
- `-t ed25519` = key **t**ype. ed25519 is a modern, fast, secure algorithm.
  Preferred over the older RSA for new keys.
- `-C "pve-node1"` = a **c**omment label (which machine this key is from). Purely
  for human identification in your GitHub key list.
- **Produces two files:** `id_ed25519` (PRIVATE — never share, never commit) and
  `id_ed25519.pub` (PUBLIC — safe to give GitHub). The public key locks; the
  private key is the only thing that unlocks. You give out locks freely, keep
  the key secret.

```
cat ~/.ssh/id_ed25519.pub
```
- Prints the PUBLIC key so you can copy it to GitHub. `~` = your home directory.
- **Security rule:** only ever display/share the `.pub` file. The one WITHOUT
  `.pub` is the private key and stays on the machine forever.

```
git clone git@github.com:USER/homelab.git
```
- Downloads a copy of the repo and links it to GitHub for push/pull.
- Using the `git@github.com:` (SSH) form — not `https://` — so it authenticates
  with the SSH key we just made, no password typing needed.
- First connection asks to trust GitHub's fingerprint → type `yes`.

```
git status
```
- Shows what's changed in your working copy vs. the last commit. Your dashboard
  for "what have I modified, what's staged, what's committed."

**The core Git rhythm (memorize this):**
```
git add .                    # stage all changes ("." = everything here)
git commit -m "message"      # save a snapshot with a note (-m = message inline)
git push                     # upload commits to GitHub
git pull                     # download commits others (or another machine) made
```
- `git add` = choose what goes in the next snapshot.
- `git commit` = take the snapshot. `-m "..."` attaches the description inline.
- Good commit messages describe WHY, for future-you: "Fix repo SSL by switching
  to http + GPG" beats "update stuff."

---

## PART 3 — LESSONS LEARNED LOG (the war stories)

### 1. USB flash must FULLY complete before removal
- **Symptom:** M720q stuck on "Checking Media Presence" / tried network boot;
  USB appeared in the boot menu but wouldn't boot.
- **Cause:** pulled the USB before the write finished (missed a sudo password
  prompt, so the write never actually completed).
- **Fix:** re-flash, wait for the tool to say FULLY done. A progress bar at 100%
  is NOT the same as finished — there's a verify/sync step after.
- **Takeaway:** never remove flash media until explicitly told it's complete.

### 2. Boot order / "Checking Media Presence" after install
- **Symptom:** same "Checking Media Presence" message right after install.
- **Cause:** USB still plugged in and/or BIOS trying network boot before the SSD.
- **Fix:** remove USB, full power cycle. (If persistent: set the NVMe/"UEFI OS"
  to the top of the BIOS boot order.)
- **Takeaway:** "Checking Media Presence" = "I can't find a bootable OS, falling
  back to network." Usually means wrong boot source, not a broken install.

### 3. RAM: verify in software, don't guess from photos
- Confused two-sticks-vs-one from a blurry photo. `dmidecode` gave the truth
  instantly: 2x8GB, both slots full.
- **Takeaway:** for hardware facts, trust the software readout over eyeballing.

### 4. Apartment wall jack was dead
- **Symptom:** "Ethernet disconnected" despite cable plugged in; `ip link`
  showed `UP` + `NO-CARRIER` with two cables.
- **Cause:** the jack isn't wired to anything (common in apartments).
- **Fix:** don't rely on it — use our own router over the building WiFi.
- **Takeaway:** `NO-CARRIER` with a known-good cable = dead port, not your gear.

### 5. Shared/managed apartment WiFi (WhiteSky) needs your own router
- Managed building WiFi (WhiteSky/wtsky.net) typically has client isolation
  (devices can't see each other) — fatal for a home server.
- **Fix:** GL.iNet Beryl AX (GL-MT3000) in "repeater / WiFi-as-WAN" mode: joins
  the building WiFi as its uplink, creates our OWN private 192.168.8.x network
  where devices CAN talk to each other. No captive portal on WhiteSky-Prose
  (lucky).
- **Takeaway:** on any network you don't control, bring your own router to create
  a private LAN. This is also the foundation for future VLANs/firewalling.

### 6. Static IP device shows "0 clients" on the router — that's normal
- The M720q had a STATIC IP, so it never asked the router's DHCP for an address,
  so it didn't appear in the router's DHCP client list ("0 LAN clients").
- It was still reachable directly at 192.168.8.2:8006 the whole time.
- **Takeaway:** static-IP devices won't necessarily show in DHCP client lists.
  Don't assume "not in the list" = "not connected." Test the IP directly.

### 7. "No valid subscription" popup = NORMAL, not an error
- Proxmox is fully free and functional without a subscription. The popup is just
  a nudge toward the paid enterprise support tier. Click OK and move on.

### 8. Fresh Proxmox uses PAID enterprise repos by default → must switch
- Out of the box, apt points at `enterprise.proxmox.com` which needs a paid key
  → 401 Unauthorized errors.
- **Fix:** replace with the free `download.proxmox.com` no-subscription repos.

### 9. The SSL cert mismatch on download.proxmox.com
- Covered in detail above. Fix = use `http://` (repos are GPG-signed, so it's
  safe). Two separate protections: GPG signing (authenticity) vs TLS (transport).

### 10. Multi-line heredoc pastes get mangled + spelling matters
- Pasting `cat << EOF` blocks into the web shell broke when interrupted; typos
  crept in (`pve-nosubscription` vs `pve-no-subscription`; `keyrings` vs
  `keyring`).
- **Fix:** delete broken files, write fresh ones carefully, ALWAYS `cat` to
  verify before `apt update`. When a config is a mess, rewrite > patch.

### 11. First reboot after a big upgrade is SLOW — wait it out
- After 52 updates (+ possible new kernel), the reboot took a few minutes and
  the network came up last. Waiting fixed the "can't reach web UI."
- **Takeaway:** give a freshly-upgraded server 2–3 full minutes before assuming
  something's broken. Have the console (TV) attached during risky reboots so you
  can see what's actually happening.

---

## PART 4 — KEY CONCEPTS WORTH INTERNALIZING

- **Pipe `|`:** feed one command's output into another. The heart of Linux.
- **Redirect `>`:** send output into a file (overwrites). `>>` appends instead.
- **`2>&1`:** merge error stream into normal output (so you can filter/see both).
- **Wildcard `*`:** "match anything" in filenames (`*.sources` = all .sources).
- **`sudo`:** run as admin. Required for system-level changes.
- **GPG signing vs TLS:** authenticity of a package vs. encryption of the
  connection — separate, independent protections.
- **Static vs DHCP IP:** static = you hardcode it (servers); DHCP = the router
  assigns it (phones/laptops). Static devices may not show in DHCP client lists.
- **CIDR (/24, /26):** how big the network is. Must match your actual network or
  devices can't communicate.
- **Public vs private key:** share the public (lock) freely; guard the private
  (key) forever. Never commit private keys to a repo.
- **"Rewrite > patch" for broken configs:** a clean fresh file beats layering
  more edits onto a mangled one.
- **Verify with software, not eyeballs:** `dmidecode`, `ip link`, `cat` the file
  — confirm reality instead of assuming.

---

## PART 5 — WHERE TO PUT THIS IN THE REPO

Suggested homes:
- This whole file → `runbooks/reference-notes.md`
- The war stories (Part 3) also belong in → `runbooks/lessons-learned.md`
  (you've been adding to this already)
- Command explanations (Part 2) are handy as → `runbooks/command-reference.md`

Or keep it as one file. Whatever you'll actually read again is the right choice.

Commit it with something like:
```
git add runbooks/reference-notes.md
git commit -m "Add full reference notes: naming, commands, flags, lessons"
git push
```
