# SESSION 63: Backup failures now reach my phone, and reading PBS's first numbers
**Date:** 2026-11-22 (about 02:00 → 09:05 CST, with a break)
**Machines touched:** pve-node1, pbs, VM 102 wazuh, command
**Identifiers:** passwords, tokens, MACs and serials are never written here. Credentials are in Apple Passwords.

**Continuity:** follows `SESSION-62`. Closes the item that has been open since SESSION-59: two backup failures went
unnoticed because nothing told me.

---

## 1. "19h until full" on PBS: reading it before reacting
The PBS dashboard estimated the datastore would be full in 19 hours. That estimate draws a straight line through recent
growth, and the datastore went from empty to two full backups in one day. The real numbers:

**Datastore → homelab → Summary**
| | Value |
|---|---|
| Usage | 39.24 GB of 97.48 GB (40%), including ~4 GB of OS |
| Backups | 2 groups (vm/101, vm/102), 4 snapshots |
| Deduplication factor | **5.14** |

**The first incremental run (Proxmox → Task History → 00:30 Backup Job)**, Wazuh:
```
INFO: scsi0: dirty-bitmap status: OK (1.4 GiB of 60.0 GiB dirty)
INFO: using fast incremental mode (dirty-bitmap), 1.4 GiB dirty of 60.0 GiB total
INFO: backup was done incrementally, reused 58.62 GiB (97%)
INFO: transferred 1.39 GiB in 7 seconds (203.4 MiB/s)
```
- **Dirty bitmap:** QEMU tracks which blocks changed since the last backup, so only those are read and sent.
- The second snapshot cost almost nothing, which is why 4 snapshots take the space of about one.
- Real daily cost: about 1–3 GB. Wazuh's change is mostly new log data, which doesn't deduplicate, so with ~3 months of
  restore points I expect the datastore to settle around 50–65%. LibreNMS's "Space on / ≥90%" alert watches it.
- **Decision:** no change. Check the same screen a week later; if it's still climbing, trim Wazuh's monthly restore points.

**Lesson: an estimate built on one day of data says more about the day than about the trend.**

---

## 2. Wazuh at 93% memory
The Proxmox graph showed VM 102 at 93% RAM. Checked inside the VM, not on the host graph:
```
ssh wazuh
free -h
#        total  used   free   shared  buff/cache  available
# Mem:   7.8Gi  3.2Gi  323Mi  1.3Mi   4.5Gi       4.5Gi
# Swap:  4.0Gi  768Ki  4.0Gi
```
| | Meaning |
|---|---|
| used 3.2 Gi | Mostly the indexer's fixed Java heap, reserved at start |
| buff/cache 4.5 Gi | File cache, handed back on demand |
| **available 4.5 Gi**, swap ~0 | Plenty of headroom |

**No change** (8 GB stays). The login banner also offered **Ubuntu 26.04**: not taken, because Wazuh 4.14 supports Ubuntu up
to 24.04 (SESSION-59). The 23 normal 24.04 updates are safe; the Wazuh repo stays disabled on purpose.

**Lesson: "used" on a host graph includes cache. `available` inside the guest is the number that means something.**

---

## 3. The line that explained two silent failures
In the same backup log:
```
INFO: notification matched the following matchers: default-matcher
INFO: notified via target 'mail-to-root'
```
Proxmox *was* reporting every backup, to a mailbox on the server that nobody reads.

### 3.1 Proxmox: a webhook to Telegram
**Datacenter → Notifications → Notification Targets → Add → Webhook**
| Field | Value |
|---|---|
| Name | `telegram` |
| Method / URL | `POST` / `https://api.telegram.org/bot{{ secrets.token }}/sendMessage` |
| Header | `Content-Type: application/json` |
| Secrets | `token`, `chat` (stored in a root-only file, never in the URL or config) |

```
{
  "chat_id": "{{ secrets.chat }}",
  "text": "Proxmox: {{ escape title }} ({{ severity }})"
}
```
The message is short on purpose: Telegram rejects messages over 4,096 characters, and a failed backup's full log can be
longer. The alert says *that* something failed; the task log has the details. **Test** → arrived on the phone.

### 3.2 Proxmox: which notices go there
**Notification Matchers → Add** `backups-to-telegram`. The form starts with a single rule, "Always match", and its
**Rule Type** turns into a group:
```
All sub-rules match
 ├ Field: type is vzdump
 └ Severity: error, warning
```
Targets: `telegram`. Successful backups stay off the phone: three "all good" messages a night would train me to ignore
the channel. Both backup jobs already "use global notification settings", so nothing changed on the jobs.

| Event | Goes to |
|---|---|
| A backup fails or warns (00:30 PBS job or 02:00 local job) | **Phone** |
| Every backup notice | `mail-to-root`, as before |

### 3.3 PBS: the same for its own jobs
**Configuration → Notifications:** the same webhook (`PBS: …` in the text), test ✅, matcher `pbs-to-telegram` =
**severity error, warning** with no type rule, because any PBS job failure (garbage collection, prune, verify) matters.
Datastore → Options → notification mode: notification system.

**Known limit:** this catches a job that runs and fails. A job that **never runs** (host off, schedule disabled) sends
nothing. The weekly look at PBS covers that for now.

---

## 4. Housekeeping
- **The todo list is now a Whiteboard**: six lanes (do first · finish the Mint move · security · backups · cleanups · big
  builds), with done items marked ✅. Linked from the Project's handoff.
- **Restore tests** (file-level and a full VM restore) wait for the 500 GB SSD: a full Hermes restore into the current
  thin pool (about 63 GB free) could fill it, and a full thin pool freezes every VM (SESSION-59).
- **A confusing push checked:** after a commit I wasn't sure about, a fresh clone showed it was exactly the MAC scrub, with
  nothing lost.
- command (32 GB) handles everything Mint did. Its 1 TB hard drive gets an SSD eventually.

## 5. State at end of session
| | |
|---|---|
| Backup alerts | **Proxmox + PBS → Telegram**, failures and warnings only |
| PBS | 40% used, dedup 5.14, incrementals ~1.4 GB / 7 s (Wazuh) |
| Wazuh VM | 8 GB, 4.5 GB available, no change |

## 6. Lessons learned
- **An estimate from one day of data is noise.** Look at the real usage and the per-night change.
- **Incremental backups read only dirty blocks.** 1.4 GiB of 60 GiB, 97% reused, 7 seconds.
- **Check memory inside the guest.** Host graphs count cache as used; `available` is what matters.
- **Don't upgrade a SIEM's OS past what the SIEM supports,** however loudly the banner offers it.
- **"Notified" doesn't mean someone read it.** Find where notifications actually go.
- **Alert on failures only.** A channel full of "OK" gets ignored.
- **Keep secrets out of URLs:** template them in from a protected store.
- **Failure alerts can't see a job that never ran.** That needs a separate check.

## 7. Next
1. Finish the Mint move: remove .142/.186 from t640 nftables, nginx and `authorized_keys`, then from `ADMINS`; delete the disabled GUI rule; LibreNMS 2FA.
2. Key-only SSH on pve, pbs, hermes, wazuh.
3. Restore tests once the SSD is in.
4. Next Sunday: PBS usage again.
