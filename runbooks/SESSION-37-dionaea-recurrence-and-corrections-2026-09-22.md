# SESSION 37 — Dionaea Froze Again: What It Disproved, and a Self-Healing Watchdog

**Date:** 2026-09-22
**Box:** Hetzner Cloud VPS `ubuntu-4gb-hel1-2` (62.238.47.215)
**Thread:** honeypot VPS
**Continuity:** Follows `SESSION-35-dionaea-silent-failure-2026-09-20.md` for the
honeypot thread. `SESSION-35-vlan10-migration-staging` and
`SESSION-36-proxmox-vlan10-cutover` belong to the switch/homelab thread.

**Summary.** Dionaea froze a second time, 24.5 hours after the session-35 restart. The
heartbeat detected it within 20 minutes — then sent 73 near-identical alerts because of
a suppression bug, and nobody acted for 24 hours. The recurrence also broke the
root-cause story written in session 35: three separate pieces of evidence disprove it.
The watchdog was rewritten to suppress correctly and to recover automatically, with
evidence preserved first. **The underlying cause is unknown.**

---

## 1. The recurrence

```text
HONEYPOT HEALTH ALERT

DIONAEA STALE: no connections for 24.2h (last 2026-09-21 12:29 UTC)
DIONAEA CPU RUNAWAY: 99.56%
```

| Event | Time (UTC) |
|---|---|
| Restart after incident 1 | 2026-09-20 ~12:01 |
| Last database write | 2026-09-21 12:29:52 |
| First heartbeat alert | within 20 min of the freeze |
| Manual restart | 2026-09-22 12:45:19 |

**Detection worked. Response didn't.** The CPU check fired on its first run after the
freeze. The restart came a day later.

### Why: 73 alerts

```bash
grep -c "HEALTH ALERT" /var/log/honeypot-alerts.log
# 73
```

73 alerts × 20 minutes ≈ 24.3 hours — one alert on *every* run from the freeze to the
restart. The session-35 heartbeat compared the full alert text to the previous run:

```python
if alerts and alerts != prev:        # alerts is a list of strings
```

and the strings contain values that change every run:

```text
DIONAEA CPU RUNAWAY: 99.58%
DIONAEA CPU RUNAWAY: 100.08%
```

They never matched, so suppression never engaged. An alert that repeats every 20
minutes for a day stops being read.

> **LESSON — detection without response is half a system.** The monitor found the
> failure in 20 minutes and it still ran for 24 hours. Either a human reliably acts on
> the alert, or the system acts on its own.

> **LESSON — suppress on *which* problems exist, not on the text describing them.**
> Any number inside an alert string makes every alert unique.

---

## 2. What the recurrence disproved

Session 35 concluded that sustained SIP volume from `82.165.65.238` created SipCall
objects whose per-call timer threads accumulated until the event loop starved, and
that 711 Cleanup lines in one second were ~11.8 hours of backlogged timer fires. All
three parts of that are wrong.

### 2.1 No flood this time

SIP sources in the 24 hours before the second freeze:

```text
172.110.223.179 | 66     <- the ~27-minute metronome scanner, normal cadence
152.32.146.196  | 12
162.217.103.70  | 10
190.211.252.3   |  6
13.86.115.170   |  4
```

About 100 SIP connections in a day. Before incident 1, `82.165.65.238` alone sent
5,970 in 20 hours. **Dionaea froze under ordinary load**, so a flood is not necessary
to trigger it. The 5,970 was real but coincident, not causal — or at most one of
several ways to reach the same state.

### 2.2 No thread accumulation

`docker stats` reported `PIDS 5` in **both** frozen states. If each SipCall had left a
live timer thread behind, that number would have climbed into the hundreds or
thousands. It didn't move.

This evidence was available during session 35 and was not reconciled with the
mechanism being proposed. That's the actual error — not missing data, but not checking
the data already in hand against the explanation.

### 2.3 The "backlog" count is a buffer size

Cleanup lines in the frozen error logs:

```text
incident 1: 711 + 1 partial, all at 2026-09-16 13:36:53
incident 2: 712,            all at 2026-09-21 12:29:52
```

Two failures after **different** runtimes (§3) produced **the same count**. A number
that encodes elapsed time cannot do that.

The sizes settle it:

```text
/root/dionaea-postmortem-20260922-131720-auto   lines/bytes:  713  49152
/root/dionaea-postmortem-20260922-124507        lines/bytes:  713  49152
/root/dionaea-postmortem-20260920-120110        lines/bytes:  712  49152
```

Every snapshot is **exactly 49,152 bytes = 12 × 4,096**. At ~69 bytes per line,
712 lines ≈ 48 KiB. The count is measuring a buffer, not a duration.

Supporting detail: the `-auto` snapshot was taken **32 minutes after a clean restart**,
and the file's size and modification time were unchanged. Cleanup logs one ~69-byte
line per minute; 32 lines ≈ 2.2 KiB, under one 4 KiB block. That is consistent with
Dionaea writing this log through a buffer that only reaches disk in whole 4 KiB
blocks.

**Consequence:** the error log is not a real-time record. "712 lines in one second,
nothing before or after" relied on it more than it supports.

---

## 3. Uptime at failure — a weaker pattern than claimed

```bash
grep -o '"StartedAt": *"[^"]*"' /root/dionaea-postmortem-20260920-120110/docker-inspect.json
# "StartedAt": "2026-09-15T19:44:03.591718347Z"
```

| Incident | Process started | Froze | Runtime |
|---|---|---|---|
| 1 | 2026-09-15 19:44:03 | 2026-09-16 13:36:53 | **17.9 h** |
| 2 | 2026-09-20 ~12:01 | 2026-09-21 12:29:52 | **24.5 h** |

"Within about a day of starting," on two data points. A counter or timer that trips at
a fixed runtime would land far closer together than 6.6 hours apart. Not ruled out,
not supported either.

The first attempt at this check printed nothing: `docker inspect` writes
`"StartedAt": "…"` with a space after the colon, and the pattern
`'"StartedAt":"[^"]*"'` didn't allow for it. Corrected to `'"StartedAt": *"…'`.

---

## 4. What is actually known

**Established across both incidents:**

| Signal | Observation |
|---|---|
| Main thread | Pinned runnable in userspace (`Rsl`, empty `wchan`, empty kernel stack) |
| Worker threads | Idle in `futex_do_wait` |
| CPU | ~100% of one core, sustained |
| Memory | Flat (~107 MB), no leak |
| `PIDS` | 5, unchanged |
| Database | Stops writing at the freeze |
| Docker | `running`, not OOM-killed, `RestartCount=0` |
| Error log | A burst of SIP Cleanup warnings stamped at the freeze second |
| Recovery | `docker restart` restores full function immediately |
| Runtime at failure | 17.9 h and 24.5 h |

**Not established:** the trigger, the loop the main thread is spinning in, and whether
SIP is involved at all beyond being the module that logged last. Two failures in ~7
days of runtime.

**Disproven:** SIP flood as a required trigger; timer-thread accumulation; the Cleanup
count as a measure of elapsed time.

---

## 5. The watchdog, rewritten

Session 35 set the rule: *accept and monitor now, auto-restart if it recurs.* It
recurred.

### 5.1 Suppression on keys

Each problem gets a stable key; the human-readable text carries the numbers:

```python
alerts = {}                                   # stable key -> text
alerts["dionaea_cpu"]   = f"DIONAEA CPU RUNAWAY: {cpu}%"
alerts["dionaea_stale"] = f"DIONAEA STALE: no connections for {db_age_h:.1f}h"

keys = sorted(alerts)
if keys and keys != state.get("keys"):        # the SET of problems changed
    tg("HONEYPOT HEALTH ALERT\n\n" + "\n".join(alerts[k] for k in keys))
elif state.get("keys") and not keys:
    tg("HONEYPOT RECOVERED: all checks passing")
state["keys"] = keys
```

One message when something breaks, one when the set of problems changes, one on
recovery. The state file migrates automatically from the v1 list format.

### 5.2 Auto-recovery on the combined signature

Restart only when **both** signals line up — each alone has an innocent explanation:

```python
runaway = ("dionaea_cpu" in alerts                     # CPU high now
           and "dionaea_cpu" in state.get("keys", [])  # ...and on the previous run
           and db_age_h is not None and db_age_h > 0.5)  # ...and DB quiet 30+ min
```

- High CPU alone could be a genuine burst of traffic — but then the database would be
  fresh.
- A quiet database alone could be a quiet period on the internet.
- **High CPU on two consecutive runs plus a quiet database** is the failure signature
  from both incidents.

Worst case from freeze to recovery: ~40 minutes.

### 5.3 Evidence before repair

Before restarting, the watchdog copies what the analysis in §2–§4 depended on:

```python
started = sh("docker", "inspect", "-f", "{{.State.StartedAt}}", "dionaea")
d = f"/root/dionaea-postmortem-{time.strftime('%Y%m%d-%H%M%S')}-auto"
for f in [DB] + glob.glob("/opt/dionaea/var/log/dionaea/dionaea*.log"):
    shutil.copy2(f, d)
open(f"{d}/threads.txt", "w").write(sh("ps", "-Lp", pid, "-o", "pid,tid,pcpu,stat,wchan:32,comm"))
open(f"{d}/meta.txt",    "w").write(f"started_at={started}\ncpu={cpu}\ndb_age_h={db_age_h}\n")
```

`started_at` is there specifically to test §3: every future freeze adds a runtime data
point with no manual forensics.

### 5.4 Loop guard

```python
RESTART_GAP_H = 6
if runaway and time.time() - state.get("last_restart", 0) > RESTART_GAP_H * 3600:
```

At most one automatic restart per six hours. If it's freezing faster than that,
something has changed and a human should look.

### 5.5 Tested

A test hook forces the restart path once:

```python
if os.environ.get("HB_FORCE_RESTART"):
    runaway, state["last_restart"] = True, 0
```

```bash
HB_FORCE_RESTART=1 python3 /opt/honeypot-alerts/heartbeat.py
```

Result — evidence folder created, Telegram `DIONAEA AUTO-RESTARTED` delivered:

```text
/root/dionaea-postmortem-20260922-131720-auto/
  dionaea-errors.log   49152
  dionaea.log          69632
  dionaea.sqlite       40886272
  meta.txt             81
  threads.txt          324

meta.txt:
  started_at=2026-09-22T12:45:19.706909733Z
  cpu=0.01
  db_age_h=0.020657569302452935
```

cpu=0.01 and a 0.02-hour-old database are correct for a test run against a healthy
process. **Side effect:** the test counts as a restart, so auto-recovery was locked out
until ~19:17 UTC on 2026-09-22. A freeze inside that window would alert but not
self-heal.

---

## 6. Downtime so far

| Incident | Frozen | Recovered | Lost |
|---|---|---|---|
| 1 | 2026-09-16 13:36:53 | 2026-09-20 ~12:01 | ~94 h |
| 2 | 2026-09-21 12:29:52 | 2026-09-22 12:45:19 | ~24 h |

About 118 hours of Dionaea coverage lost since deployment on 2026-08-31. Any analysis
spanning those windows has holes in it.

---

## 7. Files changed

| Path | Change |
|---|---|
| `/opt/honeypot-alerts/heartbeat.py` | Rewritten: key-based suppression, auto-recovery, evidence capture, 6 h guard, test hook |
| `/opt/honeypot-alerts/heartbeat_state.json` | Format changed from list to dict (migrated automatically) |
| `/root/dionaea-postmortem-20260922-124507/` | **NEW** — incident 2 evidence (manual) |
| `/root/dionaea-postmortem-20260922-131720-auto/` | **NEW** — forced test of auto-capture |
| `SESSION-35-dionaea-silent-failure-2026-09-20.md` | Correction banner added; body left as written |

---

## 8. Open items

- **Root cause is unknown.** The next useful evidence is from inside the spinning
  thread — a Python-level stack (`py-spy dump`) or native one (`gdb -p` / `eu-stack`)
  captured *before* restart. Candidate to add to the auto-capture, if either tool can
  run against the container's process.
- **Watch the `started_at` values** in future `-auto` folders to confirm or kill the
  "within a day" pattern.
- **Test disabling SIP** only if the pattern holds and captures point there. Two
  freezes that both logged from the SIP module are not enough to blame it, given §2.
- **Consider a newer Dionaea image** (0.11.0 dates to November 2020) — tested
  separately, never swapped straight onto the live sensor.
- **Repo numbering:** two `SESSION-35-*` files now exist from parallel threads. Left as
  is (suffixes are unique and filenames are referenced), but a `Thread:` header line in
  each doc, as used here, keeps the threads scannable.
- Carried: `logrotate` for `/var/log/honeypot-alerts.log`; rotate the Telegram token
  before publishing any notes containing it.

---

## 9. Lessons

- **Detection without response is half a system.** 20-minute detection, 24-hour outage.
- **Noisy alerts are ignored alerts.** 73 messages for one condition trained the reader
  to skim past them.
- **Key suppression on the problem, not the prose.** Changing numbers inside alert text
  defeat string comparison.
- **A count that repeats across incidents can't encode something that varies between
  them.** 712 lines after 17.9 h and after 24.5 h of runtime measures a constant — here,
  a 48 KiB buffer.
- **Buffered log files aren't timelines.** Output that reaches disk in 4 KiB blocks can
  hide when things happened and how many times.
- **Check the proposed mechanism against every number already in hand.** `PIDS 5` was
  on screen during session 35 and contradicted thread accumulation. Nobody held the two
  side by side.
- **Three mechanisms, three retractions, one incident.** A SIP flood, a single noisy IP,
  and a timer backlog were each stated with confidence and each fell to data already
  available or one query away. When an explanation arrives fast and fits neatly, that's
  the moment to look for the number that breaks it.
- **Write down "unknown" when it's unknown.** An honest open root cause, with a
  watchdog that collects the right evidence next time, is worth more than a tidy wrong
  one.
