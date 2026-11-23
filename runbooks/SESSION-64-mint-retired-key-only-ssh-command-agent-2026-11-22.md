# SESSION 64: Mint cut out of every allow-list, key-only SSH, and command watched by Wazuh
**Date:** 2026-11-22 (about 09:45 → 16:38 CST)
**Machines touched:** command, t640-mon, t655-fw (OPNsense), pbs, pve-node1, VM 102 wazuh, Beryl
**Identifiers:** passwords, keys, MACs and serials are never written here. Credentials are in Apple Passwords.

**Continuity:** follows `SESSION-63`. SESSION-62 moved the admin work to command; this session removes the old Mint laptop's
access everywhere and turns off password logins on the servers command can now reach with its key.

**Method for every change today:** keep the first session open, make the change, **test from a second terminal**, and know
the rollback (a backup file, or a console that isn't SSH) before you press Enter.

---

## 1. Mint's temporary SSH server and the Beryl
- The SSH server opened on Mint for the SESSION-62 file copy: **disabled and `openssh-server` removed.**
- **Beryl:** command's key added and a `Host beryl` shortcut in `~/.ssh/config`. Key login works.
  The Beryl runs **Dropbear**, so its keys live in `/etc/dropbear/authorized_keys`. To check after the next GL.iNet firmware
  upgrade (they dropped snmpd in SESSION-57).

## 2. Cutting Mint out: four allow-lists
Mint (.142 wired / .186 Wi-Fi) was trusted in four separate places (found in SESSION-60). One at a time, each tested:

| # | Where | Change | Backup | Test |
|---|---|---|---|---|
| 1 | **t640 nftables** | `ip saddr { 192.168.8.118, 192.168.8.213 } tcp dport { 22, 80 } accept` | `/etc/nftables.conf.bak-2026-11-22` | `nft -c` silent · `ssh t640-mon` · LibreNMS loads |
| 2 | **nginx (LibreNMS)** | `allow .186` and `allow .142` removed; `allow .118; deny all;` | `/root/librenms.conf.bak-2026-11-22` | `nginx -t` OK · LibreNMS loads, no 403 |
| 3 | **t640 `authorized_keys`** | Mint's key line removed | `~/.ssh/authorized_keys.bak-2026-11-22` | `ssh t640-mon` from a second terminal |
| 4 | **OPNsense alias `ADMINS`** | Now **.118 (command), .166 (Windows backup laptop)** | — | GUI still logged in · `ssh pve` · Wazuh dashboard |

Listing keys without printing them:
```
awk '{print NR": "$1" … "$NF}' ~/.ssh/authorized_keys     # line number, key type, comment
sed -i '<n>d' ~/.ssh/authorized_keys                       # delete line n
```
The alias went **last**, from command (which stays in it), so there was no way to lock myself out mid-change.
Also deleted: the "Mint wifi → GUI" WAN rule, disabled since 11-19 with nothing missing it.

**Mint now has no access to anything.** Its drive gets a full overwrite (`nwipe`) before any reuse.

---

## 3. Key-only SSH on three servers
One drop-in file per server, so the vendor's own `sshd_config` stays untouched:

| Server | File | `PermitRootLogin` |
|---|---|---|
| pbs | `/etc/ssh/sshd_config.d/10-hardening.conf` | `prohibit-password` (root logs in by key) |
| pve-node1 | `/etc/ssh/sshd_config.d/10-hardening.conf` | `prohibit-password` |
| wazuh | `/etc/ssh/sshd_config.d/10-hardening.conf` | **`no`** (log in as the user, then `sudo`) |

Plus on all three:
```
PasswordAuthentication no
KbdInteractiveAuthentication no
```
Then:
```
sshd -t                     # silent = valid
systemctl reload ssh
sshd -T | grep -E 'permitrootlogin|passwordauthentication|kbdinteractive'
```
`sshd -T` prints what's **in effect**. `prohibit-password` shows under its older name, `without-password`.

**Order matters.** sshd reads the drop-in folder before the main file, and **the first value wins**:
- Proxmox's main file may still say `PermitRootLogin yes`: the drop-in wins.
- Ubuntu often ships `50-cloud-init.conf` with `PasswordAuthentication yes`: `10-…` is read first and wins.

**The test that proves it** (second terminal):
```
ssh pbs                                   # logs in with the key
ssh -o PubkeyAuthentication=no pbs        # Permission denied (publickey)
```
Rollbacks, never needed: the PBS console, Proxmox's **web Shell** (not SSH), the Wazuh VM's console in Proxmox.

**Not done yet:**
- **Hermes.** Red Hat may ship `01-permitrootlogin.conf`, read before a `10-` file, so Hermes gets `00-hardening.conf`, and
  the old temporary password override from the migration has to be found and removed first.
- The Beryl.
- The Windows backup laptop has no key, so it can't SSH into these anymore. Accepted: it's an emergency machine, and the web
  GUIs still work.

---

## 4. Wazuh agent on command, and its first detection
command is the most valuable machine on the network, so it gets watched too.

**Firewall (OPNsense, WAN):** `ADMINS → 192.168.10.20` TCP `WAZUH_AGENT_PORTS` (1514/1515), "Admins → Wazuh agent".
One rule for command now and the Windows laptop later. `nc -zv 192.168.10.20 1515` → succeeded.

**Agent:** deployed from the dashboard (DEB amd64, server 192.168.10.20, name `command`), `<address>` checked before
starting. **5 agents:** hermes, pve-node1, t640-mon, pbs, command.

**First detection:**
```
sudo -k
sudo ls        # three wrong passwords
```
Wazuh → Threat Hunting, `agent.name:command` → sudo authentication-failure events within minutes.

### Why command shows logins and Proxmox doesn't
| Machine | Where logins are written | In Wazuh? |
|---|---|---|
| command (Mint) | a log file the agent already reads | ✅ |
| pve-node1, pbs (Debian 13) | **journald only** | ❌ (SESSION-59's blank search) |

Same agent, same rules, different log destination. That isolates the cause. The fix: a journald `<localfile>` block in the
agent config, filtered to login programs (`sshd`, `sshd-session`, `sudo`, `su`, `login`) so Proxmox's chatty services don't
fill Wazuh's disk and its backups. Planned, not applied yet.

---

## 5. State at end of session
| | |
|---|---|
| `ADMINS` | .118 (command), .166 (Windows backup) |
| t640-mon | Trusts only command (nftables, nginx, SSH key) |
| Key-only SSH | pbs, pve-node1, wazuh (+ t640-mon since SESSION-44) · Hermes and the Beryl to do |
| Wazuh | 5 agents; first detection from command |
| Mint | No access anywhere; drive to be wiped |

## 6. Lessons learned
- **Test every access change from a second session,** and know the rollback before pressing Enter.
- **Remove access from the outside in.** The alias went last, from a machine that stays in it.
- **List keys by comment, not by content:** `awk '{print NR": "$1" … "$NF}'`.
- **Drop-in files beat editing vendor configs,** and the first value wins: name the file so it's read first.
- **`sshd -T` shows the truth,** not what a file says.
- **Prove the negative:** `ssh -o PubkeyAuthentication=no` must be refused.
- **Compare a working machine with a broken one** to isolate a cause: same agent, different log destination.

## 7. Next
1. Wazuh journald block on pbs, then pve-node1; re-test SSH logins (expect "sshd: authentication success").
2. Key-only SSH on Hermes (`00-hardening.conf`, remove the old override first) and on the Beryl.
3. Wazuh agent on the Windows laptop (firewall rule already in place).
4. When the SSD arrives: LibreNMS 2FA, the restore tests, the second VM storage.
