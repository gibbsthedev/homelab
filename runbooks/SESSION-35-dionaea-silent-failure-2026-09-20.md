# SESSION 35 — The Four-Day Silent Failure: Dionaea Runaway and Heartbeat Monitoring

**Date:** 2026-09-20
**Box:** Hetzner Cloud VPS `ubuntu-4gb-hel1-2` (62.238.47.215)
**Continuity:** Follows `SESSION-33-alerting-and-incident-2026-09-16.md` for the
honeypot VPS thread. `SESSION-34-acls-and-wall-jacks-2026-09-20.md` is the SG300
switch thread, which runs in parallel and shares the date but not the subject.

**Summary.** Dionaea stopped capturing on 2026-09-16 at 13:36:53 UTC and was not
noticed until 2026-09-20 — **four days of lost sensor coverage**. Docker reported the
container healthy the entire time. The alerting built in session 33 could not detect
it by design. Root cause traced to sustained SIP volume exhausting per-call timer
handling. A heartbeat monitor was built and tested to close the gap.

---

## 1. The gap that made this possible

Session 33 built `check.py`, which reports **things never seen before** — new payload
URLs, new hashes, new uploads. It ran hourly for four days and stayed silent.

That silence was correct behaviour and completely useless:

```text
Dionaea frozen  →  no new hashes  →  nothing to report  →  no alert
Dionaea healthy →  no new hashes  →  nothing to report  →  no alert
```

**A change detector cannot distinguish "nothing new happened" from "the sensor is
dead."** Both produce the same input: absence.

Worse, the timeline shows the baseline itself was contaminated. Dionaea stopped at
**13:36:53** on Sept 16; the `check.py` baseline was established that evening around
**19:00**:

```text
Baseline established: 49 urls, 76 hashes, 50 uploads, 81 dio_hashes, 624 droppers
```

Those `81 dio_hashes` were snapshotted from an already-dead sensor. And the session-33
verification sweep recorded `dionaea: Up 24 hours` as evidence of health — the exact
false signal this incident is about.

> **LESSON — detectors that alert on presence cannot detect absence.** Every
> monitoring system needs both: one that fires when something new appears, one that
> fires when something expected *stops* appearing.

---

## 2. Detection

The discrepancy surfaced from a database timestamp, not from any alert:

```bash
stat /opt/dionaea/var/lib/dionaea/dionaea.sqlite
```

```text
Modify: 2026-09-16 13:36:53.304952302 +0000
```

Four days stale. Meanwhile:

```bash
docker ps --format 'table {{.Names}}\t{{.Status}}'
```

```text
dionaea   Up 4 days
```

**Both statements were true.** The container process existed continuously. The
application behind it had stopped working.

### Ruling out the boring explanation first

Before concluding the application had failed, the possibility that analysis was simply
reading a stale host-side file while the container wrote elsewhere had to be
eliminated:

```bash
docker inspect dionaea --format '{{json .Mounts}}' | python3 -m json.tool
```

```json
[
  {"Type":"bind","Source":"/opt/dionaea/etc","Destination":"/opt/dionaea/etc","RW":true},
  {"Type":"bind","Source":"/opt/dionaea/var/lib","Destination":"/opt/dionaea/var/lib","RW":true},
  {"Type":"bind","Source":"/opt/dionaea/var/log","Destination":"/opt/dionaea/var/log","RW":true}
]
```

Bind mounts correct, paths identical inside and out. The container and the host were
looking at the same file. The database genuinely had not been written in four days.

---

## 3. Evidence from the failed process

### Resource state

```bash
docker stats --no-stream dionaea
```

```text
CPU %      MEM USAGE / LIMIT     PIDS
100.52%    107.1MiB / 3.725GiB   5
```

**One full CPU core, pinned, for four days.** Memory flat at 107 MB — no leak.

### Docker's view

```text
Status=running  Running=true  Restarting=false
OOMKilled=false ExitCode=0    RestartCount=0
```

Not crashed. Not OOM-killed. Not restarted. Docker had no reason to intervene, because
by every signal available to it, nothing was wrong.

### Thread state — the most informative single artifact

```bash
ps -Lp 34962 -o pid,tid,ppid,pcpu,pmem,stat,wchan:32,comm
```

```text
    PID     TID  %CPU STAT WCHAN              COMMAND
  34962   34962  84.1  Rsl  -                 dionaea      <- main thread
  34962   35309   0.0  Ssl  futex_do_wait     dionaea
  34962   35310   0.0  Ssl  futex_do_wait     pool
  34962   35311   0.0  Ssl  futex_do_wait     pool
```

```bash
cat /proc/34962/wchan   # → 0
cat /proc/34962/stack   # → empty
```

Reading this precisely:

- **Main thread `Rsl`** — running/runnable, actively executing
- **Empty `wchan`, empty kernel stack** — not blocked in any kernel call
- **Worker threads `futex_do_wait`** — sleeping normally, waiting for work that never came

This rules out the common failure modes: no disk I/O wait, no network block, no worker
pool saturation, no classic futex deadlock. **The main thread was spinning in
userspace.**

### The log signature

```bash
grep -oE '[a-zA-Z0-9_/.-]+:[0-9]+-warning:.*' dionaea-errors.log | sort | uniq -c | sort -rn
```

```text
711 /dionaea/sip/__init__.py:45-warning: Cleanup
  1 /sip/__init__.py:45-warning: Cleanup
```

**711 identical log lines, all timestamped `13:36:53` — the same second.** No other
error type. No messages before or after.

---

## 4. Reading the source

The immediate suspicion was that the cleanup function itself was looping. It is not.
Located at `/opt/dionaea/lib/dionaea/python/dionaea/sip/__init__.py`:

```python
44  def cleanup(*args):
45      logger.warning("Cleanup")
46
47      # remove closed calls
48      for key in list(g_call_ids.keys()):
49          if g_call_ids[key] is None:
50              del g_call_ids[key]
```

Seven lines. It iterates a `list()` snapshot of a dict — a fixed-size sequence — and
deletes `None` values. **It cannot loop forever.** It terminates on a bounded
iteration by construction.

Scheduling, at line 79:

```python
75  if len(daemons) > 0:
76      global g_timer_cleanup
77      if g_timer_cleanup is None:
78          logger.warning("Starting cleanup loop")
79          g_timer_cleanup = Timer(
80              60,
81              cleanup,
82              repeat=True,
83          )
84          g_timer_cleanup.start()
```

A single global timer, fired once per minute, guarded against duplicate creation.

**So 711 fires in one second is ~11.8 hours of scheduled work compressed into a
second.** The cleanup function is not the cause — it is the *symptom*. Something
starved the event loop of its ability to run timers on schedule, and when it briefly
recovered, the entire backlog flushed at once.

> **LESSON — read the code before blaming it.** The loudest thing in the log was a
> seven-line function that provably cannot loop. The 711 repetitions were evidence of
> timer starvation, not of a bug in the function being logged.

### The mechanism

Line 271-277 of the same file, inside the `SipCall` class:

```python
271  # Global timers
272  self._timers = {
273      "idle": Timer(60.0, self.__handle_timeout_idle, repeat=True),
274      "invite_handler": None,
275  }
276
277  self._timers["idle"].start()
```

**Every `SipCall` object creates its own repeating 60-second timer.** And from
`dionaea/__init__.py:56`:

```python
class SubTimer(Thread):
```

These timers are **threads**. Each concurrent SIP call adds a thread to the process.

Dionaea 0.11.0 imposes no cap on concurrent `SipCall` objects.

---

## 5. Root cause — and a correction

### The wrong answer, recorded because the reasoning error matters

The connection immediately preceding the freeze was a `SipCall` from
`172.110.223.179` at 13:36:38 — 15 seconds before. Its SIP command breakdown looked
dramatic:

```text
INVITE | 3CXPhone 16.0.1.81           | 11
INVITE | Asterisk PBX 18.9.0          | 14
INVITE | Avaya One-X Deskphone/6.6.5  |  9
…30 distinct spoofed user-agents
```

This was read as a single burst of ~350 INVITEs overwhelming the timer system.

**That was wrong**, and the data said so immediately:

```text
172.110.223.179:  358 commands | 283 calls | connects every ~27 minutes since Sept 8
```

Those 358 commands are **cumulative across 8 days and ~200 sessions** — roughly 1.8
commands per session. It is a metronome-regular scanner that has connected before and
after the incident, including seven times since the restart, without causing any
problem. It ranks **10th** by SIP volume.

The error was pattern-matching an unusual-looking IP near the timestamp and building a
narrative around it. **The third time in this project that same mistake has appeared**
— after the `147.185.132.x` range (Palo Alto Networks, session 32) and the "AF_ALG
exploit family" (compiled-binary substring noise, session 32).

### The supportable answer

```bash
sqlite3 dionaea.sqlite "select count(*), min(...), max(...) from connections
                        where remote_host='82.165.65.238';"
```

```text
5,970 connections | 2026-09-15 17:16:04 → 2026-09-16 13:36:33
```

**`82.165.65.238`: 5,970 SIP connections over 20 hours, the last one 20 seconds before
the freeze.** Sustained, not bursty:

```text
09-15 17:00 | 302      09-16 03:00 | 202
09-15 18:00 | 390      09-16 06:00 | 188
09-15 19:00 | 416      09-16 08:00 | 160
09-15 20:00 | 425      09-16 09:00 | 406
09-15 21:00 | 438      09-16 10:00 | 412
09-15 22:00 | 431      09-16 11:00 | 252
09-15 23:00 | 363      09-16 12:00 | 205
09-16 00:00 | 337      09-16 13:00 | 116  <- freeze at 13:36
```

300–440/hour for eight hours, a dip overnight to ~180, back to 400+ by morning. One
connection every ~12 seconds, continuously, for 20 hours.

Daily totals confirm this was abnormal:

```text
date        total    SIP     SIP %
2026-09-13   3,147    455     14%
2026-09-14   3,962    648     16%
2026-09-15  12,202  3,937     32%    <- 6x jump
2026-09-16   8,208  5,527     67%    <- in 13.6 hours, before the freeze
```

SIP went from ~14% of all traffic to 67%.

**Mechanism:** sustained SIP connection volume created `SipCall` objects faster than
they were reclaimed, each spawning a repeating timer thread. Over 20 hours this
accumulated until the single-threaded event loop could no longer service its timer
queue. The main thread saturated one core; cleanup callbacks backed up ~11.8 hours
deep; when the loop briefly caught up, 711 queued fires flushed in one second — and
then it never recovered.

**This is not a targeted DoS.** `82.165.65.238` is a VoIP toll-fraud scanner of the
kind documented in session 32. It was doing ordinary work at a rate a 2020-vintage
honeypot cannot absorb. It has sent **zero connections since the restart** — a sweep
that finished, not an ongoing attack.

### What remains hypothesis

```text
PROVEN:     DB stopped at 13:36:53; container stayed up; not OOM-killed;
            main thread pinned runnable in userspace; workers idle;
            711 cleanup fires in one second; restart fully restored function;
            82.165.65.238 sent 5,970 SIP connections in the preceding 20 hours
            and stopped 20 seconds before the freeze.

NOT PROVEN: that a specific packet triggered it; that timer-thread exhaustion is
            the precise failure rather than a related event-loop defect;
            that the same volume would reproduce it.
```

Reproducing it deliberately would confirm the mechanism, but requires a separate
instance — not something to test on a live sensor.

---

## 6. Recovery

Evidence preserved before touching anything:

```bash
STAMP=$(date +%Y%m%d-%H%M%S)
DIR="/root/dionaea-postmortem-$STAMP"
mkdir -p "$DIR"
cp /opt/dionaea/var/lib/dionaea/dionaea.sqlite "$DIR/"
cp /opt/dionaea/var/log/dionaea/dionaea.log "$DIR/"
cp /opt/dionaea/var/log/dionaea/dionaea-errors.log "$DIR/"
docker inspect dionaea > "$DIR/docker-inspect.json"
docker stats --no-stream dionaea > "$DIR/docker-stats.txt"
docker top dionaea > "$DIR/docker-top.txt"
ps -Lp 34962 -o pid,tid,ppid,pcpu,pmem,stat,wchan:32,comm > "$DIR/threads.txt"
cat /proc/34962/wchan > "$DIR/main-wchan.txt" 2>&1
cat /proc/34962/stack > "$DIR/main-kernel-stack.txt" 2>&1
```

Preserved at `/root/dionaea-postmortem-20260920-120110`. **Keep until root-cause work
is finished** — the failed process cannot be recreated.

```bash
docker restart dionaea
```

Within 12 seconds:

```text
CPU %   MEM USAGE
0.01%   34.3MiB
```

From 100.52% / 107 MB to 0.01% / 34 MB. That alone is strong evidence of a runaway
state, but a restart is not proof of function.

### Verification by function, not by status

```bash
timeout 3 nc -v 127.0.0.1 21 < /dev/null       # → 220 DiskStation FTP server ready.
curl -k -I --max-time 3 https://127.0.0.1:443/ # → HTTP/1.1 200 OK, Server: nginx/1.18.0
timeout 3 nc -v 127.0.0.1 3306 < /dev/null     # → 5.7.16 banner
timeout 3 nc -v 127.0.0.1 1433 < /dev/null     # → connected
timeout 3 nc -v 127.0.0.1 445 < /dev/null      # → connected
```

Then the database:

```text
2026-09-20 12:03:44 | smbd       | 172.17.0.1
2026-09-20 12:03:41 | mssqld     | 172.17.0.1
2026-09-20 12:03:38 | mysqld     | 172.17.0.1
2026-09-20 12:03:38 | httpd      | 172.17.0.1
2026-09-20 12:03:34 | ftpd       | 172.17.0.1
2026-09-20 12:03:28 | SipSession | 156.225.1.10     <- REAL external traffic
```

**The SIP event at 12:03:28 arrived six seconds before the first local test** — from a
real internet source. That proves the full public path, not just the loopback one:

```text
Internet → UFW → Docker published port → bridge → Dionaea → protocol handler → SQLite
```

And cleanup is firing correctly now:

```text
[20092026 15:16:30] sip .../__init__.py:45-warning: Cleanup
[20092026 15:17:30] sip .../__init__.py:45-warning: Cleanup
[20092026 15:18:30] sip .../__init__.py:45-warning: Cleanup
```

Exactly once per minute, as the 60-second timer intends. This also confirms
**711-in-one-second was genuinely anomalous** rather than a logging artifact — a
possibility worth ruling out, since both preserved log copies happened to be exactly
49,152 bytes, which initially suggested truncation.

---

## 7. The fix: heartbeat monitoring

`check.py` alerts on presence. `heartbeat.py` alerts on **absence and abnormality** —
the inverse failure mode.

Four independent signals:

```python
STALE_HOURS = 3      # dionaea sees traffic constantly; 3h silent is abnormal
CPU_PCT     = 50.0   # sustained high CPU on an idle honeypot is a runaway

# 1. Dionaea DB freshness — would have caught this in ~3 hours
last = db.execute("select max(connection_timestamp) from connections").fetchone()[0]
if (time.time() - last) / 3600 > STALE_HOURS:
    alerts.append(f"DIONAEA STALE: no connections for {age_h:.1f}h …")

# 2. Container CPU — would have caught this in 20 minutes
cpu = float(subprocess.run(["docker","stats","--no-stream",
                            "--format","{{.CPUPerc}}","dionaea"], …).stdout.rstrip("%"))
if cpu > CPU_PCT:
    alerts.append(f"DIONAEA CPU RUNAWAY: {cpu}%")

# 3. Cowrie log freshness
# 4. systemctl is-active for all four services
```

**State-change suppression** prevents hourly repetition of a known-bad condition, and
sends a recovery notice when it clears:

```python
prev = json.load(open(STATE)) if os.path.exists(STATE) else []
json.dump(alerts, open(STATE,"w"))
if alerts and alerts != prev:
    tg("HONEYPOT HEALTH ALERT\n\n" + "\n".join(alerts))
elif prev and not alerts:
    tg("HONEYPOT RECOVERED: all checks passing")
```

Telegram credentials are read from `cowrie.cfg`, same as `notify.py` — one secret, one
location.

### Tested, not assumed

A monitor that has never fired is unproven. Forced by temporarily lowering the
threshold:

```bash
sed -i 's/^STALE_HOURS = 3/STALE_HOURS = 0/' /opt/honeypot-alerts/heartbeat.py
python3 /opt/honeypot-alerts/heartbeat.py
sed -i 's/^STALE_HOURS = 0/STALE_HOURS = 3/' /opt/honeypot-alerts/heartbeat.py
python3 /opt/honeypot-alerts/heartbeat.py
```

Delivered to Telegram:

```text
HONEYPOT HEALTH ALERT

DIONAEA STALE: no connections for 0.1h (last 2026-09-20 14:46 UTC)
COWRIE STALE: log untouched 0.0h
```

Then clean on restore, confirming both the alert path and the state-change logic.

### Scheduled

```bash
cat > /etc/cron.d/honeypot-alerts << 'EOF'
SHELL=/bin/bash
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
MAILTO=""
0   * * * * root /usr/bin/python3 /opt/honeypot-alerts/notify.py    >> /var/log/honeypot-alerts.log 2>&1
*/20 * * * * root /usr/bin/python3 /opt/honeypot-alerts/heartbeat.py >> /var/log/honeypot-alerts.log 2>&1
EOF
systemctl restart cron
```

**Detection time for this exact failure: 20 minutes instead of four days** — the CPU
check would fire on the first run after the freeze.

---

## 8. Decision: what to do about SIP

The conditions will recur. `82.165.65.238` finished its sweep, but another will come.

| Option | Effect | Cost |
|---|---|---|
| **A — accept, monitor** | Heartbeat catches a freeze in ≤20 min | Manual restart when it happens |
| **B — disable SIP** | Eliminates the failure mode | Loses ~5,500 connections/day and the toll-fraud data from session 32 |
| **C — auto-restart** | Self-healing on stale-DB + high-CPU together | Automated action on a condition seen once |

**Chosen: A, with C held in reserve.** One occurrence in three weeks does not justify
automated restarts, and the SIP data has been genuinely productive — the recovered
toll-fraud destination numbers were among the better findings in session 32. If it
recurs, C becomes justified.

---

## 9. Files changed

| Path | Change |
|---|---|
| `/opt/honeypot-alerts/heartbeat.py` | **NEW** — absence/abnormality detector, tested |
| `/opt/honeypot-alerts/heartbeat_state.json` | **NEW** — state-change suppression |
| `/etc/cron.d/honeypot-alerts` | Added 20-minute heartbeat alongside the hourly change detector |
| `/root/dionaea-postmortem-20260920-120110/` | **NEW** — preserved failed-state evidence |

---

## 10. Open items

- **Verify the timer-exhaustion mechanism** on a separate Dionaea instance rather than
  the live sensor. Until then it is the best-supported hypothesis, not proven.
- **Dionaea 0.11.0 is from November 2020.** A newer image may cap concurrent SipCall
  objects. Do not upgrade until the postmortem evidence is fully understood and the new
  image can be tested separately.
- **Four days of Dionaea data are simply gone** (2026-09-16 13:36 → 2026-09-20 12:01).
  Any analysis spanning that window must account for the hole.
- The session-33 `check.py` baseline was taken from a partly-dead sensor. Its
  `dio_hashes` count is not a clean historical record.
- Add `logrotate` for `/var/log/honeypot-alerts.log` (carried from session 33).
- Consider extending the heartbeat to the other sensors' databases, not just Cowrie's
  log mtime.

---

## 11. Lessons

- **`docker ps` is not a health check.** It reports whether a process exists. Dionaea's
  main process existed, held its ports, and did nothing useful for four days.
- **Detectors that alert on presence cannot detect absence.** Monitoring needs both
  directions: something new appeared, and something expected stopped appearing.
- **Read the code before blaming it.** The 711-times-repeated log line came from a
  seven-line function that provably cannot loop. Volume in a log indicates where to
  look, not what is broken.
- **A backlog flush looks like a storm.** 711 fires of a 60-second timer in one second
  is ~11.8 hours of deferred work, not a runaway function.
- **Thread state is diagnostic.** `Rsl` with an empty `wchan` and empty kernel stack
  says "spinning in userspace" and rules out I/O, network, and lock contention in one
  command.
- **The IP nearest the timestamp is not automatically the cause.** `172.110.223.179`
  looked guilty and was a regular visitor that still connects harmlessly today. The
  real source was 16x larger by volume and ranked first, not tenth.
- **Cumulative totals are not burst sizes.** 358 commands across 8 days and ~200
  sessions reads very differently from 358 in one session. Always bound a count by time.
- **Preserve before repairing.** The failed process cannot be recreated; thread state,
  CPU, and Docker metadata were only available before the restart.
- **Verify by function, not by status.** FTP banners, HTTP responses, a MySQL
  handshake, and a real external SIP connection proved recovery. "Container up" had
  already proved nothing for four days.
