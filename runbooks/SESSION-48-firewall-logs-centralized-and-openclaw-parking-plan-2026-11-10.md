# SESSION 48 — Firewall Logs Centralized + OpenClaw Parking Plan — 2026-11-10

**Status: COMPLETE (logging). OpenClaw parking: PLANNED, later this week.**
The t655's firewall log now streams to t640-mon alongside the switch's. A decision
was made to park OpenClaw (VM 100) and give pve-node1 to Hermes until a second
node exists; Hermes inventoried the dependencies without changing anything.

**Continuity:** follows `SESSION-47`. Item 1 of the to-do list (VM 100's 97.9% backup
disk) is superseded by the parking decision — the disk is preserved, not cleaned.

---

## 1. Firewall logs → t640-mon — DONE

### Receiving side first (t640-mon nftables)
Only the switch was allowed to send syslog. Changed the rule to a set:
```
ip saddr { 192.168.8.165, 192.168.8.195 } udp dport 514 accept
```
`sudo nft -c -f /etc/nftables.conf` (syntax check) → `systemctl restart nftables`
→ `nft list ruleset | grep 514` shows both sources.

### Sending side (OPNsense)
System → Settings → Logging → **Remote** → +: enabled, UDP(4), all applications,
host `192.168.8.24`, port 514. Save → Apply. No t655 rule needed: the firewall's
built-in "let out anything from the firewall" rule covers its own outbound traffic.

### Proof — real traffic, no staged test needed
`/var/log/remote/192.168.8.195.log` filled with OPNsense `filterlog` lines
(CSV: rule ID, interface, action, direction, IP version, protocol, src, dst, ports):

| Line (abridged) | Meaning |
|---|---|
| `pass,out … 192.168.8.195 → 192.168.8.24 … 514` | The firewall shipping its log to t640-mon |
| `pass,out … → 72.14.183.39 … 123` | NTP clock sync, allowed out |
| `block,in … 0.0.0.0 → 255.255.255.255 … 68 → 67` | A DHCP request from some device — **ignored**, proving the rogue-DHCP fix holds |
| `block,in … 192.168.8.213 → 224.0.0.251 … 5353` + `fe80::…` → `ff02::fb` | mDNS announcements, dropped by default deny |

**192.168.8.213 identified:** Rich's phone on the FBI #4 Wi-Fi. Every address on the
network should have a name — basic SOC hygiene.

Rotation: covered automatically — the 14-day logrotate rule matches every
`/var/log/remote/*.log`.

---

## 2. Decision: park OpenClaw, give node1 to Hermes

- **Why:** Hermes is the VM that's growing (its root disk alerted at ≥90%) and runs the
  public sites; RAM is the node's constraint. OpenClaw returns when a second node exists.
- **"Park" = clean shutdown, not Proxmox Hibernate.** Hibernate freezes RAM state that
  is tied to the QEMU/Proxmox version; resuming after months of updates can fail. A
  shutdown keeps both disks intact and fully reversible.

### Hermes's dependency inventory (read-only — nothing stopped or moved)
- **Must move first:** blog **Cloudflare Pages deployment** (scheduled publishing, book
  launch, rollback) runs from OpenClaw's VM. Existing pages keep serving if it stops;
  **new deployments would fail.**
- **Already independent:** ASL and the inspected public sites. **All 16 checked public
  URLs** kept their status and exact content.
- **Preserve on the parked VM:** conversations, memory, workspace, Telegram identity
  (@ScargiaBot goes quiet), scheduled jobs, both virtual disks incl. its own backups,
  trading-related data (kept isolated from Hermes).
- **Support mail watcher:** already failing auth/keyring checks → leave parked unless
  the notifications are still wanted.

### Before shutdown (this week)
- [ ] **Final backup covering both disks.** Nightly vzdump skips VM 100's scsi1
      (`backup=0`). One-time: `qm set 100 --scsi1 local-lvm:vm-100-disk-1,backup=1`,
      `vzdump 100 --mode snapshot --compress zstd --storage local`, check `local` has
      room, then set `backup=0` again. **Keep it out of the R2 sync** (10 GB free tier).
- [ ] **Root cron inspection** (Hermes lacked access) — on VM 100:
      `sudo crontab -l`, `ls /etc/cron.d`, `systemctl list-timers --all`.
- [ ] **Move blog deployment to Hermes** without disturbing the current book review —
      with a **new, narrowly scoped Cloudflare API token** (Pages only); revoke the old
      one after testing. Don't copy the old token.
- [ ] Decide on the support-mail watcher.

### At shutdown
- [ ] `qm shutdown 100`; `qm set 100 --onboot 0`
- [ ] LibreNMS: **disable** docker-agents (or it alerts "down" forever)
- [ ] Exclude VM 100 from the nightly vzdump job
- [ ] Then give Hermes the freed RAM; decide its disk (grow root vs. use the
      7%-used 20 GB scsi1)

## Follow-ups (carried)
- [ ] Alert transport (Telegram) · 2FA on LibreNMS · planned Proxmox reboot
- [ ] SSH hardening (VM 100 moot once parked; Proxmox still open) · Beryl as device 7
- [ ] "Defunct rules" review on t655 · Phase 1 (fieldkit USB Ethernet adapter)
