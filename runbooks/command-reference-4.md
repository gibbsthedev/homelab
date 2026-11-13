# Command Reference — Part 4

Everything used since Part 3 (the Cowrie / honeypot track). This part covers
**sessions 42–54**: the thin clients (t640-mon, t655-fw), central logging and
SNMP monitoring, LibreNMS, Telegram alerting, the Proxmox maintenance window,
Phase 1 (the lab VLAN behind OPNsense), the first red/blue exercise, Suricata
IDS, and locking down the alert pipeline.

Organized **by topic, not by date**, so it reads as a reference. Every block
says **which machine** it runs on. Secrets are always shown as placeholders
(`<STRING>`, `<TOKEN>`); the real values live in Apple Passwords.

| Machine | Address | Prompt / how you get there |
|---|---|---|
| Mint (admin workstation) | 192.168.8.186 (Wi-Fi) / .142 (wired) | `rich@mintstation` |
| t640-mon (Debian 13, monitoring) | 192.168.8.24 | `ssh t640-mon` → `scgiky@t640-mon` |
| t655-fw (OPNsense) | WAN 192.168.8.195 / LAB 192.168.20.1 | GUI `https://192.168.8.195` from Mint; console option 8 = shell |
| sg300-apt (Cisco SG300-10) | 192.168.8.165 | `ssh sg300` → `sg300-apt#` |
| Beryl AX (GL-MT3000, OpenWrt) | 192.168.8.1 | `ssh root@192.168.8.1` |
| pve-node1 (Proxmox, M720q) | 192.168.10.2 | SSH from Windows PowerShell → `root@pve-node1` |
| hermes (VM 101, RHEL 10) | 192.168.10.206 | `ssh ec2-user@192.168.10.206` from Proxmox |
| testkit (Kali, lab attacker) | 192.168.20.185 on gi3 | local terminal |

---

## 1 — READING A LINUX BOX BEFORE TOUCHING IT

```
ip -4 addr                 # every interface and its IPv4 address
ip link                    # interfaces + state; look inside < > for UP / NO-CARRIER
ip route                   # routing table; "default via X" = the gateway
ip route get 1.1.1.1       # which interface + source IP Linux WOULD use for that target
```
- `ip route get` is the one to remember. When a laptop has **two interfaces up**
  (Wi-Fi + USB Ethernet), Linux picks one route per destination. A test you
  think goes through the lab can silently leave over Wi-Fi. Always check
  `dev` and `src` in its output before trusting a test (SESSION-52).
- In `ip link`: `<…,UP>` missing = interface switched off;
  `NO-CARRIER` = switched on but no cable signal; `LOWER_UP` = link is alive.

```
nmcli device status                  # NetworkManager's view: connected / disconnected / unavailable
nmcli device connect eth0            # bring a wired connection back after a manual disconnect
nmcli device disconnect eth0         # take it off on purpose (e.g. to test over Wi-Fi only)
nmcli radio wifi off                 # Wi-Fi radio off entirely
```
- NetworkManager treats a manual disconnect as "the user wants this off" and
  **won't reconnect on its own** even after you replug the cable.

```
sudo dmesg | tail -20     # kernel messages: new USB devices, drivers, link up/down
lsusb                     # every USB device the system sees (vendor:product IDs)
```
- Identify hardware from **dmesg/lsusb, not the shop listing**. The "RTL8153"
  adapter was really an **ASIX AX88179B** (`0b95:1790`, driver `cdc_ncm`).
- `new high-speed USB device` = USB 2.0 port (≤480 Mbps); `new SuperSpeed` = USB 3.
- On battery, USB power saving can cut power to a USB NIC (no light, vanishes
  from `ip link`). Lab sessions → laptop on the charger.

```
timedatectl                                     # time, timezone, NTP sync, RTC mode
sudo timedatectl set-timezone America/Chicago
sudo timedatectl set-local-rtc 0                # hardware clock in UTC (correct for a Linux-only machine)
```
- **Correlate logs in UTC.** Attacker at −10:00, firewall at −06:00 and a UTC
  switch were the same instant (07:37Z). Never trust a timestamp without its offset.
- `set-local-rtc` only matters on dual-boot (one PC with Windows + Linux).
  Separate computers each have their own clock.

---

## 2 — HOST FIREWALL ON t640-mon (nftables)

```
sudo cp /etc/nftables.conf /etc/nftables.conf.bak-<date>   # always back up first
sudo nano /etc/nftables.conf
sudo nft -c -f /etc/nftables.conf     # -c = CHECK only; silence means valid
sudo systemctl restart nftables       # load it
sudo nft list ruleset                 # what is ACTUALLY loaded (the truth, not the file)
sudo nft list ruleset | grep 514
```
The input chain (default deny):
```
chain input {
  type filter hook input priority 0; policy drop;     # anything not allowed below is dropped
  ct state established,related accept                  # replies to connections WE started (stateful)
  ct state invalid drop
  iif lo accept                                        # the box talking to itself
  ip saddr { 192.168.8.186, 192.168.8.142 } tcp dport { 22, 80 } accept      # Mint → SSH + LibreNMS
  ip saddr { 192.168.8.1, 192.168.8.165, 192.168.8.195 } udp dport 514 accept # syslog senders
  ip protocol icmp accept
  ip6 nexthdr icmpv6 accept
}
```
- `policy drop` = **default deny**. A dropped packet gets no answer ("timed out"),
  so a scanner can't even tell the host exists. `reject` would answer "refused".
- `ct state established,related accept` is the **stateful** line: replies need no
  rule of their own.
- `nft` sorts address sets itself, so `.1` shows first. That's normal.
- **Order: receiver first.** Open the collector (this file) before pointing a
  sender at it, so no logs are dropped in between.

---

## 3 — CENTRAL SYSLOG (rsyslog on t640-mon)

`/etc/rsyslog.d/10-remote.conf` (RainerScript):
```
module(load="imudp")
input(type="imudp" port="514" ruleset="remote")
template(name="RemoteFile" type="string" string="/var/log/remote/%FROMHOST-IP%.log")
ruleset(name="remote") {
  action(type="omfile" dynaFile="RemoteFile"
         fileOwner="root" fileGroup="adm" fileCreateMode="0640"
         dirOwner="root" dirGroup="adm" dirCreateMode="0750")
}
```
| Piece | Meaning |
|---|---|
| `imudp` port 514 | Listen for syslog over UDP |
| `ruleset="remote"` | Network logs go through their own rules, not into the box's own logs |
| `%FROMHOST-IP%` | One file **per sender**: `192.168.8.165.log` = switch, `.195` = firewall, `.1` = Beryl |
| `fileGroup="adm"` + `0640` | Root writes, group `adm` reads, everyone else nothing (SESSION-54) |

```
sudo rsyslogd -N1                    # validate config BEFORE restarting
sudo systemctl restart rsyslog
ls -l /var/log/remote/               # one file per device + rotated .1 / .gz copies
tail -f /var/log/remote/192.168.8.1.log      # watch a device live (Ctrl+C to stop)
```
- **Never restart an unvalidated config.** While rsyslog is down, UDP syslog
  from three devices is simply lost (UDP has no retry).

Rotation — `/etc/logrotate.d/remote`:
```
/var/log/remote/*.log {
    daily
    rotate 14
    compress
    create 0640 root adm
    ...
}
```
```
sudo logrotate -d /etc/logrotate.d/remote     # -d = dry run, changes nothing
```
- `.log` = today, `.log.1` = yesterday, `.log.2.gz` … = older, compressed.
  14 days kept.

Permissions (SESSION-54):
```
sudo chgrp adm /var/log/remote /var/log/remote/*
sudo chmod 750 /var/log/remote
sudo chmod 640 /var/log/remote/*
sudo usermod -aG adm scgiky       # read logs as yourself; takes effect at NEXT login
```
- Access through **group membership**, not open permissions. `adm` is Debian's
  "may read logs" group, the same one `journalctl`'s hint mentions.

### Senders
SG300 (inside `configure`):
```
logging host 192.168.8.24
```
OPNsense: System → Settings → Logging → Remote → target 192.168.8.24:514, all applications.
Beryl (OpenWrt UCI):
```
uci set system.@system[0].log_ip='192.168.8.24'
uci set system.@system[0].log_port='514'
uci set system.@system[0].log_proto='udp'
uci changes system                 # review staged changes
uci commit system
/etc/init.d/log restart
ps | grep [l]ogread                # the forwarder: /sbin/logread -f … -r 192.168.8.24 514 -u
logger "beryl syslog test"         # send a test line
```
- The `[l]` in `grep [l]ogread` stops grep from matching its own command line.
- `logread` alone = OpenWrt's RAM log buffer, which holds history even before
  syslog forwarding exists.

---

## 4 — SEARCHING LOGS (grep / tail patterns)

```
grep 'block' /var/log/remote/192.168.8.195.log | tail -5            # last 5 firewall blocks
grep '192.168.20.185' /var/log/remote/192.168.8.195.log | grep -c block   # COUNT blocks from one source
grep -c 2100498 /var/log/remote/192.168.8.195.log*                   # count per file, incl. rotated
grep 'block' /var/log/remote/192.168.8.195.log | grep ',68,67,' | tail -3 # DHCP requests blocked on WAN
tail -n 200 /var/log/remote/192.168.8.195.log | grep -c 'suricata.*Priority: [12]'   # test a filter
```
- `grep -c` counts instead of printing. Doing the math on counts tells you about
  the attacker: **2048 = 256 hosts × 4 probes × 2 tries** = one `nmap -sn` sweep.
- `'Priority: [12]'` — square brackets = "one of these characters" (1 or 2).
- A `*` at the end of a filename matches the rotated copies too.
- **Gotcha:** a typo in the search term (`21004988`) returns nothing, which looks
  exactly like "no data". And `grep` with no search term waits for keyboard input
  (looks hung; Ctrl+C).

Reading an OPNsense `filterlog` line (CSV):
```
…filterlog[…]: 59,,,<rule-id>,re0,match,block,in,4,0x0,,255,…,17,udp,328,0.0.0.0,255.255.255.255,68,67,308
```
| Field | Meaning |
|---|---|
| `re0` | interface (WAN) |
| `block,in` | action, direction |
| `4` | IPv4 |
| `255` (the TTL) | **fingerprint**: 255 = Apple devices, 64 = most Linux, 128 = Windows |
| `17,udp` | protocol |
| `0.0.0.0 → 255.255.255.255` port `68 → 67` | a DHCP request ("anyone, give me an address") |

---

## 5 — SNMP (monitoring)

The pattern used on every host: **one address, one source, read-only, a unique
community per device** (`openssl rand -hex 16`), tested before adding to LibreNMS.

### Debian / Proxmox / RHEL (`/etc/snmp/snmpd.conf`)
```
sudo apt install snmpd                     # Debian / Proxmox
sudo dnf install net-snmp net-snmp-utils   # RHEL (Hermes)
sudo cp /etc/snmp/snmpd.conf /etc/snmp/snmpd.conf.bak   # back up, then replace the whole file:
```
```
agentaddress udp:<this-host-ip>:161
rocommunity <UNIQUE_STRING> 192.168.8.24
sysLocation "Apartment media shelf"
sysContact Rich
```
```
sudo systemctl enable --now snmpd
ss -ulnp | grep 161          # UNCONN is normal for UDP (connectionless) = listening
```
- Distro defaults ship a `public` community. Replace the file, don't patch it.
- `snmp.conf` (client) ≠ `snmpd.conf` (agent). A typo creates an empty wrong file.

Host firewalls had to allow it:
```
sudo ufw allow from 192.168.8.24 to any port 161 proto udp comment 'SNMP from t640-mon'   # Debian VM (UFW)
sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="192.168.8.24" port port="161" protocol="udp" accept'
sudo firewall-cmd --reload                                                                  # RHEL (firewalld)
sudo ausearch -m avc -ts recent       # RHEL: SELinux denials, if it still fails
```

### SG300 (inside `configure`)
```
snmp-server server
snmp-server community <STRING> ro 192.168.8.24 view Default
snmp-server location "Apartment media shelf"      # quotes, or the spaces split the value
no snmp-server community <OLD-STRING>             # remove an old one
```
- `show running-config` **prints the community in clear text**. Don't photograph it.

### OpenWrt / Beryl (UCI)
```
opkg update && opkg install snmpd
/etc/init.d/snmpd stop                 # close the window where 'public' is open
cat /etc/config/snmpd                  # read the stock config first
uci set snmpd.@agent[0].agentaddress='UDP:192.168.8.1:161'
read -s C                              # type the community; nothing echoes
uci set snmpd.public.community="$C"
uci set snmpd.public.source='192.168.8.24'
uci delete snmpd.private               # + private_*, public6, private6, public_v1, public_usm
uci set snmpd.@system[0].sysName='beryl-ax'
uci changes snmpd                      # review (don't photograph: shows the community)
uci commit snmpd && /etc/init.d/snmpd enable && /etc/init.d/snmpd start
```
- Stock OpenWrt snmpd: `public` from anywhere, a **read-write** `private`
  community, IPv6 copies of both, listening on WAN. Stop it right after install.
- `uci` **stages** changes until `commit`, same idea as Apply in OPNsense.

### OPNsense
Plugin `os-net-snmp` → Services → Net-SNMP. It has **no allowed-source field**:
the **firewall rule** (WAN, UDP 161, source 192.168.8.24) does the restricting.

### Testing (from t640-mon)
```
snmpget -v2c -c <STRING> 192.168.8.165 1.3.6.1.2.1.1.5.0   # sysName → STRING: "sg300-apt"
snmpget -v2c -c public 192.168.8.1 1.3.6.1.2.1.1.5.0       # must TIME OUT = old door shut
snmpwalk -v2c -c <STRING> 192.168.8.165 1.3.6.1.2.1.1      # the whole "system" branch
```
- Debian ships without MIB names (non-free), so use numeric OIDs.
  `1.3.6.1.2.1.1.5.0` = sysName. A sysObjectID starting `1.3.6.1.4.1.9` = Cisco.
- **Test the negative too**: proving `public` times out matters as much as proving
  your string works.

---

## 6 — LIBRENMS (t640-mon)

```
cd /opt/librenms && git describe --tags --abbrev=0   # version (26.10.0) — check against CVE fix versions
sudo -u librenms ./validate.php                      # health check
sudo -u librenms lnms config:set base_url http://192.168.8.24
sudo -u librenms php artisan config:clear            # Laravel caches config — clear after .env edits
sudo -u librenms ./daily.sh                          # updates; also proves updates work through the firewall
./scripts/check_requirements.py -v ; echo $?         # 0 = all Python requirements met
apt-cache policy python3-setuptools                  # "Candidate:" = Debian packages it → use apt, not pip
systemctl list-timers | grep librenms                # the scheduler timer is firing
sudo -u librenms lnms device:discover 192.168.8.165  # run discovery now
sudo -u librenms lnms device:poll 192.168.8.165      # run a poll now
```
| Job | Default | Collects |
|---|---|---|
| Polling | every 5 min | numbers → graphs |
| Discovery | every 6 h | what a device *is*: name, location, ports (use **Rediscover** after changing them) |

- Python deps from **apt**, never `pip install` (Debian blocks it, and pip skips
  security updates).
- Add devices by **real IP**; `127.0.0.1` collides with every other Linux host.
- Expected-down ports → **Ignore** (polled, never alerts), not Disable.
- **26.x alerting chain: Rule → Operation → Transport.** A rule without an
  operation raises the alert and sends **nothing**. Transport "Test" only proves
  delivery; pull a cable to prove alerting.

nginx (`/etc/nginx/sites-enabled/librenms.conf`) — second layer behind nftables:
```
root /opt/librenms/html;                  # only the public folder is served
allow 192.168.8.186; allow 192.168.8.142; deny all;
fastcgi_pass unix:/run/php-fpm-librenms.sock;
location ~ /\.(?!well-known).* { deny all; }   # blocks .env (DB password)
```
```
sudo nginx -t        # syntax test before reload
```
MariaDB:
```
mysql -u librenms -p librenms        # test the DB login at the source
rm ~/.mariadb_history /root/.mariadb_history   # typed SQL (incl. passwords) is kept here
```

---

## 7 — TELEGRAM WITHOUT LEAKING THE TOKEN

```
read -s TOKEN                                                     # silent prompt
curl -s "https://api.telegram.org/bot${TOKEN}/getUpdates"         # find your chat id ("chat":{"id":…)
unset TOKEN
```
| Method | In shell history? | Repeat with ↑? |
|---|---|---|
| Token typed in the command | yes ❌ | ✅ |
| Leading space before the command | no ✅ | no — retype (→ typos → `401`) |
| **`read -s` + `${VAR}`** | **no ✅** | **✅** |

```
history | tail -5 ; history -d <n>     # delete a history line that leaked a secret
```
- `401 Unauthorized` = wrong token. `chat not found` = wrong chat id.
- A bot only messages you after you've pressed **Start** in its chat.
- One bot per job. Revoke exposed tokens in BotFather (`/mybots` → API Token → Revoke).

---

## 8 — THE ALERT WATCHER (`ids-telegram.service` on t640-mon)

`/etc/ids-telegram.env` — secrets, `root:idsalert 0640`, `KEY=value` with **no spaces around `=`**:
```
TOKEN=<TOKEN>
CHAT=<CHAT_ID>
```
`/usr/local/bin/ids-telegram.sh`:
```bash
#!/bin/bash
. /etc/ids-telegram.env                       # "." = source: load the variables
LOG=/var/log/remote/192.168.8.195.log
last=0
tail -F -n0 "$LOG" | grep --line-buffered 'suricata.*Priority: [12]' | while read -r line; do
  now=$(date +%s)                             # seconds since 1970
  [ $((now - last)) -lt 60 ] && continue      # rate limit: max 1 message / 60 s
  last=$now
  msg=$(echo "$line" | sed 's/.*suricata\[[0-9]*\]: //' | cut -c1-300)
  curl -s -m 10 "https://api.telegram.org/bot${TOKEN}/sendMessage" \
    --data-urlencode "chat_id=${CHAT}" \
    --data-urlencode "text=IDS alert: ${msg}" >> /var/log/ids-telegram.log 2>&1
done
```
| Piece | Why |
|---|---|
| `tail -F -n0` | Only **new** lines; `-F` keeps following after log rotation |
| `--line-buffered` | grep passes each line on immediately instead of waiting to fill a buffer |
| `read -r line` | One log line at a time; `-r` keeps backslashes literal |
| `sed 's/.*suricata\[[0-9]*\]: //'` | Strip everything up to `suricata[pid]: ` |
| `cut -c1-300` | Cap the message length |
| `--data-urlencode` | Sends attacker-controlled text **safely as data** (never executed) |
| `>> …log 2>&1` | Keep Telegram's reply. That's how `text_id=` vs `text=` was found |

Service `/etc/systemd/system/ids-telegram.service` — see SESSION-54 for the full
sandbox. Key lines: `User=idsalert`, `NoNewPrivileges=yes`, `ProtectSystem=strict`,
`ReadWritePaths=/var/log/ids-telegram.log`, `CapabilityBoundingSet=` (empty),
`SystemCallFilter=@system-service`.

```
sudo useradd --system --no-create-home --shell /usr/sbin/nologin --groups adm idsalert
id idsalert
sudo systemctl daemon-reload                     # after ANY edit to a .service file
sudo systemctl enable --now ids-telegram         # start now + at every boot
sudo systemctl restart ids-telegram              # after editing the env file or script
systemctl status ids-telegram --no-pager         # state + the process tree (CGroup)
journalctl -u ids-telegram -n 20                 # its log (needs group adm, or sudo)
ps -o user=,cmd= -u idsalert                     # proof it runs as idsalert, not root
systemd-analyze security ids-telegram            # exposure score: 9.6 → 8.5 → 1.7
```
- **A service reads its config at start.** Edit the env file → restart, or it keeps the old values.
- **Bash never complains about a wrong name**; an empty variable just becomes "".
  The API is what catches it ("message text is empty").
- Debug a pipeline one layer at a time: process → log line → filter → delivery → exact request.

Manual delivery test (root shell, then **exit**):
```
sudo -i
. /etc/ids-telegram.env
curl -s https://api.telegram.org/bot$TOKEN/sendMessage -d chat_id=$CHAT -d text=test
exit
```

---

## 9 — CISCO SG300 (CLI)

```
show vlan                                  # VLANs and their ports
show interfaces switchport gi5             # mode (Access/Trunk), native VLAN, tagged/untagged members
show interfaces status gigabitethernet3    # Up / Down
show ip interface                          # the switch's own addresses (SVIs = VLAN gateways)
show access-lists                          # ACL contents
show running-config                        # everything (space = next page, q = quit) — prints secrets!
copy running-config startup-config         # SAVE. Without it a reboot reverts everything
```
```
configure
interface gi5
switchport trunk allowed vlan add 20       # carry VLAN 20 tagged on the firewall's trunk
exit
interface vlan 20
no ip address 192.168.20.1                 # hand-off: switch stops routing VLAN 20
exit
interface gi3
no service-acl input                       # unbind the ACL (re-bind: service-acl input vlan20-policy)
exit
username <newname> password <PW> privilege 15   # new admin (test it BEFORE: no username cisco)
```
Firmware 1.1.2.0 gotchas:
- **No `| include`** output filter → page through instead.
- `no ip address` wants the address **without** the mask.
- `% Unrecognized command` = typed at `#` instead of `(config)#`.
- `% Wrong number of parameters` = unquoted value with spaces.
- `%LINK-I-Up: Vlan 20` appears the moment any member port has link. A VLAN
  interface is only up when one of its ports is.
- Keep the config **unsaved** while testing a risky change: a reboot = free rollback.

**ACL lesson:** a switch ACL drops traffic **without logging**. With it bound to
gi3, a whole nmap sweep produced zero firewall log lines (attacker blind, defender blind).

---

## 10 — OPNSENSE (t655-fw)

Console (option 8 = shell):
```
sockstat -4l | grep ':67 '        # is anything listening for DHCP? (nothing = good on a WAN-only box)
ifconfig re0 | grep "inet "       # WAN address
ifconfig re0 | grep media         # link speed/duplex (1000baseT full-duplex after os-realtek-re)
```
GUI facts that cost time:
- **Nothing changes until Apply.** Rules, interfaces, DHCP, Suricata settings.
- **Read the action icon, not the description.** "Internet only" was saved as Block (red ✕).
- Anti-lockout: Firewall → Settings → Advanced → Miscellaneous (attaches to WAN if no LAN exists).
- Dnsmasq: **no interface selected = listens on ALL interfaces** (its own help text says so).
  Always pick the interface. That was the October rogue-DHCP incident.
- Confirm updates by the **version number** afterwards, not by having clicked Update.
- Firewall → Log Files → **Live View** = first stop when "it doesn't work".
- `config.xml` backups never go in the repo (password hash, SNMP community).

Phase 1 values (why each one: see SESSION-51 §3):
| Where | Value |
|---|---|
| Interfaces → Devices → VLAN | parent `re0`, tag `20` → `vlan01` |
| Interfaces → Assignments (red +) | `opt1` "LAB", Static `192.168.20.1/24`, **no upstream gateway** |
| Dnsmasq | Interface LAB only; range `.100–.199`; listen port left at 53053 (Unbound keeps 53) |
| Alias `PRIVATE_NETS` | `10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16` |
| LAB rules (first match wins) | 1 pass DNS → LAB addr · 2 pass ICMP → LAB addr · 3 block → This Firewall · 4 block → PRIVATE_NETS · 5 pass → any |
| NAT → Outbound | Automatic: LAB net → WAN address |

Reading NAT in a packet capture (`tcpdump -n -e -ttt -i pflog0` on the console):
```
pass in  on vlan01: 192.168.20.185.45588 > 108.156.211.27.443  ttl 64
pass out on re0:    192.168.8.195.3307   > 108.156.211.27.443  ttl 63
```
Same packet (same IP id): source rewritten (**NAT**), TTL −1 (**one routing hop**).

---

## 11 — SURICATA IDS (OPNsense) AND HOW TO TEST IT

Settings: PCAP live mode (IDS = alert, never drop), interface **LAB**, syslog alerts
+ EVE syslog on. Rulesets: ET open emerging-scan, -attack_response, -policy.

Trigger a known signature yourself (testkit):
```
curl -d 'uid=0(root) gid=0(root)' http://example.com
```
- Rule **SID 2100498** "GPL ATTACK_RESPONSE id check returned root" matches that
  text in any direction. Must be **http**: Suricata can't read inside TLS.
- testmynids.org (the classic test) **expired**. An empty alert list proves nothing
  until the test itself is proven.

Reading an alert:
```
suricata[…]: [1:2100498:7] GPL ATTACK_RESPONSE id check returned root
  [Classification: Potentially Bad Traffic] [Priority: 2] {TCP} 192.168.20.185:58802 -> 172.66.147.243:80
```
`generator:SID:revision` · priority 1 high / 2 medium / 3 low · protocol, source → destination.
One connection can alert twice (packet + reassembled stream).

Warnings worth reading: "syslog output deprecated, removed in Suricata 9.0"
(use EVE); "flowbit … checked but not set" (harmless dependency on a disabled rule).

---

## 12 — ATTACKER SIDE (testkit, lab only)

```
sudo nmap -sn 192.168.8.0/24                         # ping sweep: which hosts are alive?
nmap -p 80 --script http-title scanme.nmap.org       # scanme is the nmap project's legal target
ping -c3 -W2 192.168.8.186                           # -W2 = give up after 2 s per reply
curl -k -m5 https://192.168.20.1                     # -k ignore cert, -m5 max 5 s (needs the dash!)
dig example.com                                      # answer + "SERVER:" = which DNS answered
```
- `-sn` sends 4 probes per host (ICMP echo, TCP SYN 443, TCP ACK 80, ICMP
  timestamp) and retries once if nothing answers.
- Only scan what you own or what explicitly permits it.

The result that justifies segmentation:
| Attacker standing in | Sees | Firewall sees |
|---|---|---|
| Lab (VLAN 20) | 0 hosts | every probe |
| Home Wi-Fi (VLAN 1) | everything | nothing |

---

## 13 — PROXMOX (pve-node1)

Maintenance:
```
apt update && apt full-upgrade        # full-upgrade on Proxmox, never plain upgrade
uname -r ; pveversion                 # running kernel / version (kernels load only at reboot)
qm config 101                         # a VM's settings (disks, ide2 ISO, memory, onboot)
qm shutdown 100                       # clean shutdown (guest OS stops itself)
qm set 100 --onboot 0                 # don't start with the host
qm set 101 --memory 12288             # MB; applies at next full stop/start
qm set 101 --ide2 none,media=cdrom    # detach an ISO BEFORE deleting it, or the VM won't boot
qm status 101
```
Storage:
```
pvesm status                          # storage pools and usage
lvs                                   # thin volumes; Data% = what's really used; snapshots appear here too
df -h /                               # root filesystem usage
```
- A thin volume **promises** its size and only uses written blocks. A snapshot of
  a stopped VM stays nearly free, but it lives on the same disk: **not a backup**.

Growing a VM disk (outside in):
```
qm disk resize 101 scsi0 +20G          # Proxmox (host)
growpart <disk> <partition-number>     # in the VM: move the partition edge
pvresize <partition>                   # LVM physical volume
lvextend -r -l +100%FREE <root-LV>     # logical volume; -r grows the filesystem too
```
- Skip a layer and the space is invisible above it. **XFS grows, never shrinks.**

Backups:
```
cat /etc/pve/jobs.cfg                 # the backup job: vmid, schedule, prune-backups keep-last
ls -lh /var/lib/vz/dump/              # real backups = vzdump-qemu-<id>-<date>.vma.zst
cat /root/backup-to-r2.sh             # offsite: newest .vma.zst of ANY VM → R2, then delete others
ls /var/lib/vz/dump/*2026_11_09*      # PREVIEW a wildcard with ls …
rm -i /var/lib/vz/dump/*2026_11_09*   # … then rm -i (asks per file)
```
```
scp root@192.168.10.2:/var/lib/vz/dump/<file>.vma.zst $HOME\Downloads\    # PowerShell: copy off the box
```
- Disk math: dump size × (keep-last + 1), because vzdump writes the new dump
  **before** pruning. 16.8 GB × 4 would have filled the 68 GB root.
- A task log (`task-…qmsnapshot….txt`, 2 KB) is a receipt, not a backup.
- `keep-last` only prunes VMs that are **in** the job.

Boot order:
```
efibootmgr                                      # BootCurrent = what actually booted; BootOrder = the list
efibootmgr -o 0002,000C,0001,0008,0009,0006,0007   # proxmox first, PXE last
mokutil --sb-state                              # Secure Boot on/off
ls -l /boot/efi/EFI/proxmox /boot/efi/EFI/BOOT  # compare the real and fallback boot files
```
- Read **BootCurrent**, not just BootOrder: shim's fallback (`fbx64.efi`)
  re-creates the entry after a fallback boot and hides the evidence.
- Lenovo sorts by **device group** in the BIOS (F1 → Startup). Excluding the
  Network group there was the real fix.

---

## 14 — HERMES (RHEL 10, VM 101)

```
sudo fstrim -av                                   # release deleted blocks to the thin pool
systemctl is-enabled fstrim.timer                 # RHEL ships it disabled
sudo systemctl enable --now fstrim.timer          # weekly TRIM from now on
sudo dnf install nano                             # RHEL has only vi by default
```
- First `fstrim` after boot reports **all** free space, not what was "freed".
  Proof is `lvs` Data% on the host and the next backup's size.
- On a VM, deleting files doesn't shrink the backup until TRIM runs.

---

## 15 — SERVICES & PACKAGES (any Debian box)

```
sudo apt install curl                        # minimal installs lack basics
systemctl list-timers                        # scheduled jobs (systemd timers)
sudo nano /etc/NetworkManager/conf.d/wifi-powersave.conf   # [connection] wifi.powersave = 2 (off)
iw dev wlan0 get power_save
```
needrestart dialogs after `apt`:
- **Pending kernel upgrade** → only a reboot loads it. Plan it.
- **Daemons using outdated libraries** → defaults are fine; restarting `ssh` doesn't drop your session.

---

## 16 — HABITS THAT RAN THROUGH ALL OF IT

- Back up the file → edit → **validate** (`nft -c`, `rsyslogd -N1`, `nginx -t`,
  `sshd -t`, `logrotate -d`) → apply → verify the **live** state.
- Preview destructive wildcards with `ls` before `rm`; use `rm -i`.
- Create the new thing, **test it in a second session**, then remove the old one
  (admin users, SSH keys, IP hand-offs, log formats).
- Verify by function, not by what a screen says.
- Exit root shells when done.
- Never photograph secrets: `show running-config`, `uci changes`, QR codes, tokens.
