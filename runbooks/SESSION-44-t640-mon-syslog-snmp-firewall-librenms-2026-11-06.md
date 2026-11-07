# SESSION 44 — t640-mon: Syslog, SNMP, Host Firewall, LibreNMS (Parts 1–5) — 2026-11-03 → 2026-11-06

**Status: PARTIAL — stopped at a clean checkpoint.** `t640-mon` now has its first
real jobs: it receives the switch's logs and can poll the switch over SNMP, behind a
default-deny host firewall. LibreNMS is installed through its database and PHP pool;
the web server, scheduler and web installer are next.

**Continuity:** follows `SESSION-42` (thin clients built) and `SESSION-43` (honeypot
final pull and decommission). Everything here was **typed by hand on the t640's
console** — a deliberate choice for learning.

---

## 0. Hardware and power

- Everything moved onto a **surge protector**: Beryl, SG300, M720q, both thin clients.
- **t655-fw is offline: its power supply failed.** All the usual swaps were tried
  (outlet, AC cord, brick). OPNsense isn't routing anything yet, so nothing depends
  on it. Replace with a brick matching the label (HP, 19.5 V, same watts and tip).
- `t640-mon` came back on its reserved `192.168.8.24`, reachable wired and over Wi-Fi.

---

## 1. Central syslog — switch → t640-mon — DONE

**Why:** one place to read every device's events, with correct timestamps now that
SNTP is fixed (SESSION-42). This is the same job a SIEM or CloudWatch Logs does.

### Receiver (t640-mon)
- `rsyslog`, listening on **UDP 514** (syslog's standard port).
- `/etc/rsyslog.d/10-remote.conf` writes **one file per sender IP** to
  `/var/log/remote/<ip>.log`, using a dedicated ruleset so remote logs never mix
  into the box's own logs.
- Check: `sudo ss -ulnp | grep 514`.

### Rotation, from day one
`/etc/logrotate.d/remote`: daily, keep 14, compress (`delaycompress`), skip empty,
signal rsyslog after rotating. Dry run with `sudo logrotate -d /etc/logrotate.d/remote`.
The dry run said "does not exist -- skipping" only because no remote log existed yet.

### Sender (SG300)
```
configure
logging host 192.168.8.24
exit
copy running-config startup-config
```

### Proof
`sudo tail -f /var/log/remote/192.168.8.165.log`, then unplug and replug Mint on gi4:
```
20:59:29 %SYSLOG-N-NEWSYSLOGSERVER: configure new syslog server t640-mon...
21:00:01 %COPY-N-TRAP: The copy operation was completed successfully
21:01:59 %LINK-W-Down: gi4
21:02:00 %LINK-I-Up: gi4
21:02:05 %STP-W-PORTSTATUS: gi4: STP status Forwarding
```
The switch logged its own config change first. Forwarding 5 s after link-up again
confirms **Rapid STP** (SESSION-42).

---

## 2. SNMP — t640-mon polls the switch — DONE

**Why:** syslog tells you about *events*; SNMP lets the monitoring box *ask* for
numbers (per-port traffic, errors, uptime) on a schedule. LibreNMS needs it.

### Switch
```
configure
snmp-server server
snmp-server community <STRING> ro 192.168.8.24 view Default
exit
copy running-config startup-config
```
- `ro` = read-only. The IP = **only t640-mon may query**.
- **Display glitch:** the community line showed on screen as `$community …` after a
  dropped SSH session. It *was* accepted — proven by the walk below. Verify by
  function, not by what the terminal shows.

### Debian gotcha: names vs. numbers
```
snmpwalk -v2c -c <STRING> 192.168.8.165 system
system: Unknown Object Identifier (Sub-id not found: (top) -> system)
```
Debian doesn't ship the MIB files that map names to OIDs (non-free). Use the number:
```
snmpwalk -v2c -c <STRING> 192.168.8.165 1.3.6.1.2.1.1
```

| OID | Returned | Meaning |
|---|---|---|
| `.1.1.0` sysDescr | 10-port Gigabit Managed Switch | What it is |
| `.1.2.0` sysObjectID | `1.3.6.1.4.1.9.…` | `.9` = Cisco's enterprise number |
| `.1.3.0` sysUpTime | 3 days, 3:03 | Last reboot ≈ Nov 3 (power-strip move) |
| `.1.5.0` sysName | sg300-apt | Hostname |
| `.1.4.0` / `.1.6.0` | empty | Contact / location not set (optional) |

### Secret hygiene
- The first community string was visible in screenshots and contained personal
  details. **Replaced** with a random one; old one removed (`no snmp-server community …`).
- Stored in Apple Passwords only. **Never written in this repo.**

---

## 3. Host firewall on t640-mon (nftables) — DONE

**Why:** LibreNMS had serious CVEs this year, so the web app must only be reachable
from the admin workstation — before it is even installed.

| CVE | Type | Auth needed | Fixed in |
|---|---|---|---|
| CVE-2026-26988 | SQL injection (`ajax_table.php`), CVSS 9.1, public PoC | **No** | 26.2.0 |
| CVE-2026-6204 | OS command injection → RCE, CVSS 8.5, public PoC | Admin | 26.3.0 |

### `/etc/nftables.conf` (typed in nano)
```
table inet filter {
  chain input {
    type filter hook input priority 0; policy drop;
    ct state established,related accept
    ct state invalid drop
    iif lo accept
    ip saddr { 192.168.8.186, 192.168.8.142 } tcp dport { 22, 80 } accept
    ip saddr 192.168.8.165 udp dport 514 accept
    ip protocol icmp accept
    ip6 nexthdr icmpv6 accept
  }
  chain forward { type filter hook forward priority 0; policy drop; }
  chain output  { type filter hook output  priority 0; policy accept; }
}
```
- `policy drop` = **default deny**.
- `ct state established,related accept` = **stateful**: replies to connections this
  box started (SNMP to the switch, apt, NTP) come back without their own rule. The
  same idea as OPNsense vs. the SG300's stateless ACLs.
- Syntax check before loading: `sudo nft -c -f /etc/nftables.conf` (no output = OK).
- Enabled at boot: `sudo systemctl enable --now nftables`.

### Verified by function
| Test | Result |
|---|---|
| `ssh t640-mon` from Mint | ✅ allowed |
| Switch syslog after the firewall went up (21:47) | ✅ still arriving |
| `ssh` from the **Windows laptop** | ❌ `Connection timed out` |

**Timed out, not refused:** `drop` discards silently, so an outsider can't even tell
the host exists. `reject` would answer "refused."

Recovery if SSH ever breaks: work at the t640's own console (not filtered) and
`sudo systemctl stop nftables`.

---

## 4. LibreNMS — Parts 1–5 of 8

### Part 1 — packages
Split into three `apt install` lines to keep typos findable: MariaDB + nginx; PHP 8.4
modules (cli, fpm, curl, gd, gmp, mbstring, mysql, snmp, xml, zip); tools (acl, fping,
git, graphviz, imagemagick, mtr-tiny, nmap, rrdtool, unzip, whois, python3 deps).
**PHP 8.4.27.**

### Part 2 — service account and code
```
sudo useradd librenms -d /opt/librenms -M -r -s /bin/bash
sudo git clone https://github.com/librenms/librenms.git /opt/librenms
sudo git describe --tags --abbrev=0      # 26.10.0 — past both CVE fixes
sudo chown -R librenms:librenms /opt/librenms
sudo chmod 771 /opt/librenms
sudo setfacl -d -m g::rwx …/rrd …/logs …/bootstrap/cache …/storage
sudo setfacl -R -m g::rwx …(same four)
sudo su - librenms ; ./scripts/composer_wrapper.php install --no-dev ; exit
```
- `-r` system account, no password — **the app never runs as root** (least privilege).
- ACLs here are file ACLs: `-d` default for new files, `-R` existing ones.
- Proof composer finished: `ls /opt/librenms/vendor` lists folders.

### Part 3 — timezone
`date.timezone = America/Chicago` in **both** `/etc/php/8.4/fpm/php.ini` (web) and
`/etc/php/8.4/cli/php.ini` (commands / scheduled jobs), edited in nano with Ctrl+W.
`timedatectl`: America/Chicago (CST), clock synchronized, NTP active.

### Part 4 — database
- `/etc/mysql/mariadb.conf.d/60-librenms.cnf`: `innodb_file_per_table=1`,
  `lower_case_table_names=0`.
- `CREATE DATABASE librenms CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;`
- `CREATE USER 'librenms'@'localhost' …` — local connections only.
- `GRANT ALL PRIVILEGES ON librenms.* …` — this one database only.
- Typo `PRIVELEGES` → `ERROR 1064` (SQL syntax). Read the error, retyped. Fine.
- `ss -tlnp | grep 3306` → **`127.0.0.1:3306`**: not network-reachable.

**Secret hygiene:** the first DB password was visible in a screenshot **and was the
same string as the SNMP community** (password reuse). Changed with `ALTER USER`, and
the MariaDB client history (which records typed passwords) was deleted:
`sudo rm -f /root/.mariadb_history /root/.mysql_history`.

### Part 5 — dedicated PHP-FPM pool
`/etc/php/8.4/fpm/pool.d/librenms.conf` (copied from `www.conf`):
| Setting | Value | Why |
|---|---|---|
| `[librenms]` | pool name | separate from the default `www` pool |
| `user` / `group` | `librenms` | PHP code runs with LibreNMS's rights only |
| `listen` | `/run/php-fpm-librenms.sock` | must differ from `www`'s socket or FPM won't start |
| `listen.owner` / `group` | `www-data` (left as-is) | nginx runs as www-data and must reach the socket |

Result: `srw-rw---- www-data www-data /run/php-fpm-librenms.sock` — a socket, not
network-reachable, readable only by nginx.

---

## 5. State at end of session

```
SG300 (.165) ──syslog UDP 514──► t640-mon (.24)  /var/log/remote/192.168.8.165.log
SG300 (.165) ◄──SNMP v2c ro───── t640-mon (.24)  (replies via ct established)
Mint (.186/.142) ──SSH 22 / HTTP 80──► t640-mon  (only allowed sources)
everything else ──────────X────────► t640-mon  (policy drop, verified from Windows)
```

## Follow-ups

- [ ] **Part 6:** nginx site for `/opt/librenms/html`, `fastcgi_pass` to the librenms
      socket, **`allow` Mint / `deny all`** (second layer behind nftables), `nginx -t`
- [ ] **Part 7:** `lnms` symlink, scheduler timer, cron, logrotate
- [ ] **Part 8:** web installer at `http://192.168.8.24/install`, admin account,
      **enable 2FA**, run Validate, add SG300 (v2c, new community)
- [ ] Optional: `snmp-server contact` / `location` on the switch
- [ ] Replace t655-fw power supply; then add it as a syslog + SNMP source
- [ ] Keep LibreNMS auto-updates on (future CVE fixes arrive that way)
