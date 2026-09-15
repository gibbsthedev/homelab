# SESSION 31 — Sensor Remediation: Fixing the Gaps the Analysis Found

**Date:** 2026-09-15
**Box:** Hetzner Cloud VPS `ubuntu-4gb-hel1-2` (62.238.47.215)
**Continuity:** Follows `SESSION-30-full-threat-intel-analysis-2026-09-15.md`, which
identified four sensor defects. This session fixes them.

**Outcome:** three fixed and verified, one attempted and unresolved (documented in §4
with the diagnostic trail, not silently dropped).

---

## 1. Mailoney source-IP blindness — FIXED

### The defect

Every one of 517 SMTP sessions across 24 days recorded `127.0.0.1` as the source.
Zero real attacker IPs. The `credentials` table was empty.

**Root cause:** mailoney was bound to loopback and fed by an iptables NAT redirect:

```text
ExecStart=… main.py -i 127.0.0.1 -p 12525 …
-A PREROUTING -p tcp --dport 25 -m conntrack --ctstate NEW \
   -m limit --limit 20/min --limit-burst 10 -j REDIRECT --to-ports 12525
```

`REDIRECT` rewrites the packet's destination and hands the socket a loopback peer, so
`addr[0]` — which `mailoney/core.py:129` logs as the source — was always `127.0.0.1`.
The data was never captured; it is not recoverable retroactively.

**Secondary defect found while reading the rule:** `--limit 20/min --limit-burst 10`
was silently dropping any SMTP traffic above 20 connections/minute. Beyond losing
attribution, an unknown share of real traffic was never reaching the sensor at all.

### Why the source code mattered

```bash
grep -rn "bind_ip\|'-i'" mailoney/config.py mailoney/core.py
```

```text
mailoney/config.py:19:    bind_ip: str = Field(default="0.0.0.0")
mailoney/core.py:361:        '-i', '--ip',
mailoney/core.py:129:  logger.info(f"Connection from {addr[0]}:{addr[1]} …")
```

Mailoney already defaults to `0.0.0.0` and accepts `-i`. It logs the true peer
address. Nothing needed patching in the application — only the deployment was wrong.

### The fix

Port 25 is privileged and the service runs as an unprivileged user, so binding it
directly requires granting the capability rather than running as root:

```bash
cp /etc/systemd/system/mailoney.service /etc/systemd/system/mailoney.service.bak
```

```diff
-ExecStart=…/python main.py -i 127.0.0.1 -p 12525 -s app-prod-01 --mail-dir …
+ExecStart=…/python main.py -i 0.0.0.0    -p 25    -s app-prod-01 --mail-dir …

 MemoryMax=256M
+AmbientCapabilities=CAP_NET_BIND_SERVICE
+CapabilityBoundingSet=CAP_NET_BIND_SERVICE
```

- `AmbientCapabilities=CAP_NET_BIND_SERVICE` — lets the unprivileged `mailoney` user
  bind a port below 1024. The correct alternative to `User=root`.
- `CapabilityBoundingSet=…` — caps the maximum capability set to just that one. Added
  as defence in depth; verified not to break startup.

Then remove the redirect from **both** the live table and ufw's persistent config —
missing the second would restore it on the next reboot:

```bash
iptables -t nat -D PREROUTING -p tcp -m tcp --dport 25 \
  -m conntrack --ctstate NEW -m limit --limit 20/min --limit-burst 10 \
  -j REDIRECT --to-ports 12525

cp /etc/ufw/before.rules /etc/ufw/before.rules.bak2
python3 -c "
import pathlib, re
p = pathlib.Path('/etc/ufw/before.rules'); t = p.read_text()
t2 = re.sub(r'# mailoney SMTP redirect.*?COMMIT\n\n', '', t, flags=re.S)
p.write_text(t2); print('removed' if t2 != t else 'PATTERN NOT FOUND')
"
```

**And open port 25 in the firewall** — it had never been opened, because traffic only
ever arrived via the redirect, which bypassed ufw's filter chain:

```bash
ufw allow 25/tcp
systemctl daemon-reload && systemctl restart mailoney
```

### Verification

```text
LISTEN 0  10  0.0.0.0:25  users:(("python",pid=33123,fd=3))
```

```text
(518, '2026-09-15 19:37:00', '104.241.55.33', 25)   <- real source IP
(517, '2026-09-13 09:47:39', '127.0.0.1',    12525) <- the old, blind pattern
```

Session 518 records the actual connecting address. **Fixed.**

> Note: removing the NAT rule also removed the 20/min rate limit. Expect noticeably
> more SMTP volume — both real attribution *and* traffic that was previously being
> dropped.

> **LESSON — a NAT `REDIRECT` destroys source attribution.** If a sensor's whole
> purpose is recording who connected, it must bind the real port directly.
> `AmbientCapabilities` makes that safe without running as root.

---

## 2. Cowrie unbounded downloads — FIXED

### The defect

`tmp_290ordr` reached **690,503,425 bytes** before failing, and was a material
contributor to the disk exhaustion investigated in session 26. Analysis showed it was
an incomplete MP4 transfer — the honeypot being used as free bandwidth, not attacked.

The setting existed but was commented out:

```text
etc/cowrie.cfg:87:  #download_limit_size = 10485760
```

Default is `0` — unlimited.

### The fix

```bash
cp etc/cowrie.cfg etc/cowrie.cfg.bak
```

```diff
-#download_limit_size = 10485760
+download_limit_size = 10485760
```

The surrounding comments confirm the semantics are exactly what was needed:

```text
# Maximum file size (in bytes) for downloaded files to be stored in 'download_path'.
# A value of 0 means no limit. If the file size is known to be too big from the start,
# the file will not be stored on disk at all.
```

**Tradeoff, stated explicitly:** 10 MB truncates anything larger. The largest
legitimate ELF captures in the corpus are a few hundred KB, so no real malware sample
is at risk — but a genuinely large payload would now be lost. Given the corpus, that
is the right trade.

**Fixed and verified** (`systemctl restart cowrie` → `active`).

---

## 3. Dionaea HTTP fingerprint — FIXED (with a caveat)

### The defect

Dionaea's HTTP module answered on 443 with `Server: nginx` while serving a Python
`SimpleHTTPServer`-style body:

```text
Server: nginx
…
<title>Directory listing for /</title>
```

Real nginx never produces that markup. Trivially detectable.

### Three compounding causes

```yaml
global_headers:
  - ["Server", "nginx"]        # claims nginx, no version
template:
  enabled: false               # nginx-styled templates exist but are disabled
  path: "var/lib/dionaea/http/template/nginx"
root: "var/lib/dionaea/http/root"   # empty -> every / request hits the autoindex
```

The `autoindex.html.j2` and `error.html.j2` templates were already present and are
genuinely well-made — the autoindex reproduces nginx's exact column spacing and
`dd-Mmm-YYYY HH:MM` date format; the error page matches nginx's real structure with
`{{ values.full_name|default("nginx") }}` in the footer.

### Attempted: enable the templates — blocked

```bash
docker exec dionaea python3 -c "import jinja2"
# ModuleNotFoundError: No module named 'jinja2'
docker exec dionaea pip3 install jinja2
# exec: "pip3": executable file not found in $PATH
```

**jinja2 is not present in the `dinotools/dionaea` image, and neither is pip.**
Setting `enabled: true` silently does nothing — the module falls back to built-in
pages. Enabling it would require a custom image build, which is disproportionate here.

Reverted, with an explicit note left in the config so this is not re-attempted blindly:

```yaml
      # NOTE: jinja2 is NOT present in the dinotools/dionaea image -- setting this
      # to true silently does nothing. Requires a custom image build to enable.
      enabled: false
```

### What was fixed instead

**1. Version the `Server` header** to match the existing PHP fingerprint:

```diff
-  - ["Server", "nginx"]
+  - ["Server", "nginx/1.18.0"]
```

`nginx/1.18.0` is Ubuntu 20.04 stock, coherent with the `X-Powered-By:
PHP/5.5.9-1ubuntu4.5` header already configured. **Consistency across the fingerprint
matters more than picking a current version.**

**2. Give the root real content** so `/` returns a page instead of an autoindex. This
is the higher-value half: 62 of the captured HTTP requests hit `/`, only a handful hit
nonexistent paths.

```bash
cat > /opt/dionaea/var/lib/dionaea/http/root/index.html   # stock nginx welcome page
chmod 644 …/index.html && docker restart dionaea
```

### Verification

```text
HTTP/1.1 200 OK
Server: nginx/1.18.0
Content-Type: text/html
Content-Length: 612
…
<title>Welcome to nginx!</title>
```

**Caveat, stated plainly:** the 404 page is improved but still not nginx-authentic —
it now returns XHTML (`<title>404 - Not Found</title>`) rather than the Python
directory-listing markup, but real nginx uses `<center><h1>404 Not
Found</h1></center>`. Without jinja2 this cannot be fully fixed. **The tell is
reduced, not eliminated.**

---

## 4. The `345gs5662d34` honeypot probe — ATTEMPTED, NOT RESOLVED

### The defect

`345gs5662d34 / 345gs5662d34` (519 attempts in 24 days) is a deliberately absurd
credential pair used by scanners to *fingerprint honeypots*: a real system rejects it,
a permissive honeypot accepts it and reveals itself. This box accepts it.

### The reasoning behind attempting a narrow fix

Cowrie's permissiveness is load-bearing — it is why the session-27 loader was
captured, why 1,694 downloads exist, and why a human operator explored the decoys.
Blocking one exact credential pair is surgical: every other combination keeps working,
so there is no capture-rate cost.

### The config change (correct, and verified correct in isolation)

`auth_class = UserDB` with rules in `etc/userdb.txt`, first match wins. The file
already contained a working negation example (`root:x:!root`):

```diff
 root:x:!root
+345gs5662d34:x:!345gs5662d34
 root:x:*
 admin:x:*
 *:x:*
```

Order matters — the deny must precede the `*:x:*` catch-all.

`adduser()` parses a leading `!` as `policy = False`, and `checklogin()` iterates an
`OrderedDict` in file order. Running Cowrie's own auth logic against the real file
confirms the rules are correct:

```text
--- loaded rules in order:
   user=b'root'          pass=b'root'          -> False
   user=b'345gs5662d34'  pass=b'345gs5662d34'  -> False
   user=b'root'          pass=b'*'             -> True
   user=b'admin'         pass=b'*'             -> True
   user=b'*'             pass=b'*'             -> True

345gs/345gs  -> False      <- correctly denied
root/root    -> False      <- correctly denied
root/other   -> True
testuser/x   -> True
```

**The configuration is right.** `UserDB.checklogin()` denies the pair.

### Why it still does not work in practice

Live testing against the running service kept succeeding, and the logs show why:

```text
cowrie.login.success | user: 'testuser'     | pass: None
cowrie.login.success | user: '345gs5662d34' | pass: None
cowrie.login.success | user: 'root'         | pass: 'hi3518'   <- a real attacker
```

**`pass: None`.** No password was ever sent, even with
`-o PubkeyAuthentication=no -o PreferredAuthentications=password`. Verbose client
output gives the exact reason:

```bash
ssh -o PubkeyAuthentication=no -o KbdInteractiveAuthentication=no \
    -o PreferredAuthentications=password -v 345gs5662d34@62.238.47.215
```

```text
Authenticated to 62.238.47.215 ([62.238.47.215]:22) using "none".
```

**Cowrie accepts the SSH `none` authentication method.** `none` is the probe a client
sends first to discover which auth methods a server supports — a real server answers
with a list of acceptable methods; Cowrie answers by granting the session outright.
Authentication completes before any password or key is ever offered, so
`UserDB.checklogin()` is never reached. No client-side flag can defeat this, because
`none` is attempted before every method the flags control.

The `root/hi3518` line proves password auth works for attackers that do negotiate it;
the userdb rule simply is not on the code path when `none` succeeds first.

### Status and the broader finding

**Unresolved.** The config is in place and correct; it will deny the probe whenever a
password *is* sent, which includes a meaningful share of real scanners. It does not
deny key-based or keyboard-interactive attempts.

> **This is itself a finding worth recording: Cowrie grants sessions on the SSH
> `none` auth method, bypassing `userdb.txt` entirely.** Any credential policy in that
> file is only enforced when a client actually negotiates password authentication —
> and a client that accepts the `none` result never does. That has implications well
> beyond this one probe: it means `userdb.txt` cannot be relied on as an access
> control of any kind, and it is very likely why the corpus shows 35,195 login
> "successes" against only 474 failures (§1 of session 30).
>
> It is also a realism tell in its own right, independent of any credential: a real
> SSH server does not authenticate a client that offered nothing.

Stopped here deliberately — this is a minor realism tell, and the three fixes above
addressed real data loss and real detectability. Continuing to chase Cowrie's auth
dispatch was disproportionate to the value.

---

## 5. Files changed

| File | Change | Backup |
|---|---|---|
| `/etc/systemd/system/mailoney.service` | Bind `0.0.0.0:25`, add `CAP_NET_BIND_SERVICE` | `.bak` |
| `/etc/ufw/before.rules` | Removed the mailoney `*nat` redirect block | `.bak2` |
| iptables NAT (live) | Deleted the `--dport 25 REDIRECT` rule | — |
| ufw rules | `ufw allow 25/tcp` | — |
| `/home/cowrie/cowrie/etc/cowrie.cfg` | `download_limit_size = 10485760` | `.bak` |
| `/home/cowrie/cowrie/etc/userdb.txt` | Added `345gs5662d34:x:!345gs5662d34` | `.bak` |
| `…/dionaea/services-available/http.yaml` | `Server: nginx/1.18.0`, `full_name`, jinja2 note | `.bak` |
| `…/dionaea/http/root/index.html` | **NEW** — stock nginx welcome page | — |

---

## 6. Still open

- **Cowrie's `none`-auth acceptance** (§4) — sessions are granted before any
  credential is offered, so `userdb.txt` is unenforceable. Worth understanding
  properly: it affects any future credential policy, is a realism tell on its own,
  and likely explains the corpus-wide 35,195:474 success:failure ratio.
- **Dionaea 404 page** — cannot be made nginx-authentic without a custom image
  carrying jinja2.
- **Verify mailoney captures credentials now.** The `credentials` table was empty for
  24 days. With real connections arriving, confirm AUTH attempts are being recorded —
  the source-IP fix does not guarantee the credential path works.
- **Confirm the fixes survive a reboot** — particularly the ufw/NAT changes and the
  `CAP_NET_BIND_SERVICE` grant. Session 28's lockout came from an unattended reboot
  landing mid-change.

---

## 7. Lessons

- **A NAT `REDIRECT` destroys source attribution.** Any sensor whose purpose is
  recording *who* connected must bind the real port. `AmbientCapabilities` is the
  right way to do that without root.
- **Read the whole firewall rule, not just its target.** The `--limit 20/min` clause
  on the mailoney redirect was silently dropping traffic — a second defect found only
  by reading a rule being removed for an unrelated reason.
- **Removing a NAT rule means checking both the live table and the persistent
  config.** Missing `before.rules` would have restored it on the next reboot.
- **A commented-out setting is not a default.** `download_limit_size` shipped
  commented out, defaulting to unlimited — which is how a 690 MB file landed.
- **Verify a config change is actually reachable before trusting it.** `enabled: true`
  for Dionaea's templates parsed fine, restarted fine, and did nothing, because jinja2
  is absent from the image. A silent no-op is worse than an error.
- **Testing a rule in isolation proves the rule, not the system.** `UserDB.checklogin()`
  returned `False` exactly as intended while the live service kept accepting the same
  credentials — because the auth method in use never consults `UserDB`.
- **`pass: None` in an auth log is a signal, not noise.** It was the single clue that
  explained why a verified-correct deny rule had no effect — and `ssh -v` turned that
  clue into a precise answer (`Authenticated … using "none"`) in one command. When a
  server's behaviour is inexplicable, ask the client what actually happened.
- **Know when to stop.** Three fixes addressing real data loss and real detectability
  landed cleanly. The fourth was a minor realism tell; documenting the diagnostic trail
  was worth more than continuing to chase it.
