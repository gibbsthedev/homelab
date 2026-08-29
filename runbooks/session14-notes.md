# Session 14 — Log Analysis Training: From grep Basics to Real Correlation

Date: 2026-08-23 into 2026-08-24

This is the core SOC-analyst training session — a deliberately staged arc
from basic log filtering up to genuine multi-indicator correlation, worked
directly against the honeypot's real, live attacker data rather than
practice logs.

---

## PART 1 — THE MENTAL MODEL

Log analysis, almost all of it, reduces to three moves chained with the pipe
(`|`): **filter → extract → count/sort.**

| Move | Tool |
|---|---|
| Filter to matching lines | `grep PATTERN` |
| Extract just the matched part | `grep -o PATTERN` |
| Invert (show non-matches) | `grep -v PATTERN` |
| Rank by frequency | `sort \| uniq -c \| sort -rn` |
| Unique values, no count | `sort -u` |
| Grab one column | `cut -d' ' -f1` or `awk -F'sep' '{print $N}'` |
| Find/replace streamed text | `sed 's/find/replace/'` |
| Feed results into another command | `xargs -I{} cmd {}` |

`sort | uniq -c | sort -rn` — group, count, rank — is the single idiom behind
almost every "what are the most common X" question a SOC analyst asks.

## PART 2 — A REAL BUG, CAUGHT LIVE

An early "activity by hour" command used `cut -c14-15` (a fixed character
position) on `grep -o` output — but `grep -o` includes the matched field
*name*, not just the value, so a fixed position landed inside the year, not
the hour. Fixed by matching the actual *shape* of the timestamp
(`T[0-9]{2}:`) instead of counting characters. **Generalized lesson: match
shape, don't count positions** — anything that assumes a fixed offset breaks
the moment the surrounding text changes length.

## PART 3 — SESSION RECONSTRUCTION: THE "ZOOM IN" MOVE

Every Cowrie event carries a `session` field — a short hex ID unique to one
connection. Filtering on it turns a flat pile of log lines into one
attacker's complete, chronologically-ordered story, with no sorting needed
since the log is already written in event order:
```bash
grep '"session":"4fd8beaebb66"' cowrie.json
```
This complements the "zoom out" ranking commands (most common X across
everyone) — reconstruction is "zoom in" on one thread to the end.

**A real caution surfaced from actual data:** the top result of a "most
active session" ranking turned out to be a personal test session, not a real
attacker. **Rule adopted: always verify the source IP behind a ranking
before trusting it.**

## PART 4 — CORRELATION: PROVING TWO INDICATORS POINT AT ONE CAMPAIGN

The reusable pattern, `comm -12` (set intersection of two sorted lists):
```bash
comm -12 <(grep 'INDICATOR_1' cowrie.json | grep -oE '"src_ip":"[^"]+"' | sort -u) \
         <(grep 'INDICATOR_2' cowrie.json | grep -oE '"src_ip":"[^"]+"' | sort -u)
```

**Applied to a real finding — an SMTP-relay botnet cluster:** noticed
several SSH sessions from different IPs all immediately issuing a
`direct-tcp connection request to 77.88.21.158:25` (port 25 = SMTP, a
spam-relay pivot) after login. Two independent indicators tied them
together: the shared relay target IP, and a shared `hassh` SSH-client
fingerprint (`acaa53e0a7d7ac7d1255103f37901306`) — meaning the same client
tooling regardless of source IP. Running the intersection confirmed a
**4-IP cluster** sharing both signals — genuine proof of one coordinated
campaign, not four coincidences. The cluster stayed active across multiple
days, with new IPs joining under the same fingerprint and target.

## PART 5 — awk, AND A MESSY REAL-WORLD EXERCISE

`awk` introduced as a step up from `grep` — column-oriented rather than
line-oriented (`-F` for custom separators, `$N` for a specific field, `$NF`
for "whichever field is last," useful when field count varies).

Applied to ranking attacker countries via `whois` lookups, which surfaced a
genuinely messy real-world problem: regional internet registries (ARIN,
RIPE, APNIC) use different field labels and casing for the same data.
Solved by chaining `while read` (more flexible than `xargs` here),
`grep -im1` (first-match-only, preventing double-counting), `awk
'{print $NF}'` (label-agnostic value extraction), and `tr 'a-z' 'A-Z'`
(case normalization) — each piece chosen for the specific messiness it
fixed, not applied by habit.

## PART 6 — TELLING HUMANS FROM BOTS (harder than it sounds)

First-pass attempt filtered on command vocabulary alone (`cat`, `nano`,
`ls -la`) — and leaked bots straight into the "likely human" bucket, since
botnet scripts also run `cat /proc/cpuinfo` and `cd /tmp`. **Command
vocabulary alone isn't identity.**

The doctrine that actually held up, keyed to **intent-shaped behavior**, not
just surface commands:
- **Persistence attempts** (`authorized_keys` planting, `chattr`/`lockr`
  locking) — no legitimate reason for someone poking around to backdoor
  anything. Unambiguous attacker signal.
- **Payload downloads** (`wget`/`curl` pulling and executing a binary) —
  same logic.
- **Credential harvesting at scale** vs. one or two attempts then giving up.
- **Exfiltration-shaped reads vs. curiosity-shaped reads** — reading a
  honeytoken and moving on reads differently than reading it and piping it
  out or referencing its contents in follow-up commands.
- **Leaving things working (`apt update`) vs. trying to break/take things.**

One live case this mattered for: two sessions that looked exploratory rather
than malicious were tentatively traced to a public Reddit thread that had
disclosed the honeypot's IP — likely curious readers trying a posted
password, not attackers demonstrating tradecraft. Treated as a useful
control group *only* once verified against the persistence/payload
checklist above, not assumed automatically.

## PART 7 — A SECOND MAJOR FINDING: THE CRYPTO-TARGETED CREDENTIAL CLUSTER

A new heavy-hitter IP surfaced (810 overnight hits). Behavior: one command
per session, always identical — `/bin/./uname -s -v -n -r -m`, pure
OS-fingerprinting, no further exploration.

**The real find was in the username wordlist, not the behavior.** Attempted
usernames included `sol`, `solana`, `eth`, `ethdocker`, `blockchain`, and —
the unambiguous tell — **`firedancer`**, a real, fairly obscure Solana
validator client name no generic scanner would guess by coincidence. This is
a deliberately built target list aimed at cryptocurrency node operators, not
random dictionary spraying — the kind of specific, hard-to-fake detail that
separates a documented finding from a guess.

## PART 8 — IOC COLLECTION

Two distinct IoT botnet lineages confirmed via signature commands, both
telnet-side:
- **Mirai family:** `enable → linuxshell → system → shell → sh →
  /bin/busybox UNSTABLE` — the busybox call with a random applet name is
  Mirai's device-compatibility fingerprint check.
- **Gafgyt/Bashlite family:** a hex-encoded `echo` spelling out `"GAYFGT"` —
  a known hardcoded marker some botnets use to check whether a target is
  already infected, occasionally to avoid or evict rival botnets from the
  same device. Distinct lineage from Mirai despite similar IoT targeting.

## LESSONS

- **Filter → extract → count/sort, chained with `|`, covers almost every
  question a SOC analyst asks of a log.**
- **Match shape, not fixed character positions** — offsets break the moment
  surrounding text changes length.
- **Always verify the IP behind a "most active" ranking** — your own test
  traffic can outrank real attackers.
- **Two independent, provably-shared indicators (not one) turn a hunch into
  a documented cluster** — `comm -12` is the reusable tool for that.
- **Command vocabulary alone doesn't distinguish human from bot** — intent-
  shaped behavior (persistence, payload delivery, exfiltration-shaped reads,
  scale) does.
- **A specific, hard-to-fake detail in a target list (like `firedancer`) is
  stronger evidence than volume alone** — it's what makes a finding
  defensible rather than speculative.

---
END. This is the strongest single portfolio piece so far — real correlation
work, a defensible original finding, and a documented, reusable methodology.
