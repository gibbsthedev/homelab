# SESSION 53 — Suricata IDS on the Lab: First Detection, Centrally Logged — 2026-11-13 (night)

**Status: COMPLETE.** The firewall now **detects** attacks, not just blocks them. A test attack
from the lab raised a Suricata alert on the t655, and the alert reached the central log on
t640-mon. Next: push high-priority alerts to the phone.

**Continuity:** follows `SESSION-52` (Phase 1 proven, red/blue exercise). Same night.

```
testkit (lab) ──► vlan01 [Suricata, IDS mode] ──► alert ──► syslog ──► t640-mon /var/log/remote/192.168.8.195.log
```

---

## 1. Capacity check
t655 dashboard: **17.71% memory used** (of 4 GB). Room for a selective ruleset.

## 2. Settings (Services → Intrusion Detection → Administration)

| Setting | Value | Why |
|---|---|---|
| Enabled | ✅ | |
| Capture mode | **PCAP live mode (IDS)** | Watches a copy of traffic, never blocks. A bad rule or the Realtek NIC can't break the network. IPS (Netmap) later, maybe |
| Promiscuous mode | off | Only needed for IPS on VLANs |
| Interfaces | **LAB only** (default was **WAN**) | Watch where the attacker lives. WAN would mostly see home-network chatter |
| Home networks | default (RFC 1918) | Defines "inside" for rule direction |
| Enable syslog alerts (advanced) | ✅ | Needed for alerts to reach t640-mon (see §4) |

## 3. Rules (Download tab)
ET open: **emerging-scan**, **emerging-attack_response**, emerging-policy → Enable selected →
Download & Update Rules. Kept selective: every ruleset costs memory on a 4 GB box.

## 4. Testing — and two failures worth keeping

### Failure 1: the test site was dead
`curl http://testmynids.org/uid/index.html` (the classic IDS test) returned a **parked
"domain may be for sale" page**. The domain expired, so the fake `uid=0(root)` response never came
and the Alerts tab stayed empty. **Correctly empty.** (My recommendation, from memory, unchecked.
Lesson: verify a test target still exists before trusting a negative result.)

**Working test: generate the signature yourself.** The target rule matches the text in any
direction, so testkit sends it:
```
curl -d 'uid=0(root) gid=0(root)' http://example.com
```
`http` (not https) so Suricata can read the payload. → Alert ✅, action **`alert`** (not `drop`):
IDS mode reports and lets the packet through.

### Failure 2: alert on the firewall, not in the central log
`grep -i suricata …192.168.8.195.log` only showed `configd.py … query suricata alerts`, which is
**Rich opening the Alerts tab** (OPNsense logs admin activity, a useful audit trail). Cause:
**syslog alerts not enabled/applied.** Ticked "Enable syslog alerts" → **Apply** → re-sent the curl.

### Success — the alert in the central log
```
suricata[78276]: [1:2100498:7] GPL ATTACK_RESPONSE id check returned root
  [Classification: Potentially Bad Traffic] [Priority: 2]
  {TCP} 192.168.20.185:58802 -> 172.66.147.243:80
```
| Field | Meaning |
|---|---|
| `1:2100498:7` | generator : **SID** : revision. Search alerts by SID |
| Classification / Priority | Severity: 1 = high, 2 = medium, 3 = low. Triage order |
| `{TCP} src:port -> dst:port` | testkit → example.com:80 |

Search by SID: `grep 2100498 /var/log/remote/192.168.8.195.log`

### Warnings read along the way
- **"The syslog output has been deprecated and will be removed in Suricata 9.0."** Works now; a
  future OPNsense update could silently stop forwarding. → Switch to **EVE syslog output** first.
- `flowbit 'ET.ApacheSpark_UnauthRegisterApplication' is checked but not set`: one rule depends on
  a rule in a ruleset that isn't enabled. Harmless; it just never fires.

### Test 2 (nmap NSE user-agent) — pending
`nmap -p 80 --script http-title scanme.nmap.org` may not fire: ET SCAN user-agent rules are often
written for **inbound** traffic ($EXTERNAL_NET → $HOME_NET), and this is outbound. **Rules have a direction.**

---

## 5. Side findings

- **Firewall logs on t640-mon are world-readable**: `grep` worked without `sudo`. → Set rsyslog
  `$FileCreateMode 0640` (+ group `adm`) for `/var/log/remote/`, and fix the existing files.
- **Beryl `ra0` (2.4 GHz radio): 4.75 M receive errors**, ~23 errors/s vs ~60 packets/s, since
  boot 59 days ago. Damaged frames from a crowded 2.4 GHz band, not a security issue. → Channel
  check later; prefer 5 GHz.

---

## State at end of session
- Suricata 8.x on t655, **IDS mode on LAB**, ET open scan / attack_response / policy.
- Alerts in the GUI **and** forwarded to t640-mon via syslog.
- Detection proven with SID 2100498 from 192.168.20.185.

## Next
- [ ] **Alerts to the phone:** small watcher service on t640-mon → Telegram (alerts bot), Priority 1–2
      only, rate-limited, token in a root-only file (`chmod 600`)
- [ ] Switch Suricata syslog → **EVE syslog output** (deprecation)
- [ ] Try test 2 (nmap NSE); learn rule direction
- [ ] rsyslog file permissions 0640 on t640-mon
- [ ] Post-phase1 OPNsense config backup · LibreNMS rediscover t655-fw (LAB graph)
- [ ] Carried: Phase 3 (VLAN 10 behind OPNsense) · Proxmox timezone · SG300 `cisco` user ·
      Beryl stale route · 2FA · SSH hardening · Beryl firmware · Beryl 2.4 GHz errors
