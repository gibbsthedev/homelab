# SESSION 50 — Backups Audited, Boot Order Fixed for Good, Alerts on My Phone, Beryl Monitored + Logging — 2026-11-11

**Status: COMPLETE.** One day, six results: the backup chain was
checked end to end (and two gaps found), the PXE boot problem from September was
fixed in the BIOS, LibreNMS alerts now reach my phone through Telegram, and the
Beryl, the internet gateway, became monitored device 7.

**Continuity:** follows `SESSION-49` (same morning). Closes its BIOS follow-up.

---

## 1. Backup audit — two gaps found

### 1a. The nightly job only ever backed up OpenClaw
`cat /etc/pve/jobs.cfg` → the job now reads `vmid 101` (Hermes only; VM 100 excluded ✅).
But `ls -lh /var/lib/vz/dump/` showed **only `vzdump-qemu-100-…` files**, no 101 at all.

**Finding:** until the parking, the nightly job covered **OpenClaw only**. **Hermes, the VM
that runs the public sites, had no nightly backup in this folder.** Its first runs tonight at 02:00.

### 1b. The "Windows copy" was a receipt, not a backup
The file saved to Windows was `task-pve-node1-qmsnapshot-….txt`, **2 KB**: the **log**
of the snapshot task. The snapshot itself lives on the NVMe, and so does the OpenClaw
backup on Hermes. **Every copy of OpenClaw was on one disk.**

Fix: copy the final vzdump off the box (PowerShell on Windows):
```
scp root@192.168.10.2:/var/lib/vz/dump/vzdump-qemu-100-2026_11_11-02_00_04.vma.zst $HOME\Downloads\
```
A real backup = a `.vma.zst` file of the right size (9.8 GB), stored on another machine.
**Done:** copy finished (about 3 minutes over the LAN), before the 03:30 R2 swap.

### 1c. What the R2 script actually does (`/root/backup-to-r2.sh`)
```
LATEST=$(ls -t /var/lib/vz/dump/*.vma.zst | head -1)   # newest dump, ANY VM
rclone copy "$LATEST" r2:homelab-backups/
# only if the upload landed: delete everything else in the bucket
rclone delete r2:homelab-backups/ --exclude "$BASENAME"
```
No secrets in the script (keys live in rclone's config).

| When | Effect |
|---|---|
| 11-11 03:30 (already ran) | R2 = OpenClaw's final backup |
| **11-12 02:00** | Hermes's first dump becomes the newest file |
| **11-12 03:30** | R2 = Hermes ✅ — **OpenClaw's R2 copy deleted** |

From tonight on, the **Windows copy is OpenClaw's only off-NVMe backup.**
Watch: Hermes's compressed dump may be close to R2's 10 GB free tier.

### 1d. Cleanup
`keep-last=3` only prunes VMs **in** the job, so VM 100's dumps would never be pruned.
Previewed with `ls`, then deleted the 11-09 and 11-10 sets with `rm -i` (one confirmation per file).
Kept 11-11 (final). **pve-root: 72% → 56% (ISO removed) → 27%** (18 GB used, 40 GB free).

---

## 2. Boot order — fixed in the BIOS, verified

The BIOS's own **Primary Boot Sequence** showed the real cause:
```
Network 1:  UEFI PXE IPv4 / IPv6     ← first
USB CDROM / USB HDD
M.2 Drive 1: SAMSUNG …  (proxmox, UEFI OS)
```
Lenovo firmware sorts by **device group**. `efibootmgr` (SESSION-49) reordered the
individual entries, but the firmware still tried the **Network group** first.

Fix (F1 → Startup → Primary Boot Sequence):
- **Network 1 → Excluded** (`x`); Networks 2–4 were already excluded
- **M.2 Drive 1 → top** (`+`), above USB. A stray USB stick can't hijack a boot;
  F12 still boots USB on purpose
- Boot Order Lock off; F10 save

**Verified:** watched reboot went straight to Proxmox with no key presses. Open since 2026-09-20 — **closed.**

---

## 3. LibreNMS → Telegram

### Bot
- **New bot just for alerts.** Not OpenClaw's (@ScargiaBot), not Hermes's, not the old
  honeypot bot. One bot per job means a leak only affects one thing.
- **Old honeypot bot token revoked** in BotFather (exposed since September).
- Token + chat ID in Apple Passwords; never in screenshots or this repo.

### Getting the chat ID without leaking the token
- `curl` not installed on t640-mon (minimal Debian) → `apt install curl`.
- First try: leading space keeps the line out of history, but then **↑ can't recall it**,
  so the retyped token had a typo → `401 Unauthorized`.
- **Better pattern:**
  ```
  read -s TOKEN                     # silent prompt, nothing on screen
  curl -s "https://api.telegram.org/bot${TOKEN}/getUpdates"   # safe to repeat with ↑
  unset TOKEN
  ```
  History only ever shows `${TOKEN}`.

| Method | In history? | Repeatable? |
|---|---|---|
| Token in the command | yes ❌ | ✅ |
| Leading space | no ✅ | retype ❌ |
| `read -s` + variable | no ✅ | ✅ |

### Transport → test worked … real alert didn't
Alert Transports → Telegram (Default Alert on, format blank) → **Test arrived** ✅.
Real test: unplugged the t655 from gi5 → **no Telegram.**

**Cause: LibreNMS 26.x added Operations.** A rule no longer sends to a transport directly:
```
Rule (what) → Operation (when + who) → Transport (how) → phone
```
Docs: *"A rule without an operation still raises the alert. LibreNMS then sends no
notification."* Every rule showed **None** for its operation. My earlier "None = uses the
default transport" was older-version behavior and was **wrong for 26.x**. Rich's call to
search the docs found it.

Fix: created an operation (one segment, steps 1→1, start 0, step duration 60, transport
Telegram) and attached it to every rule. A GUI glitch drew the operation menu in the
wrong place in the rule dialog. It was there; scroll the whole dialog before deciding an option is missing.

**Verified end to end:** fault → poll → rule → operation → Telegram → phone → recovery.

---

## 4. Beryl AX (GL-MT3000) — device 7

### Before touching the gateway
Config exported from the admin panel and **kept out of the repo** (passwords, Wi-Fi keys).

### Install, then stop immediately
```
opkg update && opkg install snmpd
/etc/init.d/snmpd stop      # closes the window where 'public' is open
```

### OpenWrt's stock `/etc/config/snmpd` (UCI)
| Default | Risk |
|---|---|
| `com2sec public` — source `default`, community `public` | Anyone who can reach it can read everything |
| `agentaddress UDP:161,UDP6:161` | Every interface, IPv4 + IPv6, WAN included |
| `com2sec private` + `private_access` — **write all** | A read-**write** community |
| `public6` / `private6` | The same holes over IPv6 |
| sysName `HeartOfGold`, contact `bofh@example.com` | Placeholder jokes |

### Locked down with `uci` (staged until `commit`, like Apply on a firewall)
```
uci set snmpd.@agent[0].agentaddress='UDP:192.168.8.1:161'
read -s C
uci set snmpd.public.community="$C"          # openssl rand -hex 16, unique to this device
uci set snmpd.public.source='192.168.8.24'
uci delete snmpd.private ; uci delete snmpd.private_v1 ; uci delete snmpd.private_v2c
uci delete snmpd.private_usm ; uci delete snmpd.private_access
uci delete snmpd.public6 ; uci delete snmpd.private6
uci delete snmpd.public_v1 ; uci delete snmpd.public_usm
uci set snmpd.@system[0].sysName='beryl-ax'
uci set snmpd.@system[0].sysContact='Rich'
uci set snmpd.@system[0].sysLocation='Apartment media shelf'
uci changes snmpd     # review (not photographed — shows the community)
uci commit snmpd && /etc/init.d/snmpd enable && /etc/init.d/snmpd start
```
Result: v2c read-only, LAN address only, answers only t640-mon. No write community, no IPv6 listener.

### Verified (from t640-mon)
- New community → `STRING: "beryl-ax"` ✅
- `-c public` → `Timeout: No Response` ✅ (the old door is shut)
- Added in LibreNMS via the credential store → **device 7**.

---

## 5. First Beryl alert — and my misread

### `apcli0` "down" at 09:45
Telegram: *Port status up/down — apcli0 — critical*, at **09:45:06**, minutes after the Beryl
was added → LibreNMS's **first look**, not a drop. No recovery followed.

`ifconfig apcli0` → `UP`, but **`RX packets:0`**; `iwinfo apcli0 info` → ESSID unknown,
AP `00:00:00:00:00:00`, signal **-256 dBm** (placeholder = nothing measured), **channel 3 (2.4 GHz)**.
Beryl uptime 59 days. I read this as "the backup WAN has been down since the last reboot."

**Wrong.** The admin page showed **WhiteSky-Prose connected on 5 GHz** (DFS) with its own
CGNAT address and gateway. The Beryl has **one repeater client per radio**:

| Interface | Radio | Role |
|---|---|---|
| `apcli0` | 2.4 GHz | Unused — never connects, `RX 0` forever |
| **`apclix0`** | **5 GHz** | **The real backup WAN** (moved to 5 GHz in SESSION-29) |

Fix: `apcli0` → **Ignore** in LibreNMS; `apclix0` stays alerting.
Lesson: before concluding a link is dead, confirm **which** interface carries it, and check the
GUI's view as well as the CLI's.

### Bonus find in `logread` (the Beryl's RAM log buffer)
A Wi-Fi client `78:2b:46:xx:xx:xx` (Intel OUI) sending **DE-AUTH, then re-associating, about every
9 minutes** (10:31, 10:40, 10:49, 10:51). `DE-AUTH … from STA` = the **client** leaves, not the router.
`/tmp/dhcp.leases` → **testkit**. Likely Wi-Fi power saving (fix on the laptop:
`wifi.powersave = 2` in `/etc/NetworkManager/conf.d/`). Harmless, but now **named**: every MAC
on the network should have a name.

**Network inventory (names, not MACs):** `.213` = Rich's phone (FBI #4 Wi-Fi) · **testkit** = Intel Wi-Fi laptop on the Beryl Wi-Fi.

---

## 6. Beryl syslog → t640-mon

Receiving side first (t640-mon):
```
sudo cp /etc/nftables.conf /etc/nftables.conf.bak-2026-11-11
# 514 rule → ip saddr { 192.168.8.165, 192.168.8.195, 192.168.8.1 } udp dport 514 accept
sudo nft -c -f /etc/nftables.conf && sudo systemctl restart nftables
sudo nft list ruleset | grep 514     # nft sorts the set: { 192.168.8.1, .165, .195 }
```
Sending side (Beryl, UCI):
```
uci set system.@system[0].log_ip='192.168.8.24'
uci set system.@system[0].log_port='514'
uci set system.@system[0].log_proto='udp'
uci commit system && /etc/init.d/log restart
ps | grep [l]ogread          # → /sbin/logread -f … -r 192.168.8.24 514 -u  (the forwarder)
logger "beryl syslog test"
```
Proof, `/var/log/remote/192.168.8.1.log`:
```
… GL-MT3000 logread[…]: Logread connected to 192.168.8.24:514
… GL-MT3000 root: beryl syslog test
```
The hostname is `GL-MT3000` (system hostname), not the SNMP `beryl-ax`. That's expected.
Same `ls` also showed the SG300's `.log.1`, `.log.2.gz`, `.log.3.gz`: **logrotate working**.

**Now logging centrally:** SG300 (.165), t655-fw (.195), Beryl (.1).

---

## State at end of session

| ID | Device | IP | Notes |
|---|---|---|---|
| 1 | sg300-apt | 192.168.8.165 | |
| 2 | pve-node1 | 192.168.10.2 | Boot order fixed (BIOS + efibootmgr); kernel 7.0.14-28-pve |
| 3 | t640-mon | 192.168.8.24 | Telegram transport + operation |
| 4 | docker-agents | 192.168.10.10 | **Disabled** (parked) |
| 5 | hermes | 192.168.10.206 | First nightly backup 11-12 02:00 |
| 6 | t655-fw | 192.168.8.195 | |
| 7 | **beryl-ax** | 192.168.8.1 | **New** — OpenWrt (GL.iNet v4.8.1), SNMP v2c ro, syslog → t640-mon; `apcli0` ignored |

Alerts: every rule → operation "Telegram – notify" → phone.

## Follow-ups

- [x] ~~Copy the final OpenClaw vzdump to Windows~~ — done (9.8 GB, ~3 min)
- [x] ~~Devices up/down rule~~ — exists, with the Telegram operation
- [x] ~~Beryl syslog~~ — live (§6); `apcli0` ignored, `apclix0` alerting (§5)
- [ ] 11-12 morning: Hermes's first dump exists; R2 now holds Hermes; check its size vs 10 GB
- [ ] **2FA on LibreNMS** (`lnms config:set twofactor true` → My Settings → Apple Passwords codes)
- [ ] **Beryl firmware** v4.8.1 → current (update badge showing; kernel built Aug 2025). Back up the config first;
      re-check `snmpd` + `log_ip` afterwards (GL.iNet upgrades can drop opkg packages)
- [ ] testkit: turn off Wi-Fi power saving (optional, quiets the Beryl log)
- [ ] Carried: Proxmox SSH hardening · t655 "Defunct rules" · Hermes `/` growth · thin-pool
      monitoring · Phase 1 (USB Ethernet adapter) · scrub MACs from older runbooks
