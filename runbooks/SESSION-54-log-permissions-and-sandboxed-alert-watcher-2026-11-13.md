# SESSION 54 — Locked-Down Logs and a Sandboxed Alert Watcher — 2026-11-13

**Status: COMPLETE.** Two findings from SESSION-53 closed: the central logs were world-readable,
and the Telegram watcher ran as root. Logs are now `0640 root:adm`; the watcher runs as its
own user inside a systemd sandbox. **Exposure score 9.6 → 8.5 → 1.7**, and alerts still reach the phone.

**Continuity:** follows `SESSION-53` (Suricata → central log → phone).

---

## Part A — Logs readable only by `adm`

```
BEFORE  -rw-r--r-- root root    any user on t640-mon could read firewall/IDS logs
AFTER   -rw-r----- root adm     root writes, group adm reads, everyone else nothing
```
`adm` = Debian's group for "may read logs" (the same group `journalctl`'s hint names).

1. **New files** — `/etc/rsyslog.d/10-remote.conf` (backup `.bak` first):
   ```
   action(type="omfile" dynaFile="RemoteFile"
          fileOwner="root" fileGroup="adm" fileCreateMode="0640"
          dirOwner="root" dirGroup="adm" dirCreateMode="0750")
   ```
   Validated with `rsyslogd -N1` **before** restarting. While rsyslog is down, UDP syslog from
   three devices is simply lost, so never restart an unvalidated config.
2. **Existing files** — `chgrp adm` + `chmod 750` (dir) / `chmod 640` (files).
3. **Rotation** — `/etc/logrotate.d/remote`: `create 0640 root adm`; checked with `logrotate -d`.
4. **Access the proper way** — `usermod -aG adm scgiky` (applies at next login). Logs and
   `journalctl` now work without `sudo` because of group membership, not open permissions.

Habit noted: a `sudo -i` root shell was left open from the Telegram test. Exit root shells
when done; `sudo` per command leaves an audit trail.

---

## Part B — Watcher as a non-root, sandboxed service

**Why:** the script handles **attacker-controlled text** (the alert contains the packet's content).
It's passed safely as data (quoted, URL-encoded, never executed), but if that ever slipped, root
would be the one executing it. Least privilege: read logs, read the token, write one file — nothing else.

| Step | Command | Result |
|---|---|---|
| Service user | `useradd --system --no-create-home --shell /usr/sbin/nologin --groups adm idsalert` | No password, no home, no login; can read logs via `adm` |
| Secrets | `chown root:idsalert` + `chmod 640` `/etc/ids-telegram.env` | Only root and the service can read the token (not even scgiky) |
| Output log | `/var/log/ids-telegram.log` → `idsalert:adm 640` | The one writable file |
| Proof | `ps -o user=,cmd= -u idsalert` | bash, tail, grep all owned by **idsalert** |

### The sandbox — measured with `systemd-analyze security`

| Stage | Score |
|---|---|
| Plain service (systemd default) | ~9.6 UNSAFE |
| `User=`, `NoNewPrivileges`, `ProtectSystem=strict`, `ProtectHome`, `PrivateTmp`, `ReadWritePaths=` | **8.5 EXPOSED** |
| + the lines below | **1.7 OK** 🙂 — and the phone test still passed |

```
[Service]
ExecStart=/usr/local/bin/ids-telegram.sh
Restart=always
RestartSec=10
User=idsalert
Group=idsalert
NoNewPrivileges=yes
ProtectSystem=strict
ProtectHome=yes
PrivateTmp=yes
ReadWritePaths=/var/log/ids-telegram.log
CapabilityBoundingSet=
PrivateDevices=yes
ProtectClock=yes
ProtectHostname=yes
ProtectKernelTunables=yes
ProtectKernelModules=yes
ProtectKernelLogs=yes
ProtectControlGroups=yes
ProtectProc=invisible
ProcSubset=pid
RestrictAddressFamilies=AF_INET AF_INET6 AF_UNIX
RestrictNamespaces=yes
RestrictRealtime=yes
RestrictSUIDSGID=yes
LockPersonality=yes
MemoryDenyWriteExecute=yes
SystemCallArchitectures=native
SystemCallFilter=@system-service
UMask=0027
```
| Lines | Take away |
|---|---|
| `CapabilityBoundingSet=` (empty) | Every special kernel power (`CAP_*`) |
| `Protect*`, `PrivateDevices` | Kernel tunables/modules/logs, clock, hardware devices |
| `ProtectProc`, `ProcSubset` | Visibility of other processes |
| `RestrictAddressFamilies` | Raw sockets / exotic protocols — only normal IP + local sockets |
| `SystemCallFilter=@system-service` | Any system call a normal service doesn't use |

Method: list what it needs, deny the rest, **measure**, then **test** — a low score that breaks
the service is worth nothing. If a line breaks it, comment out one line at a time.

---

## Part C — Suricata EVE syslog
Done earlier (during SESSION-53): EVE syslog output enabled on the t655 and running.

---

## Follow-ups
- [ ] Before Suricata 9.0: confirm the watcher matches EVE lines, then turn off the deprecated syslog output
- [ ] Optional: daily summary of alerts the 1/min rate limit suppressed
- [ ] Carried: Proxmox timezone · SG300 `cisco` user · Beryl stale route · 2FA (LibreNMS) ·
      SSH hardening (Proxmox + Hermes) · Beryl firmware · post-phase1 OPNsense backup ·
      Phase 3 (VLAN 10 + management VLAN) · RAM upgrade → Wazuh
