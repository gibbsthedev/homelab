# SESSION 62: Moving the admin workstation from Mint to command, and scrubbing MACs from the repo
**Date:** 2026-11-21 → 2026-11-22 (about 21:55 → 00:50 CST)
**Machines touched:** Mint (old admin laptop), command (new admin laptop), t640-mon, pve-node1, pbs, hermes, wazuh
**Identifiers:** passwords, keys, MACs and serials are never written here. Credentials are in Apple Passwords.

**Continuity:** follows `SESSION-61`. command (HP, 32 GB) has been in `ADMINS` since SESSION-60, but everything that
made Mint *the* admin machine (SSH shortcuts, keys, config backups, the honeypot archive) still lived on Mint.

---

## 1. Real MACs in the public repo
A scan of every `.md` file for MAC-shaped strings found real addresses in **7 files**, not the 2 on the todo list:

| Where | What | Risk |
|---|---|---|
| SESSION-29 | **BSSIDs of the building's Wi-Fi access points** | Public Wi-Fi maps can turn a BSSID into a street location |
| SESSION-29, 36, 39, 41, session2/3, sg300-10.md | Laptops, Beryl, switch, an old VM | Device fingerprinting |

Replaced with `xx:xx:xx:xx:xx:xx`. The two BSSIDs became `<BSSID-A>` / `<BSSID-B>` so the "the BSSID changed" point in
SESSION-29 still reads. Kept: all-zero and broadcast addresses, and Cowrie's **fake** HWaddr in SESSION-20 (a decoy, not
a device). **Git history still contains the old values**: removing them means `git filter-repo` + force push (open decision).

---

## 2. Inventory before moving anything
```
ls -la ~/.ssh        # keys and shortcuts
cat ~/.ssh/config    # no secrets in it, safe to read
du -sh ~/* 2>/dev/null | sort -rh | head -20
crontab -l           # "no crontab for rich"
```
A private key is never `cat`ed or photographed, only listed.

| Item | Decision | Why |
|---|---|---|
| `homelab_mint` key | **Not copied** | command gets its own key; retiring Mint = removing one key |
| `hetzner_mint`, `honeypot_mint` keys | **Retired** | Those servers were deleted (SESSION-43) |
| `~/.ssh/config` | **Recreated** | |
| `homelab/` | **Re-clone** | It's the GitHub repo |
| Kali VirtualBox images (~33 GB, 3 duplicates), Proxmox ISO, VirtualBox VMs | **Skipped** | Re-downloadable |
| OPNsense configs (5), Beryl backups (4) | **Copied, private** | Password hashes, keys, Wi-Fi passwords |
| Honeypot archive (3.7 GB) + `.sha256` + file list | **Copied to quarantine** | Live malware samples |
| Desktop, Pictures, Documents, Videos, metasploitable zip, mp3 | **Copied** | Metasploitable = a future Phase 5 target |

Also found: the `sg300` shortcut still said `User cisco`, a user deleted in SESSION-55. **A config file can outlive the
account it points to.**

---

## 3. Copying: pull from command, over the cable
A temporary SSH server on Mint (`apt install openssh-server`, ufw allowing only `192.168.8.118`), then from command:
```bash
mkdir -p ~/from-mint ~/secure-backups ~/quarantine
chmod 700 ~/secure-backups ~/quarantine
rsync -avh --progress rich@192.168.8.142:{Desktop,Pictures,Documents,Videos,richproject} ~/from-mint/
rsync -avh --progress rich@192.168.8.142:Downloads/{old-official-music-video.mp3,metasploitable-linux-2.0.0.zip} ~/from-mint/
rsync -avh --progress 'rich@192.168.8.142:Downloads/config-t655-fw.lan-*.xml' 'rich@192.168.8.142:Downloads/backup-GL-MT3000-*.tar.gz' rich@192.168.8.142:beryl-pre-4.11-2026-11-13.tar.gz ~/secure-backups/
chmod 600 ~/secure-backups/*
rsync -avh --progress 'rich@192.168.8.142:honeypot-final-2026-10-30*' ~/quarantine/
```
| Folder | Permissions | Holds |
|---|---|---|
| `~/from-mint` | normal | Everyday files |
| `~/secure-backups` | 700 / files 600 | Firewall and router config backups |
| `~/quarantine` | 700 | The malware archive. **Never extracted on this machine** |

- The first run ended with **`rsync error: some files/attrs were not transferred (code 23)`**. A re-run (rsync only
  copies what's missing) finished clean: a one-off.
- Quoted wildcards (`'…/config-*.xml'`) are expanded on **Mint**, not on command, where those files don't exist.

### The checksum that "failed"
```
sha256sum -c honeypot-final-2026-10-30.tar.xz.sha256
# /home/rich/honeypot-final-2026-10-30.tar.xz: No such file or directory   FAILED open or read
```
The `.sha256` file stored Mint's **full path**. The archive was fine; the path wasn't. Checked without editing the file:
```bash
cd ~/quarantine
sed 's|/home/rich/||' honeypot-final-2026-10-30.tar.xz.sha256 | sha256sum -c -
# honeypot-final-2026-10-30.tar.xz: OK
```
(The first try ran from `~` instead of `~/quarantine`. Read the prompt.)

---

## 4. SSH: one key per machine, same shortcuts
command already had `~/.ssh/id_ed25519` (kept, never overwritten). `~/.ssh/config` on command:

| Shortcut | Host | User | Auth |
|---|---|---|---|
| `sg300` | 192.168.8.165 | current switch admin (not `cisco`) + legacy KEX/cipher options | Password (the switch) |
| `t640-mon` | 192.168.8.24 | scgiky | command's key |
| `pve` | 192.168.10.2 | root | command's key |
| `pbs` | 192.168.10.30 | root | command's key |
| `hermes` | 192.168.10.206 | ec2-user | command's key |
| `wazuh` | 192.168.10.20 | (VM user) | command's key |

All except `sg300` use `IdentityFile ~/.ssh/id_ed25519` + `IdentitiesOnly yes`.

**t640-mon is key-only**, so command couldn't use `ssh-copy-id`. Mint, which t640 already trusts, passed command's
**public** key along:
```bash
# command
scp ~/.ssh/id_ed25519.pub rich@192.168.8.142:/tmp/command.pub
# Mint
cat /tmp/command.pub | ssh t640-mon 'cat >> ~/.ssh/authorized_keys'
```
`>>` appends. A single `>` would have replaced Mint's key and locked Mint out.

The servers that still allow passwords got the key the normal way, one password prompt each:
```bash
ssh-copy-id pve ; ssh-copy-id pbs ; ssh-copy-id hermes ; ssh-copy-id wazuh   # "Number of key(s) added: 1"
```
✅ **command now logs in to all six by name, and to the five key-capable ones without a password.** Passwords still work on
pve/pbs/hermes/wazuh: turning them off is a separate, careful session (second session open, console as rollback).

---

## 5. Inside the old Mint laptop
| Part | Found | Meaning |
|---|---|---|
| Storage | **Spinning 2.5" SATA hard drive** | The main reason it feels slow |
| RAM | 1 × 4 GB **DDR3**, second slot empty | Upgradable, but DDR3 only. The spare DDR4 sticks don't fit |

Future: a Phase 5 target (with a cheap SATA SSD), or retired. **Either way the drive gets a full overwrite (`nwipe`)
first**: it still holds Mint's private keys, firewall and router config backups and the malware archive. A reinstall
doesn't erase a hard drive.

---

## 6. State at end of session
| | |
|---|---|
| Admin workstation | **command**: files, config backups, quarantined archive, SSH shortcuts to every device |
| Mint | Still in `ADMINS`, t640 nftables + nginx, and t640's `authorized_keys` until fully retired |
| Repo | Real MACs replaced in 7 files (history unchanged) |

## 7. Lessons learned
- **Search before trusting a todo list.** "2 files with MACs" was 7, including geolocatable BSSIDs.
- **Inventory first, then sort:** copy, re-download, retire. 36 GB of Downloads came down to a few MB that mattered.
- **Don't copy private keys between machines.** Give each machine its own and authorize it; retiring a machine = removing one key.
- **`>>` appends, `>` replaces.** On `authorized_keys`, the difference is a lockout.
- **A checksum file can hold an absolute path.** A "FAILED" can be the path, not the data.
- **Quote remote wildcards** so the remote side expands them.
- **rsync is safe to re-run.** It copies only what's missing, so a code 23 is worth one retry before digging.
- **Config files outlive accounts.** The `cisco` shortcut pointed at a user deleted weeks earlier.
- **Old drives get wiped, not reinstalled,** before reuse.

## 8. Next
1. Turn off the temporary SSH server on Mint (`systemctl disable --now ssh`, remove openssh-server, delete the ufw rule).
2. After a day or two on command only: remove Mint (.142/.186) from `ADMINS`, t640 nftables and nginx; remove `homelab_mint` from t640's `authorized_keys`; LibreNMS 2FA.
3. Key-only SSH on pve, pbs, hermes, wazuh.
4. Decide on rewriting git history for the MACs.
5. Later: `nwipe` Mint's drive.
