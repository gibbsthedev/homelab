# WordPress Honeypot — Log Analysis Commands

A focused reference for reading the HTTP honeypot's request log,
`/var/log/wp-honeypot/http.jsonl`. Companion to Part 3 section 7 (the fake
WordPress + nginx setup); this file is the day-to-day "what hit the site"
toolkit.

Same style as the other command references: each block is followed by what it
does and WHY you'd reach for it. All of this runs on the **real** VPS (port
2200) — the `http.jsonl` path does not exist inside Cowrie.

Every request the honeypot receives is logged as one JSON object per line
(append-only). The fields used below: `ts` (UTC timestamp), `ip` (real client,
from nginx's `X-Real-IP`), `method`, `path`, `body`, and `headers`.

---

## 1 — THE TABLE PRINTER (`wp-posts`)

The raw JSONL is hard to skim. This installs a small script that prints POST
requests as an aligned table, skips your own test IP, and labels the common
Nmap scan shapes so real login attempts stand out from scanner noise.

```
cat > /usr/local/bin/wp-posts << 'EOF'
#!/usr/bin/env python3
import json
from pathlib import Path
from urllib.parse import unquote_plus

LOG = Path("/var/log/wp-honeypot/http.jsonl")
SKIP_IP = "104.241.55.33"

print(f"{'TIME UTC':<20} {'IP':<16} {'PATH':<16} BODY / WHAT")
print("-" * 88)
for line in LOG.read_text().splitlines():
    if not line.strip():
        continue
    d = json.loads(line)
    if d.get("ip") == SKIP_IP or d.get("method") != "POST":
        continue
    body = unquote_plus(d.get("body") or "")
    ua = (d.get("headers") or {}).get("User-Agent", "")
    if "Nmap" in ua:
        extra = "Nmap"
        if "listMethods" in body:
            extra = "Nmap xmlrpc listMethods"
        elif "RetrieveServiceContent" in body or d.get("path") == "/sdk":
            extra = "Nmap VMware/SOAP /sdk"
        elif not body:
            extra = "Nmap empty POST"
    else:
        extra = body[:80] or "(empty)"
    ts = d.get("ts", "")[:19].replace("T", " ")
    print(f"{ts:<20} {d.get('ip', ''):<16} {d.get('path', ''):<16} {extra}")
EOF
chmod +x /usr/local/bin/wp-posts
```
- `cat > /usr/local/bin/wp-posts << 'EOF' ... EOF` is a heredoc that writes the
  whole script to a file in one paste. The quoted `'EOF'` means no shell
  expansion — `$`, backticks, and the Python `{}` format braces all land in
  the file literally instead of being interpreted by bash first.
- WHY `/usr/local/bin/`: it's on the default `$PATH`, so after `chmod +x` the
  script runs as a plain `wp-posts` command from anywhere — no `./` or full
  path needed.
- WHY it filters to `method != "POST"`: GETs are mostly scanners pulling
  static pages; the interesting behavior (login attempts, xmlrpc probes) is in
  the POST bodies.
- `unquote_plus(body)` decodes URL-encoded form data (`log=admin&pwd=test%21`
  → readable `test!`) so credential attempts are legible instead of escaped.
- `SKIP_IP` drops your own test traffic so it can't clutter or outrank the real
  attacker data — the same "subtract yourself before trusting the view"
  discipline used when ranking Cowrie IPs.
- The Nmap labeling reads the `User-Agent` and body shape to name the scan
  (`listMethods`, VMware SOAP `/sdk` probe, or a bare empty POST) instead of
  dumping raw scanner payloads — turns recognizable noise into a one-word tag.

---

## 2 — DAILY USE

```
wp-posts                 # all POSTs except your IP (Nmap + real logins)
wp-posts | grep -v Nmap  # humans / real attackers only — drop the labeled scans
tail -f /var/log/wp-honeypot/http.jsonl   # watch new requests land live
```
- `grep -v Nmap` inverts the match (drops any line the printer tagged as Nmap),
  leaving the genuine login attempts and anything not-obviously-a-scanner.
- `tail -f` follows the raw log live. Unlike a piped `grep`, a bare `tail -f`
  needs no `--line-buffered` — it's the filtering pipes that buffer.

---

## 3 — ONE-OFF FILTERS (no script needed)

Quick answers straight off the raw JSONL when you don't want the table. All
use `http.jsonl*` (globbed) so they include rotated days, not just today.

```
wc -l /var/log/wp-honeypot/http.jsonl*
```
- Total request count across all days. `-l` counts lines, and one line = one
  request.

```
sed -n 's/.*"path": "\([^"]*\)".*/\1/p' /var/log/wp-honeypot/http.jsonl* \
  | sort | uniq -c | sort -rn | head -40
```
- Most-requested paths, ranked. `sed -n '...p'` prints only lines where the
  substitution matched: `\([^"]*\)` captures the path value (everything up to
  the next quote), `\1` replaces the whole line with just that capture. Then
  the usual `sort | uniq -c | sort -rn` groups, counts, and ranks by frequency.
- WHY: instantly shows what scanners are hunting for — `wp-login.php`,
  `xmlrpc.php`, `.env`, `/sdk`, etc.

```
sed -n 's/.*"ip": "\([^"]*\)".*/\1/p' /var/log/wp-honeypot/http.jsonl* \
  | sort | uniq -c | sort -rn | head -20
```
- Same idea for source IPs — the top talkers hitting the HTTP honeypot.

```
sed -n 's/.*"method": "\([^"]*\)".*/\1/p' /var/log/wp-honeypot/http.jsonl* \
  | sort | uniq -c | sort -rn
```
- Method breakdown (GET vs POST vs PUT vs the odd HEAD). A spike in PUTs, for
  instance, is upload-probing worth a closer look.

```
grep -E 'wp-login|xmlrpc|wp-json|wp-content|wlwmanifest|admin-ajax' \
  /var/log/wp-honeypot/http.jsonl*
```
- Pulls every request touching a recognizably-WordPress path. `-E` enables the
  `|` alternation (match any of these). WHY these specific strings: they're the
  files WordPress-targeting scanners reliably probe, so this is a fast "is this
  actually a WP scanner" filter.

```
grep '"method": "POST"' /var/log/wp-honeypot/http.jsonl \
  | grep -v 104.241.55.33 \
  | grep -v 'Nmap Scripting Engine'
```
- The manual equivalent of `wp-posts | grep -v Nmap`, straight off the raw log:
  keep POSTs, drop your own IP, drop Nmap's User-Agent. Useful when you want the
  full raw JSON of each hit rather than the trimmed table.

---

## 4 — THE PRINTER WITHOUT INSTALLING

Same table as section 1, run inline as a one-shot without writing a file —
handy on a box where you don't want to leave the script installed, or to test
a tweak before saving it.

```
python3 - << 'PY'
import json
from pathlib import Path
from urllib.parse import unquote_plus
p = Path("/var/log/wp-honeypot/http.jsonl")
print(f"{'TIME UTC':<20} {'IP':<16} {'PATH':<16} BODY / WHAT")
print("-" * 88)
for line in p.read_text().splitlines():
    if not line.strip():
        continue
    d = json.loads(line)
    if d.get("ip") == "104.241.55.33" or d.get("method") != "POST":
        continue
    body = unquote_plus(d.get("body") or "")
    ua = (d.get("headers") or {}).get("User-Agent", "")
    if "Nmap" in ua:
        extra = "Nmap"
        if "listMethods" in body: extra = "Nmap xmlrpc listMethods"
        elif "RetrieveServiceContent" in body or d.get("path") == "/sdk": extra = "Nmap VMware/SOAP /sdk"
        elif not body: extra = "Nmap empty POST"
    else:
        extra = body[:80] or "(empty)"
    ts = d.get("ts","")[:19].replace("T"," ")
    print(f"{ts:<20} {d.get('ip',''):<16} {d.get('path',''):<16} {extra}")
PY
```
- `python3 - << 'PY' ... PY` feeds the script to Python on stdin (the `-` means
  "read the program from standard input"), running it directly without ever
  saving a file. Quoted `'PY'` again keeps the body literal.

---

END. Pair with Part 3 section 7 (the fake WordPress + nginx build) for how this
log gets produced in the first place.
