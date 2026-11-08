# SESSION 45 — LibreNMS Live, First Device Added — 2026-11-07

**Status: COMPLETE for this phase.** LibreNMS is installed, validates clean, and is
monitoring its first device: the SG300 switch. 2FA is **deliberately deferred**
until all devices are added (see §6).

**Continuity:** continues `SESSION-44` (syslog, SNMP, host firewall, LibreNMS Parts 1–5).
Everything typed by hand on the t640's console, same as before.

---

## 1. Part 6 — nginx (second layer behind the firewall)

`/etc/nginx/sites-enabled/default` removed. `/etc/nginx/sites-enabled/librenms.conf`:
```
server {
  listen 80;
  server_name 192.168.8.24 t640-mon;
  root /opt/librenms/html;
  index index.php;
  charset utf-8;

  allow 192.168.8.186;
  allow 192.168.8.142;
  deny all;

  location / { try_files $uri $uri/ /index.php?$query_string; }
  location ~ [^/]\.php(/|$) {
    fastcgi_pass unix:/run/php-fpm-librenms.sock;
    fastcgi_split_path_info ^(.+\.php)(/.+)$;
    include fastcgi.conf;
  }
  location ~ /\.(?!well-known).* { deny all; }
}
```
| Part | Why |
|---|---|
| `root /opt/librenms/html` | Only the public folder is served; config and code are not |
| `allow … / deny all` | **Defense in depth:** even if nftables were wrong, nginx answers only Mint (403 for others) |
| `fastcgi_pass unix:…librenms.sock` | Hands PHP to the dedicated pool from SESSION-44 |
| `location ~ /\.` deny | Blocks `.env` (holds the DB password). The honeypot saw `.env` probes **3,728 times** in 15 days |

`sudo nginx -t` → syntax ok, test successful. Browsing to `http://192.168.8.24`
showed LibreNMS **Pre-Install Checks**: PHP 8.4.27 ✅, net-snmp 5.9.4 ✅, fping 5.1 ✅ —
proving the whole chain: Mint → nftables → nginx → PHP socket → LibreNMS.

---

## 2. Part 7 — scheduling

`/opt/librenms/dist/` held more files than expected. Picked **one** mechanism per job:

| File | Used? | Why |
|---|---|---|
| `librenms-scheduler.service` + `.timer` | ✅ | systemd timer, every minute |
| `librenms-scheduler.cron` | ❌ | Cron version of the same job — both = double runs |
| `librenms.cron` → `/etc/cron.d/librenms` | ✅ | 5-minute SNMP polling + daily update |
| `librenms-worker.service` | ❌ | Alternative poller (dispatcher) for large networks |
| `rrdcached/` | ❌ | Graph write cache; not needed for a handful of devices |

Also: `lnms` symlinked to `/usr/bin/lnms` (+ bash completion), and
`misc/librenms.logrotate` → `/etc/logrotate.d/librenms`.
`systemctl list-timers | grep librenms` → last run 12 s ago, next in 47 s. ✅

---

## 3. Part 8 — web installer

- **Database step failed:** `SQLSTATE[HY000] [1045] Access denied for user 'librenms'@'localhost' (using password: YES)`.
  - Isolated it at the source first: `mysql -u librenms -p librenms` on the box.
  - Stored password didn't match → reset with `ALTER USER`, avoiding `'` `"` `\`
    (they break SQL quoting), cleared `/root/.mariadb_history` again, retested, then
    used it in the form. ✅
- Database built (~300 tables), admin user created with a **non-default username**
  (not `admin`/`root`/`librenms`).

---

## 4. Validate — from five problems to clean

| Result | Cause | Fix |
|---|---|---|
| ❌ APP_URL not set correctly | `.env` line was still commented: `#APP_URL=…` | Removed the `#`; `php artisan config:clear` as librenms (Laravel caches config) |
| ⚠️ Install >24 h out of date | `daily.sh` cron installed tonight, never run | Ran `./daily.sh` by hand as librenms — also proves updates (CVE fixes) work through the firewall |
| ⚠️ base_url could be more specific | Not set | `lnms config:set base_url http://192.168.8.24` |
| ❌ Scheduler is not running | Timer fired before the DB existed; Validate remembered the failures | Cleared on its own once installed |
| ❌ Python3 module issue `''` | `python3-setuptools` **missing from the hand-typed package list** (my omission when splitting it into 3 lines). The checker itself crashed on `import pkg_resources`, so it couldn't name anything | See below |

### Python — fixed with apt, not pip
`./scripts/check_requirements.py -v` → `ModuleNotFoundError: No module named 'pkg_resources'`.

| requirements.txt | Debian package | Status |
|---|---|---|
| PyMySQL | python3-pymysql | installed in Part 1 |
| python-dotenv | python3-dotenv | installed in Part 1 |
| redis>=4.0 | python3-redis | installed in Part 1 |
| setuptools | **python3-setuptools** | **added** |
| psutil>=5.9.6 | python3-psutil | installed in Part 1 |
| command_runner>=1.3.0 | **python3-command-runner** (1.7.0-1) | **added** |

- `apt-cache policy <pkg>` showing a `Candidate:` = Debian packages it.
- Chosen over Validate's suggested `pip3 install -r requirements.txt`, which Debian 13
  blocks (`externally-managed-environment`) and which would skip apt security updates.
- Re-run: only a `DeprecationWarning` about `pkg_resources` (LibreNMS's own import
  style, not ours). `echo $?` → exit code `0` = all requirements met.

**Validate: no errors.**

---

## 5. First device — SG300

- First Add Device attempt failed with a credentials error. LibreNMS 26.x keeps SNMP
  communities in a **credentials/secrets store**; the fix was creating the secret
  there and selecting it, rather than typing the string per device.
- Device 1: `192.168.8.165` — System Name `sg300-apt`, OS **Cisco Small Business**,
  Object ID `1.3.6.1.4.1.9.6.1.83.10.1` (same OID seen with `snmpwalk` in SESSION-44).
- Discovery/polling run on schedule; optional immediate run:
  `lnms device:discover 192.168.8.165` then `lnms device:poll 192.168.8.165` (as librenms).
- Graphs need two polls (~10–15 min) before lines appear.

---

## 6. Decision: 2FA deferred

2FA for the LibreNMS admin is postponed until every device is added, to do it as its
own focused step. **Accepted risk until then:** the web UI is reachable only from
Mint (nftables + nginx allow list), LibreNMS is 26.10.0 (past both 2026 CVE fixes),
and the admin username is non-default.

---

## 7. State at end of session

```
Mint (.186/.142) ──HTTP 80──► nftables ──► nginx (allow Mint) ──► php-fpm[librenms] ──► LibreNMS 26.10.0
                                                                                      │
SG300 (.165) ◄──────────── SNMP v2c ro (every 5 min, cron) ───────────────────────────┘
SG300 (.165) ──────────── syslog UDP 514 ──► /var/log/remote/192.168.8.165.log
```

## Follow-ups

- [ ] Add **t640-mon itself** (snmpd, localhost only)
- [ ] Add **pve-node1** (snmpd, answer only 192.168.8.24)
- [ ] Optional: VMs 100/101; Beryl if its firmware supports SNMP
- [ ] t655-fw once its power supply is replaced (`os-net-snmp` plugin)
- [ ] **2FA on the LibreNMS admin** — its own step, right after devices
- [ ] Optional: `snmp-server contact` / `location` on the switch
