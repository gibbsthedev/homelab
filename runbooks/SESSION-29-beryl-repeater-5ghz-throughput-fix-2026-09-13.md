# SESSION 29 — Beryl Repeater Throughput: 15x Fix via 5GHz Band Switch

**Date:** 2026-09-13
**Device:** GL.iNet Beryl AX (GL-MT3000), `192.168.8.1`, firmware GL.iNet 4.8.1 /
OpenWrt 21.02-SNAPSHOT, kernel 5.4.211
**Continuity:** Follows `SESSION-28-spamhaus-audit-and-ssh-lockout-2026-09-12.md`.
Unrelated to the honeypot work — this addresses the homelab's long-standing network
bottleneck, previously logged only as "persistent source of latency/congestion."

**Result:** 10.39 / 13.12 Mb/s → **163.43 / 200.65 Mb/s**. Roughly 15x, both
directions. Upstream signal improved from **-74 dBm to -44 dBm**.

---

## 0. The symptom, quantified

Two speedtests, same laptop, same physical location, minutes apart:

| Connection | Ping | Down | Up |
|---|---|---|---|
| WhiteSky-Prose directly (building WiFi) | 14 ms | **303.74 Mb/s** | **365.94 Mb/s** |
| Through Beryl's WiFi (5GHz LAN) | 17 ms | **10.39 Mb/s** | **13.12 Mb/s** |

A ~30x drop. Far beyond normal repeater overhead — which typically costs something
closer to half throughput, not 97% of it. The building's connection was demonstrably
fine, so the loss had to be inside Beryl's repeater hop.

**Why this is structurally tricky to diagnose:** Beryl runs in **WiFi-as-WAN
(repeater) mode** — it is itself a WiFi *client* of `WhiteSky-Prose`, then
re-broadcasts its own WiFi for LAN devices. A client-side speedtest measures both
hops combined and cannot tell you which one is the bottleneck. The two hops have to
be measured separately.

---

## 1. Ruling out the LAN hop first

LuCI → Status → Overview showed both radios in Master (AP) mode with their clients:

```text
mt798111  802.11bgnax  Channel 1 (2.412 GHz)   SSID: FBI Surveillance Van #4
  64:5A:04:74:B4:94  mintstation.lan   -13 dBm   72.0 Mbit/s, 20 MHz, MCS 7

mt798112  802.11anacax Channel 48 (5.240 GHz)  SSID: FBI Surveillance Van #5
  A0:59:50:1B:1C:32  SchoolWork.lan    -25 dBm   1201.0 Mbit/s, 80 MHz, MCS 11, HE-MCS 11
```

The test laptop (`SchoolWork`) was associated at **-25 dBm, 1201 Mbit/s negotiated**
— an excellent link. **The LAN hop was conclusively not the problem.** Whatever was
wrong lived on the WAN side, between Beryl and WhiteSky.

Also noted in passing from the same page: the upstream IP is `100.110.148.162/26` —
inside the `100.64.0.0/10` carrier-grade NAT range, meaning WhiteSky itself sits
behind another NAT layer. Normal for building ISPs, not a throughput cause, but worth
recording as a known property of this network.

---

## 2. The interface the UI wouldn't show

The repeater's client-side connection does **not** appear in LuCI's Wireless
Overview — that page lists only the Master/AP interfaces. GL.iNet's firmware manages
the repeater client connection through its own layer, outside standard OpenWrt
wireless config, so no amount of clicking through LuCI surfaces it.

The Status page did reveal the interface name, buried in the Network section:
`Device: Ethernet Adapter: "apcli0"`. That was enough to query it directly:

```bash
ssh root@192.168.8.1
iwinfo apcli0 info
```

```text
apcli0    ESSID: "WhiteSky-Prose"
          Access Point: 04:CD:C0:9A:28:03
          Mode: Client  Channel: 1 (2.412 GHz) HT Mode: HE20
          Tx-Power: 20 dBm  Link Quality: 89/100
          Signal: -74 dBm  Noise: -40 dBm
          Bit Rate: 286.0 MBit/s
```

**Root cause, in one line: the repeater uplink was on 2.4GHz, channel 1, at -74 dBm.**

Reading those numbers properly:
- **-74 dBm is genuinely weak.** -50 dBm is excellent; -67 dBm is the usual floor for
  a reliable connection. At -74, constant retransmissions and packet loss crater
  *actual* throughput far below the negotiated rate.
- **"Bit Rate: 286.0 MBit/s" and "Link Quality: 89/100" are misleading here.** Those
  report the negotiated PHY rate and an instantaneous quality snapshot — not sustained
  real-world throughput. A link can negotiate 286 Mbit/s and still deliver 10.
- **Channel 1 is one of the three non-overlapping 2.4GHz channels (1/6/11)** that
  nearly every consumer device defaults to. In a multi-unit building, it is almost
  certainly shared with dozens of neighboring networks — heavy contention on top of
  an already-weak signal.

> **LESSON — a client-side speedtest cannot diagnose a repeater.** It measures both
> hops as one number. Isolating which hop is at fault required querying the WAN-side
> client interface directly, and that interface was invisible in both web UIs.
> `iwinfo <iface> info` over SSH was the only thing that actually answered the
> question.

> **LESSON — negotiated bitrate is not throughput.** 286 Mbit/s negotiated alongside
> 10 Mb/s measured is not a contradiction; it is what a weak, congested link looks
> like. Read `Signal:` first, not `Bit Rate:`.

---

## 3. The fix

GL.iNet admin UI → **INTERNET** (top sidebar item) → the `WhiteSky-Prose` connection
card → **gear icon** → **Repeater Options**.

Two settings there:
- *Allow Switching to Other Networks Mode*: `Auto Switching`
- **Band Selection: `Auto`** ← the relevant one

`Auto` had settled on 2.4GHz. Changed to **5GHz**.

**Navigation note for next time:** clicking the "Repeater" icon in the dashboard's
topology diagram lands on Network → Multi-WAN, *not* the repeater settings. The
actual path is the sidebar's top-level **INTERNET** item, then the gear on the
connection card.

---

## 4. The DFS trap — a success that looked exactly like a failure

After applying 5GHz, the UI showed **"Connecting…"** and then **timed out**. The
reasonable read was "no 5GHz WhiteSky signal reaches this spot." That read was
**wrong**, and acting on it caused the rest of this section.

**What actually happened:** the connection landed on **channel 108 (5.540 GHz), a DFS
channel.** DFS (Dynamic Frequency Selection) bands are shared with weather and
military radar, so regulation requires a device to listen silently for 60+ seconds
before transmitting, confirming no radar is present. **The web UI's timeout is
shorter than that mandatory quiet period.** The connection was succeeding the entire
time; the UI simply gave up before DFS finished.

### What the wrong conclusion cost

Believing it had failed led to: attempting to abort (button unresponsive — the UI
backend was busy with the pending connection), attempting SSH (also unresponsive
during the transition), and finally a physical power cycle. Post-reboot, the laptop
auto-rejoined `WhiteSky-Prose` directly instead of Beryl's SSID, so `ping 192.168.8.1`
failed and it briefly appeared the router was dead — when in fact `ipconfig` clearly
showed the laptop sitting on `4107.wtsky.net` at `100.110.148.172`, a completely
different network. Nothing was broken at any point.

> **LESSON — on a DFS channel, wait a full two minutes before concluding a WiFi
> connection failed.** The device is legally required to sit silent and listen for
> radar. A UI timeout during that window means nothing.

> **LESSON — when a gateway "goes down," check which network the client is actually
> on before assuming the gateway is at fault.** `ipconfig`'s DNS suffix and default
> gateway answer this instantly. A client that silently failed over to a different
> SSID looks identical to a dead router from the ping's perspective.

---

## 5. The second gotcha: the interface name changes with the band

Post-switch, checking the old interface returned nonsense:

```bash
iwinfo apcli0 info
```

```text
apcli0    ESSID: unknown
          Access Point: 00:00:00:00:00:00
          Signal: -256 dBm  Noise: -47 dBm
```

`-256 dBm` is a null placeholder, not a reading. The cause: **2.4GHz and 5GHz are
separate physical radios (`mt798111` / `mt798112`), so each has its own client
interface.** `apcli0` is the 2.4GHz client — now dead and stale. The live one is
`apclix0`.

```bash
ip link show | grep -i apcli
```

```text
9:  apcli0:  <NO-CARRIER,BROADCAST,MULTICAST,UP> state DOWN
10: apclix0: <BROADCAST,MULTICAST,UP,LOWER_UP>   state UP
```

`ip link show` is the reliable way to find which client interface is actually live —
`LOWER_UP` is the one carrying traffic.

```bash
iwinfo apclix0 info
```

```text
apclix0   ESSID: "WhiteSky-Prose"
          Access Point: C8:78:67:F9:66:B3
          Mode: Client  Channel: 108 (5.540 GHz) HT Mode: HE40
          Tx-Power: 20 dBm  Link Quality: 100/100
          Signal: -44 dBm  Noise: -47 dBm
          Bit Rate: 573.0 MBit/s
```

Note the BSSID also changed (`04:CD:C0:9A:28:03` → `C8:78:67:F9:66:B3`) — a genuinely
different radio on WhiteSky's infrastructure, not just a different band on the same AP.

---

## 6. Results

| Metric | Before (2.4GHz) | After (5GHz) |
|---|---|---|
| Interface | `apcli0` | `apclix0` |
| Channel | 1 (2.412 GHz) | 108 (5.540 GHz, DFS) |
| **Signal** | **-74 dBm** | **-44 dBm** |
| Link Quality | 89/100 | 100/100 |
| Negotiated Bit Rate | 286.0 Mbit/s | 573.0 Mbit/s |

**-74 → -44 dBm is a 30 dB improvement. Because dBm is logarithmic, that is roughly
a 1000x stronger received signal.**

Measured throughput through Beryl:

| Test | Ping | Down | Up |
|---|---|---|---|
| Before | 17 ms | 10.39 Mb/s | 13.12 Mb/s |
| After (run 1) | 14 ms | 154.01 Mb/s | 37.73 Mb/s |
| After (run 2) | 15 ms | 163.43 Mb/s | 200.65 Mb/s |

The 37.73 Mb/s upload on run 1 was an outlier — run 2 settled at 200.65. **Re-running
before chasing an anomaly was the right call**; a single speedtest result is not a
measurement.

The remaining gap versus WhiteSky-direct (304/366) is expected repeater overhead:
Beryl splits radio time between receiving upstream and broadcasting to LAN clients.
Not worth chasing.

---

## 7. Open items / things to watch

- **DFS stability.** Channel 108 is a DFS channel. If radar is detected nearby, the
  link is required to vacate and rescan, causing dropouts that would not occur on a
  non-DFS channel. If intermittent disconnects appear over the coming days, this is
  the first suspect — and worth knowing before the honeypot workflow leans on this
  link. A non-DFS 5GHz channel would trade some of this gain for stability if needed.
- **Band Selection is still set to a specific band, not `Auto`.** Worth confirming it
  survives a reboot and that `Auto` does not silently drag it back to 2.4GHz.
- **Upload variance.** Run 1 vs run 2 differed by 5x on upload. Worth a few more
  samples over time to establish what is actually typical.
- This resolves the "persistent source of latency/congestion" noted against Beryl in
  the project overview — that entry should be updated.

---

## 8. Lessons (consolidated)

- A client-side speedtest measures a repeater's two hops as one number and cannot
  isolate either. Query the WAN-side client interface directly.
- GL.iNet's repeater client interface is invisible in both LuCI and the native UI's
  status pages. `iwinfo <iface> info` over SSH is the way to see it.
- Negotiated bit rate is not throughput. `Signal:` is the number that predicts real
  performance.
- On DFS channels, a connection attempt can take 60+ seconds of mandatory silent
  radar listening — longer than the UI's own timeout. Wait two minutes before
  concluding failure.
- When the gateway seems dead, verify which network the client is actually attached
  to first. A silent failover to another SSID is indistinguishable from a dead router
  by ping alone.
- 2.4GHz and 5GHz client interfaces are separate devices (`apcli0` / `apclix0`).
  After a band switch, the old one reports stale garbage. `ip link show` finds the
  live one.
- Re-run a speedtest before chasing an anomalous result.
