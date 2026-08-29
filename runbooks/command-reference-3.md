# Command Reference — Part 3

Everything used since Part 2 (which covered network troubleshooting, services/
logs/ports, systemd scopes, login sessions, Tailscale, rclone/R2, and Proxmox
backups — all on the home lab through session 9).

This part covers the security track: building and running the Cowrie honeypot
on the Hetzner VPS, editing Cowrie's fake filesystem, reading and correlating
honeypot logs, safely handling captured malware, the SMTP + HTTP honeypot
layers, and the systemd persistence / OOM work that keeps it all alive.

Organized by topic, not chronologically, so it reads as a reference. Unless a
block says otherwise, admin commands run on the **real** VPS — SSH in on
**port 2200** as root, NOT port 22 (port 22 is the honeypot).

---

## 1 — TWO PORTS, TWO WORLDS (never confuse them)

```
ssh root@62.238.47.215 -p 2200    # the REAL server (OpenSSH / sshd)
ssh root@62.238.47.215            # the HONEYPOT (Cowrie / twistd) on port 22
```
- This is the single most important thing to keep straight on this box. Port
  2200 is the real admin door. Port 22 is Cowrie's fake shell — anything you
  type there runs *inside the emulation*, not on the real server.
- WHY the real SSH was moved off 22: so port 22 is free for Cowrie to present
  as bait. The move was tested and confirmed working on 2200 *before* Cowrie
  ever touched 22 — same "test the new door before locking the old one"
  discipline as the SSH-hardening plan in Part 2.
- **The real danger:** `exit` one too many times out of the real shell drops
  you back to your own laptop, and the next `ssh` without `-p 2200` lands you
  in the honeypot by mistake. Real admin commands you thought you ran may have
  run inside the jar instead. Always confirm which world you're in:

```
whoami        # you (root) vs. a fake cowrie user
hostname      # real host is ubuntu-4gb-hel1-2; Cowrie presents app-prod-01
ss -tlnp | grep -E ':(22|2200)\b'   # 22=twistd (honeypot), 2200=sshd (you)
```
- `ss -tlnp`: `-t` TCP, `-l` listening, `-n` numeric ports, `-p` owning
  process. The `\b` is a word boundary so `:22` doesn't also match `:2200` or
  `:2223`.

---

## 2 — RUNNING COWRIE (as the `cowrie` user, from `~/cowrie`)

Cowrie must NOT run as root — it's deliberately internet-exposed bait, so it
runs as an unprivileged user whose home holds the venv and fake filesystem.

```
su - cowrie
cd ~/cowrie
source /home/cowrie/cowrie-env/bin/activate    # NOT /root/cowrie-env — doesn't exist
```
- WHY `activate`: Cowrie's commands (`cowrie`, `fsctl`) are console scripts
  that live *inside* the Python virtualenv. Without activating it, bash says
  `command not found` because they aren't on the normal `$PATH`.

```
cowrie start
cowrie stop
cowrie restart
```
- The user-level way to control it. In normal operation systemd owns it
  instead (section 8), but these are what you use when hand-testing.

### The stale-process trap (this bites, repeatedly)
```
ps aux | grep twistd            # is a real Cowrie process alive?
ss -tlnp | grep -E ':(22|23|2200)\b'   # who actually holds the ports?
killall -9 twistd               # blunt but fine on a dedicated honeypot
```
- WHY this matters: `cowrie stop` printing "success" does NOT prove the
  process died. A stale `twistd` can survive holding ports 22/23 while a new
  start fails with "address already in use." Two `twistd` PIDs = two Cowries
  fighting over one pidfile.
- **The real fix was structural** — see section 8 (Type=simple). Until that
  was in place, the reliable workaround was: `stop` → confirm `ps` shows
  nothing → `kill` any survivors → `start`. Never a bare `restart` after a
  failed start.

---

## 3 — COWRIE'S FAKE FILESYSTEM (making believable bait)

Cowrie's fake shell reads its directory tree from a "pickle" file. The bundled
one is read-only inside the package, so the first step is copying it somewhere
writable and pointing the config at the copy.

```
python -c "from cowrie.core.resources import read_data_bytes; \
  open('var/lib/cowrie/fs.pickle', 'wb').write(read_data_bytes('fs.pickle'))"
```
- Run from `~/cowrie` as the `cowrie` user. This loads Cowrie's stock tree
  (a Debian-ish `/bin`, `/etc`, empty `/root`) and writes a private, editable
  copy under `var/lib/cowrie/`.
- Then in `etc/cowrie.cfg`, under `[shell]`: `filesystem = var/lib/cowrie/fs.pickle`
- WHY: without that config line, Cowrie keeps using the bundled read-only
  pickle and every edit you make silently vanishes on restart.

### Two-part file creation: content on the real disk, then registered
A fake file needs BOTH halves or it's broken — a directory entry (so it shows
in `ls`) AND loaded content (so `cat` returns something).

```
mkdir -p /root/fake_files                       # -p: make parents, don't error if exists
nano /root/fake_files/customer_list_private.csv # author the real bait file
wc -c /root/fake_files/customer_list_private.csv # EXACT byte count — needed next
```
- `wc -c` gives the byte count `fsctl touch` needs. A wrong size makes `ls -l`
  inside Cowrie show the wrong length, and a placeholder like `XXXXX` throws
  `ValueError: invalid literal for int()`.

```
cat > /path/file << 'EOF'
...text...
EOF
```
- A heredoc. The quoted `'EOF'` means no shell expansion — safe for `$`,
  backticks, and other literal text you want preserved exactly.

### Registering files inside the pickle (`fsctl`)
```
fsctl var/lib/cowrie/fs.pickle
```
- Opens an interactive editor with its own prompt, `fs.pickle:/$`. This is a
  **different interpreter than bash** — its commands only work here.
- **NEVER paste `fs.pickle:/$` lines into a real bash shell.** That's exactly
  how real `htop`, `df`, and `cat .aws/credentials` got run on the actual VPS
  during one session — the fake-filesystem commands hit the real box instead.

```
mkdir /home/acampbell/.local            # one level at a time — fsctl mkdir isn't always -p
touch /root/customer_list_private.csv 41460   # touch PATH SIZE — size must match wc -c
load /root/customer_list_private.csv /root/fake_files/customer_list_private.csv
```
- `touch PATH SIZE` creates the *directory entry* at that byte size. `load
  VIRTUAL REAL` attaches real content: first path is the virtual location an
  attacker sees, second is the real file on disk fsctl reads from.

```
chown 1000 /root/customer_list_private.csv    # SINGLE numeric uid only
chgrp 1000 /root/customer_list_private.csv    # group is a SEPARATE command
chmod 600 /root/customer_list_private.csv     # -rw------- , looks like a real secret
```
- **Gotcha caught the hard way:** `chown 1000:1000` (the normal Linux
  colon-pair form) FAILS in fsctl with `Incorrect uid`. Cowrie's fsctl wants a
  single numeric uid for `chown` and a separate `chgrp` for the group. After
  two identical failures, switching to separate commands was the fix — two
  identical errors is the signal to stop guessing at syntax and change
  approach.
- The uid `1000` maps to a fake user (`acampbell`) that exists only inside the
  pickle's `/etc/passwd`, NOT on the real box — which is why `chown acampbell`
  on the *real* disk failed with `invalid user`. Ownership of the source files
  on the real disk is irrelevant to what attackers see; only the ownership set
  *inside* fsctl matters.

---

## 4 — READING COWRIE LOGS

Logs live in `/home/cowrie/cowrie/var/log/cowrie/`. Two formats: `cowrie.log`
(human text) and `cowrie.json` (one JSON object per event). **Prefer the
JSON** — its field names are stable, while the text log's wording changes.

```
cd /home/cowrie/cowrie/var/log/cowrie
```
- **Critical:** logs rotate daily at midnight UTC. `cowrie.json` is TODAY only;
  older data moves to dated files like `cowrie.json.2026-08-23`. For any
  all-time question, glob every file with `cowrie.json*` — querying only the
  current file silently misses history with no warning. (This caused a real
  "where did my data go?" scare when a 1092-hit IP showed 0 in the fresh file.)

### The three core counts
```
grep -h '"eventid":"cowrie.session.connect"' cowrie.json* | wc -l   # connections
grep -h '"eventid":"cowrie.login.success"'   cowrie.json* | wc -l   # successful logins
grep -h '"eventid":"cowrie.command.input"'   cowrie.json* | wc -l   # commands run
```
- `-h` suppresses the filename prefix grep normally adds when searching
  multiple files, so `wc -l` counts clean. `eventid` is the stable key to
  filter on.

### Top attacker IPs
```
grep -h '"src_ip":"' cowrie.json* | sed 's/.*"src_ip":"//;s/".*//' \
  | sort | uniq -c | sort -rn | head -20
```
- The `sed` strips everything before and after the IP value, leaving just the
  address. `sort | uniq -c | sort -rn` is the workhorse idiom from Part 2 —
  group, count, rank by frequency, highest first.
- **Always subtract your own test IP before trusting the ranking** — your own
  connections can outrank real attackers.

### Reconstructing one attacker's whole session ("zoom in")
```
grep '"session":"4fd8beaebb66"' cowrie.json
```
- Every event carries a `session` hex ID unique to one connection. Because the
  log is written in event order, this output IS the timeline — no sorting
  needed. This is the "zoom in" move; the ranking commands above are "zoom out."

### De-duplicating an attacker's commands
```
grep "117.50.122.20" cowrie.json* | grep '"eventid":"cowrie.command.input"' \
  | sed -E 's/.*"input":"//;s/",".*//' | sort | uniq
```
- Cowrie logs each command twice in the text log (`CMD:` and `Executing
  command`), so working from JSON `input` fields plus `sort | uniq` collapses
  the same `uname -a` repeated 200 times into one line.
- Note: `grep "IP" cowrie.log` matches the letters I-P, not an address — a real
  gotcha that prints nothing and looks like "no data" when it's a typo.

### Live tailing without drowning in bot noise
```
tail -f cowrie.log | grep --line-buffered -E "file_download|direct-tcp|Command found" \
  | grep --line-buffered -v -E "uname|busybox|enable|linuxshell"
```
- `--line-buffered` is essential: without it, grep in a pipe waits for a full
  4KB block before showing anything, so a live tail appears frozen.
- `-E` extended regex (so `|` means OR), `-v` inverts (drops the noisy
  repetitive bot greetings you're tired of seeing).

### Proving two indicators are one campaign (correlation)
```
comm -12 <(grep 'INDICATOR_1' cowrie.json | grep -oE '"src_ip":"[^"]+"' | sort -u) \
         <(grep 'INDICATOR_2' cowrie.json | grep -oE '"src_ip":"[^"]+"' | sort -u)
```
- `comm -12` prints only lines common to both sorted lists (set intersection);
  `-1`/`-2` would suppress column 1 or 2. The `<(...)` is process substitution
  — it feeds each command's output in as if it were a file.
- WHY: this is how you prove a cluster. Used to confirm a 4-IP SMTP-relay
  botnet by intersecting IPs that shared BOTH a relay target and an SSH client
  fingerprint (`hassh`) — two independent shared signals, not a coincidence.

---

## 5 — TTY REPLAYS AND CAPTURED FILES (handle malware safely)

### What they actually typed
```
cd ~/cowrie
strings var/lib/cowrie/tty/<sha256>
```
- Cowrie records each interactive session as a binary TTY file. `strings`
  pulls out the readable text and drops the control codes (arrow-key escapes
  like `^[[A`). `cat -v` shows the same but noisier. (Run from `~/cowrie` —
  a wrong CWD gives "no such file.")

### Cataloging the downloads folder
```
cd ~/cowrie
for f in var/lib/cowrie/downloads/*; do
  echo "=== $(basename $f) ==="; ls -lh "$f"; file "$f"; echo
done
sha256sum var/lib/cowrie/downloads/*
```
- `file` reports what each capture really is by its magic bytes — ELF binary
  (and which CPU arch), shell script, ASCII text, MP4, etc. — regardless of
  its name.
- `find var/lib/cowrie/downloads/ -type f -size +10k` filters to real
  payloads; anything under ~10KB is usually a marker or a failed grab.

```
head -c 32 var/lib/cowrie/downloads/<hash> | xxd
```
- Dumps the first 32 bytes as hex. `ftyp isom` / `avc1mp41` confirmed a real
  MP4 container (the "kitten video") rather than a renamed executable — a way
  to verify a file's true type before trusting it.

### The hard rules for this directory
- **NEVER** `chmod +x` and run anything here. **NEVER** `wget` a payload URL
  (like `https://217.60.195.113/sh`) from the VPS "to see it" — that fetches
  live malware onto your only server. Hash it, `file` it, read scripts/keys
  with `cat`, check the hash on VirusTotal — that's the whole safe workflow.
- **A VirusTotal miss ≠ unknown/safe.** Some families (redtail) rebuild their
  ELF binaries constantly, so the binary hash drifts while the stable
  artifacts don't. Identify by those instead: filenames (`clean.sh`,
  `setup.sh`, `redtail.*`), the embedded SSH key comment, the `hassh` client
  fingerprint, and the C2 address.

---

## 6 — THE SMTP HONEYPOT (mailoney)

```
ss -tlnp | grep 12525
systemctl status mailoney.service
```
- Mailoney is a fake SMTP server bound to `127.0.0.1:12525` — localhost only,
  never exposed to the internet directly. Cowrie redirects an attacker's
  `direct-tcp` SMTP-relay attempts to it internally.
- WHY localhost-only: there's no reason to expose SMTP on the public NIC; the
  only thing that should ever connect is Cowrie, locally. The Cowrie log line
  `redirected direct-tcp ... to <ip>:25 to 127.0.0.1:12525` proves the handoff.

```
# query mailoney's DB as the mailoney user (its files are mode-locked to it)
python3 -c "import sqlite3; c=sqlite3.connect('/home/mailoney/mailoney/mailoney.db'); \
  print(c.execute('SELECT COUNT(*) FROM smtp_sessions').fetchone()[0])"
```
- Running this as the `cowrie` user fails with `unable to open database file` —
  that's correct isolation (each honeypot's files are readable only by its own
  user), not a bug. Query mailoney's data as `mailoney`.

---

## 7 — THE HTTP HONEYPOT (fake WordPress) + nginx

A small Python server logs every request and returns believable WordPress
responses; nginx sits in front of it on port 80.

```
systemctl daemon-reload        # REQUIRED after writing any new unit file
systemctl enable --now wp-honeypot   # enable (start at boot) + start now, in one
```
- The Python logger binds `127.0.0.1:8080` only — the outside world talks to
  nginx on :80, which proxies inward. It reads the real client IP from the
  `X-Real-IP` header nginx sets; without that, every log line would wrongly say
  `127.0.0.1`.

```
nginx -t                # parse/validate config WITHOUT applying it
systemctl reload nginx  # apply with zero downtime
```
- `nginx -t` first is the habit — it catches a broken config before you reload
  into an outage. The proxy config forwards POST bodies (`proxy_pass_request_body
  on`) so credential-stuffing attempts against `wp-login.php` are captured, and
  sets `X-Real-IP $remote_addr` to feed the logger the true source.

```
tail -f /var/log/wp-honeypot/http.jsonl
```
- One JSON object per HTTP request, append-only. Must be run on the REAL VPS —
  this path doesn't exist inside Cowrie.

### logrotate — one real gotcha
```
logrotate -d /etc/logrotate.d/wp-honeypot   # -d = dry run, shows what WOULD happen
```
- **Do NOT wrap Cowrie's JSON logs in logrotate** — Cowrie already dates its own
  files, and a `rotate 14` would silently delete captured evidence after 14
  days. Only the HTTP honeypot's `http.jsonl` gets rotated, with `copytruncate`
  (copy then zero the live file so the running process keeps writing to the
  same handle) and a very high `rotate` count so nothing is discarded.

---

## 8 — PERSISTENCE & THE OOM INCIDENT (systemd)

### Why Type=simple beat Type=forking
The honeypot's first systemd unit used `Type=forking` and suffered a recurring
orphan-process bug: systemd tracked the wrong PID after the fork, so it could
report `dead` while Cowrie was actually serving traffic — meaning
`Restart=on-failure` would never fire if the real process died. The fix was
`Type=simple` with a direct, non-daemonizing `ExecStart`:
```ini
[Service]
Type=simple
ExecStart=/usr/bin/authbind --deep /home/cowrie/cowrie-env/bin/twistd \
  --umask=0022 --nodaemon --logger cowrie.python.logfile.logger cowrie
Restart=on-failure
```
- `--nodaemon` keeps twistd in the foreground so systemd owns the exact process
  — no fork, no PID-guessing. `authbind --deep` lets the non-root `cowrie` user
  bind privileged ports 22/23; it's invoked directly here because the old
  `AUTHBIND_ENABLED=yes` env var only worked via the wrapper script this setup
  bypasses.
- **Verify health with a three-way PID match:** the PID in `ps aux | grep
  twistd`, the PID in `ss -tlnp` on the ports, and `Main PID` in `systemctl
  status` should all be the same number. A single "running" status is not
  proof, given it was already shown wrong in both directions.

### Diagnosing the OOM crash-loop
```
dmesg -T | grep -iE 'oom|kill|error|reset|panic' | tail
```
- `dmesg -T` (human-readable timestamps) is the FIRST call for anything the
  kernel did — same principle as the NIC bug in Part 2. This surfaced
  `oom_kill_process constraint=CONSTRAINT_MEMCG ... oom_memcg=.../cowrie.service`
  — systemd's own memory cgroup killing Cowrie at its `MemoryMax=512M` cap,
  over and over, which looked like an outage from outside while the host stayed
  healthy.

```
ps -o pid,rss,etime -p <pid>   # rss = memory, etime = how long it's been alive
```
- **Key lesson:** RSS pinned at the cap LOOKS like an active crash-loop, but
  `etime` showing 6+ hours of uptime proved it had actually stabilized at the
  ceiling. Check uptime, not just memory, before concluding a process is dying.

### Raising the cap safely (the systemctl edit gotcha)
```
systemctl edit cowrie.service        # creates an override drop-in
systemctl daemon-reload
systemctl restart cowrie.service
systemctl show cowrie.service -p MemoryMax,MemoryHigh,MemoryCurrent
```
- WHY `systemctl edit` (not editing the unit directly): it writes a drop-in
  under `/etc/systemd/system/cowrie.service.d/override.conf`, so a package or
  reinstall can't wipe your change.
- **The trap:** the editor shows the current unit's settings as *commented*
  reference lines. Those `# MemoryMax=...` lines are NOT the file you're
  writing — leaving the override empty or fully commented does nothing,
  silently. You must type real, uncommented directives:
  ```ini
  [Service]
  MemoryMax=1536M
  MemoryHigh=1200M
  ```

```
watch -n 5 'systemctl show cowrie.service -p MemoryCurrent,MemoryMax,ActiveState'
```
- `watch -n 5` re-runs every 5 seconds to confirm memory now sits below the new
  ceiling instead of climbing into a kill. (Ctrl+C on `watch` is safe — it only
  stops watching.)

### Optional swap safety net (considered, not applied)
```
fallocate -l 1G /swapfile          # allocate a 1GB file
chmod 600 /swapfile                 # root-only, or swapon refuses it
mkswap /swapfile                    # format it as swap
swapon /swapfile                    # activate
echo '/swapfile none swap sw 0 0' | tee -a /etc/fstab   # persist across reboot
```
- Left as a documented option — the raised cgroup cap already resolved the
  actual symptom, so swap wasn't needed. Included here because it's the natural
  next hardening step if memory pressure returns.

---

## 9 — THINGS DELIBERATELY NOT RUN (and why)

Discipline matters more on an internet-exposed box than anywhere else in the
lab:
- **Never** execute anything in `downloads/` — it's live malware, inert only
  because it's never run.
- **Never** `wget`/`curl` an attacker's payload URL from the VPS — that pulls
  malware onto your only server.
- **Never** use a captured private key to connect to an attacker's C2 — it's
  someone else's infrastructure; capture and document only.
- **Never** run real `apt`/Samba/NFS/HTTP uploads "for" an attacker who asks
  (via echoed commands) — that turns the honeypot into free hosting for them.
- **Never** paste `fsctl` (`fs.pickle:/$`) commands into a real bash shell —
  they'll hit the real box.

---

END. Detailed per-session narratives are in `runbooks/session12-notes.md`
through `session18-notes.md`; the deep operator manual for the honeypot box is
the project's honeypot command explainer. This file is the topic-organized
quick reference.
