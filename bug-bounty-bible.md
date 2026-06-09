# 🛡️ Bug Bounty Bible — Complete Beginner's Guide

> **How to use this:** Read top to bottom once. Then use it as a reference every time you hunt.  
> Every concept here is something you WILL use on real targets.

---

## 📋 Table of Contents

1. [How the Web Works](#1-how-the-web-works)
2. [HTTP — The Language of the Web](#2-http--the-language-of-the-web)
3. [How Authentication Works](#3-how-authentication-works)
4. [APIs — Where Most Bugs Hide](#4-apis--where-most-bugs-hide)
5. [Burp Suite — Your Main Weapon](#5-burp-suite--your-main-weapon)
6. [Vulnerability Classes](#6-vulnerability-classes)
7. [Recon Methodology](#7-recon-methodology)
8. [Attacker Mindset](#8-attacker-mindset)
9. [Writing Bug Reports](#9-writing-bug-reports)

---

## 1. How the Web Works

### The Big Picture

When you type `https://example.com` and hit Enter, here is **exactly** what happens step by step:

```
You type URL
    ↓
Your browser does a DNS lookup
(DNS = phonebook of the internet — converts "example.com" to an IP like 93.184.216.34)
    ↓
Your browser opens a TCP connection to that IP
    ↓
Your browser sends an HTTP request
    ↓
The server processes the request
    ↓
Server sends back an HTTP response
    ↓
Your browser renders the HTML/CSS/JS
```

> **As an attacker:** You sit in the middle of this. Burp Suite intercepts the HTTP request BEFORE it reaches the server — giving you full control to read and modify it.

---

### What is a Client and What is a Server?

| Term | What it is | Example |
|------|-----------|---------|
| **Client** | The browser — sends requests | Chrome, Firefox |
| **Server** | The computer that receives requests and responds | Apache, Nginx |
| **Frontend** | Code that runs in YOUR browser | HTML, CSS, JavaScript |
| **Backend** | Code that runs on the SERVER | Python, Node.js, PHP |

> **Critical insight:** The server should NEVER trust what the client sends. When it does — that's a bug.

---

### What is DNS?

DNS converts human-readable names to IP addresses.

```
example.com  →  DNS lookup  →  93.184.216.34
```

**Why it matters for bug bounty:**
- Subdomains like `dev.example.com` or `api.example.com` are separate attack surfaces
- Forgotten subdomains often have weaker security
- Subdomain takeover happens when a DNS record points to a service that no longer exists

---

### What is HTTPS?

HTTPS = HTTP + TLS encryption. The traffic between your browser and server is encrypted.

**Why it matters for Burp:**
- Burp installs its own certificate to decrypt HTTPS traffic
- This is called a "man-in-the-middle" — and it's exactly how you read encrypted traffic during testing

---

## 2. HTTP — The Language of the Web

### Every Web Action is an HTTP Request

HTTP is just text. A request is a text message your browser sends to a server. A response is a text message the server sends back. **That's it.**

---

### Anatomy of an HTTP Request

```http
POST /api/v1/user/profile HTTP/1.1
Host: target.com
Cookie: session=eyJhbGciOiJIUzI1NiJ9...
Content-Type: application/json
Authorization: Bearer eyJhbGciOiJIUzI1NiJ9...
Content-Length: 42

{"user_id": "123", "email": "test@test.com"}
```

Let's break down every single line:

| Part | What it is | What to test |
|------|-----------|-------------|
| `POST` | HTTP method — what action you're doing | Try changing to GET, PUT, DELETE |
| `/api/v1/user/profile` | The endpoint path | Try `/api/v2/`, `/admin/`, add `/../` |
| `HTTP/1.1` | Protocol version | Usually ignore |
| `Host: target.com` | Which server to talk to | Try host header injection |
| `Cookie: session=...` | Your session token | Try tampering, try another user's cookie |
| `Content-Type` | Format of the body | Try changing to `text/xml` for XXE |
| `Authorization: Bearer ...` | JWT token | Try removing it, try modifying the token |
| `{"user_id": "123"}` | The request body | Change `123` to another user's ID |

> **Golden rule:** Every single value in a request is user-controlled. The server should validate all of them. When it doesn't — there's a bug.

---

### Anatomy of an HTTP Response

```http
HTTP/1.1 200 OK
Content-Type: application/json
Set-Cookie: session=newtoken; HttpOnly; Secure
X-Frame-Options: DENY

{"id": 123, "email": "victim@example.com", "role": "admin", "balance": 5000}
```

| Part | What it tells you |
|------|-----------------|
| `200 OK` | Request succeeded |
| `Set-Cookie` | Server is setting a cookie — check its flags |
| `X-Frame-Options` | If missing — clickjacking possible |
| The JSON body | Look for data you shouldn't be seeing |

---

### HTTP Methods — What Each One Does

| Method | Purpose | Bug bounty angle |
|--------|---------|-----------------|
| `GET` | Fetch data. Params go in URL | Parameters in URL easy to spot and test |
| `POST` | Send data. Params go in body | Forms, login, creating resources |
| `PUT` | Replace an entire resource | Test if you can replace someone else's resource |
| `PATCH` | Partially update a resource | Test if you can update fields you shouldn't |
| `DELETE` | Delete a resource | Test deleting another user's data |
| `OPTIONS` | Ask what methods the server allows | Reveals allowed methods — check for CORS |
| `HEAD` | Like GET but no body returned | Sometimes reveals info |

> **Pro tip:** Try sending a `GET` request to an endpoint that normally only accepts `POST`. Servers sometimes handle it differently and expose data.

---

### HTTP Status Codes — What They Tell You

| Code | Meaning | What to do as a hunter |
|------|---------|----------------------|
| `200 OK` | Success | Read the response carefully |
| `201 Created` | Something was created | Note what was created |
| `301/302` | Redirect | Test for open redirect |
| `400 Bad Request` | You sent something wrong | Note — keep trying |
| `401 Unauthorized` | Not logged in | Try to bypass |
| `403 Forbidden` | Logged in but not allowed | **Try to bypass — huge bug class** |
| `404 Not Found` | Doesn't exist | Keep fuzzing |
| `405 Method Not Allowed` | Wrong HTTP method | Try other methods |
| `429 Too Many Requests` | Rate limited | Note — try to bypass |
| `500 Internal Server Error` | Server crashed | **Something interesting happened — note it** |

> **500 errors are gold.** When the server crashes, it means your input did something unexpected. Stack traces in 500 responses reveal server technology, file paths, and code structure.

---

### HTTP Headers — The Security-Critical Ones

Headers carry metadata about the request or response. As a bug hunter you need to know which headers affect security.

#### Request headers you should know:

```
Host: target.com           ← Target server name. Test: Host: evil.com
Cookie: session=abc123     ← Your session. Test: change it, remove it
Authorization: Bearer ...  ← Your token. Test: remove it, modify it
Referer: https://site.com  ← Where you came from. Sometimes checked for CSRF
Origin: https://site.com   ← CORS origin. Test: change to another site
X-Forwarded-For: 127.0.0.1 ← Client's real IP. Test: spoof to bypass IP restrictions
Content-Type: application/json  ← Format of body. Test: change to XML
```

#### Response headers you should check:

```
Set-Cookie: session=abc; HttpOnly; Secure; SameSite=Strict
```

| Cookie Flag | What it does | Missing = vulnerability |
|------------|-------------|------------------------|
| `HttpOnly` | JS cannot read this cookie | Missing = XSS can steal the session |
| `Secure` | Only sent over HTTPS | Missing = cookie sent over HTTP (can be sniffed) |
| `SameSite=Strict` | Only sent from same site | Missing = CSRF possible |

```
Content-Security-Policy: script-src 'self'   ← Prevents XSS. Missing = easier XSS
X-Frame-Options: DENY                         ← Prevents clickjacking. Missing = bug
Access-Control-Allow-Origin: *               ← CORS. * = any site can read API
Strict-Transport-Security: max-age=31536000  ← Forces HTTPS. Missing = downgrade possible
```

---

### URL Structure — Every Part Matters

```
https://api.target.com:443/v1/users/profile?id=123&format=json#section

|_____|  |____________| |_| |______________|  |______________|  |_____|
scheme      host        port     path             query          fragment
```

| Part | What to test |
|------|-------------|
| `https` | Try `http` — does it redirect properly? |
| `api.target.com` | The subdomain — separate attack surface |
| `/v1/users/profile` | Try `/v2/`, `/admin/`, path traversal `/../` |
| `id=123` | **Change this number — IDOR test** |
| `format=json` | Try `format=xml` — XXE possible? |

---

### Cookies vs Local Storage vs Session Storage

| Storage | Where | Accessible by JS? | Sent automatically? | Bug angle |
|---------|-------|------------------|--------------------|---------  |
| **Cookie** | Browser | Only if no HttpOnly | Yes, every request | Steal via XSS, forge |
| **localStorage** | Browser | Yes | No | Steal via XSS |
| **sessionStorage** | Browser | Yes | No | Steal via XSS |

> **Why this matters:** Tokens stored in localStorage/sessionStorage are accessible by JavaScript. If there's XSS — you can steal them with `document.localStorage.getItem('token')`.

---

## 3. How Authentication Works

### Session-Based Authentication

```
1. You log in with username + password
2. Server verifies credentials
3. Server creates a session record in its database
4. Server sends you a session ID in a cookie: Set-Cookie: session=abc123
5. Every future request automatically includes that cookie
6. Server looks up abc123 in its database → knows who you are
```

**What to attack:**
- Guess or brute force the session ID (if it's predictable)
- Steal someone's session ID via XSS
- Test if session is invalidated after logout

---

### Token-Based Authentication (JWT)

JWT = JSON Web Token. Used by modern APIs instead of sessions.

```
A JWT looks like this:
eyJhbGciOiJIUzI1NiJ9.eyJ1c2VySWQiOiIxMjMiLCJyb2xlIjoidXNlciJ9.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c

Split by dots:
Part 1 (Header):  eyJhbGciOiJIUzI1NiJ9
Part 2 (Payload): eyJ1c2VySWQiOiIxMjMiLCJyb2xlIjoidXNlciJ9
Part 3 (Signature): SflKxwRJSMeKKF2QT4...
```

Decode Part 2 in Base64 and you get:
```json
{
  "userId": "123",
  "role": "user",
  "exp": 1893456000
}
```

**JWT attacks to test:**

| Attack | What you do | What it exploits |
|--------|------------|-----------------|
| **None algorithm** | Change `"alg":"HS256"` to `"alg":"none"`, delete signature | Server skips signature verification |
| **Weak secret** | Server uses "password" or "secret" as signing key | Brute force the key, forge tokens |
| **Change claims** | Decode payload, change `"role":"user"` to `"role":"admin"`, re-sign | Privilege escalation |
| **Expired token** | Use a token after `exp` date | Server doesn't check expiry |

---

### Password Reset Flows — Always Test These

Password reset is a common source of vulnerabilities.

**What to test:**
1. Is the reset token random or predictable? (e.g. based on timestamp)
2. Does the token expire? Can you use it 24 hours later?
3. Can you use the same token twice?
4. What happens if you change the `Host` header during reset? (host header injection — sends reset link to your domain)
5. Is there rate limiting on reset requests?

---

## 4. APIs — Where Most Bugs Hide

### What is an API?

Modern web apps are split in two:
- **Frontend:** JavaScript running in your browser (React, Vue, Angular)
- **Backend:** An API server that the JavaScript calls

```
Browser JS  →  fetch('/api/users/123')  →  API Server  →  Database
                                                              ↓
Browser JS  ←  {"id":123, "email":"..."}  ←  API Server  ←
```

> **Why APIs have more bugs:** Developers often protect the HTML pages but forget to protect the API endpoints. The page checks if you're admin — but the API behind it sometimes doesn't.

---

### REST API Patterns — Recognize These

```
GET    /api/v1/users          → Get all users
GET    /api/v1/users/123      → Get user 123
POST   /api/v1/users          → Create a new user
PUT    /api/v1/users/123      → Replace user 123
PATCH  /api/v1/users/123      → Update user 123
DELETE /api/v1/users/123      → Delete user 123
```

**IDOR test on every single one of these:** replace `123` with `124`, `125`, `1`, `999999`

---

### How to Find API Endpoints

1. **Burp HTTP History** — browse the app, all API calls appear here
2. **JavaScript files** — endpoints are hardcoded in JS
3. **Network tab in DevTools** — watch XHR/fetch requests as you use the app
4. **Common patterns:**
   - `/api/` `/api/v1/` `/api/v2/`
   - `/graphql`
   - `/rest/`
   - `/service/`

```bash
# Find endpoints from wayback machine
waybackurls target.com | grep "/api"

# Find endpoints in JS files
cat app.js | grep -E "(GET|POST|PUT|DELETE|PATCH|fetch|axios)" 
```

---

### GraphQL — Special API Type

GraphQL is a different API style where everything goes to one endpoint.

```
POST /graphql

{"query": "{ user(id: 123) { email password role } }"}
```

**What to test:**
- Introspection query — reveals entire API schema
- Try accessing other users' data in queries
- Try mutations (write operations) on other users' data

```graphql
# Introspection — reveals all queries and mutations
{ __schema { types { name fields { name } } } }
```

---

## 5. Burp Suite — Your Main Weapon

### What Burp Actually Does

Burp Suite sits between your browser and the internet. Every request your browser sends passes through Burp first. Burp captures it, shows it to you, and lets you modify it before it reaches the server.

```
Your Browser  ──────→  Burp (port 8080)  ──────→  Target Server
              ←──────                    ←──────
              
You see and control every request and response
```

---

### One-Time Setup (Do This Once)

```
1. Open Burp Suite
2. Go to Proxy → Options → verify listener is 127.0.0.1:8080
3. Open Firefox → install FoxyProxy extension
4. In FoxyProxy: add proxy → 127.0.0.1 → port 8080
5. In Firefox: visit http://burpsuite → download CA certificate
6. Firefox → Settings → search "Certificates" → View Certificates
   → Authorities tab → Import the certificate you downloaded
7. Done — Burp can now see all HTTPS traffic
```

---

### The 4 Tools You Use Every Day

---

#### Tool 1: Proxy — The Traffic Capture Layer

**What it does:** Passively captures every request your browser makes while you browse the target app.

**Two modes:**

| Mode | What happens |
|------|-------------|
| **Intercept ON** | Every request PAUSES and waits for you to approve or edit it before continuing |
| **Intercept OFF** | Requests flow through automatically, silently captured in HTTP History |

**When to use each:**
- **Intercept OFF** most of the time — browse naturally, capture everything
- **Intercept ON** when you want to catch and edit a specific request before it sends

---

**HTTP History — Your Most Important View**

```
Proxy → HTTP History
```

This is a log of EVERY request your browser made. This is where your hunting starts.

**How to use HTTP History effectively:**

```
1. Browse the entire target app with Intercept OFF
2. Go to HTTP History — you'll see hundreds of requests
3. Filter by interesting patterns:
   - Right-click column headers → filter
   - Search bar at top: type "api" or "user" or "admin"
4. Look for requests with:
   - Parameters that contain IDs (user_id, invoice_id, order_id)
   - POST requests to /api/ endpoints
   - Requests with Authorization headers
   - Requests to /admin/, /account/, /profile/
5. Right-click any interesting request → Send to Repeater
```

**What to look for in HTTP History:**

| What you see | Why it's interesting |
|-------------|---------------------|
| `/api/users/123` | Change 123 to 124 — IDOR test |
| `?redirect=https://...` | Test open redirect |
| Large JSON response | Check if it contains data you shouldn't see |
| Request without auth | API endpoint missing authentication |
| `/admin/` endpoint | Try to access it |

---

#### Tool 2: Repeater — Your Testing Bench

**What it does:** Takes one specific HTTP request and lets you edit it and re-send it as many times as you want. This is where you actually test for vulnerabilities.

**How to get a request into Repeater:**
```
HTTP History → Right-click any request → "Send to Repeater"
```

**The Repeater interface:**

```
┌─────────────────────────────────────────────────────────┐
│  LEFT SIDE: Your request (you edit this)                │
│                                                         │
│  POST /api/users/123 HTTP/1.1                          │
│  Host: target.com                                       │
│  Cookie: session=abc123                                 │
│  {"action": "getProfile"}                              │
│                                                         │
├─────────────────────────────────────────────────────────┤
│  RIGHT SIDE: Server's response (you read this)         │
│                                                         │
│  HTTP/1.1 200 OK                                       │
│  {"id": 123, "email": "user@example.com"}              │
└─────────────────────────────────────────────────────────┘
```

**Hit Send → response appears on the right. Modify request → hit Send again. Repeat.**

---

**What to do in Repeater for different bug types:**

**Testing IDOR:**
```
Original request:  GET /api/invoice/1234
                   Cookie: session=[your token]

Change to:         GET /api/invoice/1235
                   Cookie: session=[your token]

If response shows someone else's invoice → IDOR confirmed ✓
```

**Testing authentication:**
```
Original request:  GET /api/admin/users
                   Authorization: Bearer eyJhbGci...

Remove header:     GET /api/admin/users
                   (no Authorization header)

If response still returns data → missing auth ✓
```

**Testing XSS:**
```
Original request:  POST /api/comments
                   {"text": "hello"}

Change to:         POST /api/comments
                   {"text": "<script>alert(1)</script>"}

If the script appears unescaped in response → XSS ✓
```

**Testing SQLi:**
```
Original request:  GET /api/search?q=phone

Change to:         GET /api/search?q=phone'

If response is a 500 error or database error → possible SQLi ✓
```

**What to look for in the response:**
- Response `200` instead of `403` after removing auth → auth bypass
- Another user's data in the response → IDOR
- Your `<script>` tag reflected back unescaped → XSS
- A database error message → SQLi lead
- Different response length → something changed

---

#### Tool 3: Intruder — Automated Payload Testing

**What it does:** Takes one request, marks a position (a parameter value you want to fuzz), and automatically fires a list of payloads at it one by one.

**Use cases:**
- Testing 1000 different user IDs to find IDOR
- Testing XSS payloads on an input
- Brute forcing a login
- Testing a parameter with many different values

**How to use Intruder:**

```
Step 1: Send request from Repeater or HTTP History to Intruder
        Right-click → Send to Intruder

Step 2: In Intruder → Positions tab
        Clear all auto-detected positions (Clear § button)
        Highlight the value you want to fuzz
        Click Add § button
        
        Result: GET /api/user/§123§ HTTP/1.1

Step 3: Payloads tab
        Add your payload list (numbers 1-1000, XSS payloads, etc.)

Step 4: Click Start Attack

Step 5: Sort results by:
        - Response Length (different length = different behaviour = interesting)
        - Status Code (200 when everything else is 403 = found something)
```

**Attack types:**

| Type | Use case |
|------|---------|
| **Sniper** | One position, one payload list — use this 90% of the time |
| **Battering ram** | Same payload in multiple positions simultaneously |
| **Pitchfork** | Multiple positions, multiple lists — one-to-one matching |
| **Cluster bomb** | Multiple positions, all combinations — very slow |

> **Important:** Community Edition throttles Intruder to ~1 request/second. For fast fuzzing use `ffuf` in terminal instead.

---

#### Tool 4: Decoder — Decode Anything Instantly

**What it does:** Encodes and decodes data in Base64, URL encoding, HTML encoding, hex, and more.

**When you use it:**
- Session tokens are usually Base64 encoded — decode to read them
- JWT tokens are Base64 — decode to see the claims
- URL parameters are URL encoded — decode to read them
- After decoding, modify the value, re-encode, paste back into request

**How to use:**
```
1. Copy the encoded value from your request
2. Go to Decoder tab
3. Paste the value
4. Select encoding type (Base64, URL, HTML...)
5. Click Decode → you can now read it
6. Modify the decoded value
7. Click Encode → copy the new encoded value
8. Paste back into your request in Repeater
```

**Example — decoding a JWT:**
```
JWT token: eyJhbGciOiJIUzI1NiJ9.eyJ1c2VySWQiOiIxMjMifQ.xxxxx

Take the middle part: eyJ1c2VySWQiOiIxMjMifQ
Decode as Base64: {"userId":"123","role":"user"}

Now you know the structure — try changing "role":"user" to "role":"admin"
Re-encode → replace in the Authorization header → send in Repeater
```

---

### Burp Workflow — Exactly What to Do on a New Target

```
STEP 1: Setup
─────────────
Open Burp → Proxy → turn Intercept OFF
Open Firefox with FoxyProxy → point to 127.0.0.1:8080

STEP 2: Map the application
────────────────────────────
Browse every single feature of the target:
  ✓ Register and log in
  ✓ View your profile
  ✓ Edit your profile
  ✓ Upload a file (if available)
  ✓ Use the search
  ✓ View invoices, orders, settings
  ✓ Log out → log back in
  ✓ Use the password reset flow
  ✓ Try any admin features

Everything is now in HTTP History.

STEP 3: Review HTTP History
────────────────────────────
Filter for interesting requests:
  → Search for: /api, /user, /admin, /account, /profile, /invoice
  → Look for requests with IDs in URL or body
  → Look for POST/PUT/DELETE requests
  → Look for requests with Authorization headers

STEP 4: Test in Repeater
─────────────────────────
For each interesting request:
  → Send to Repeater
  → Change user IDs → IDOR test
  → Remove Authorization header → auth test
  → Add ' to parameters → SQLi test
  → Add XSS payload to text inputs
  → Change your user ID to another's in POST body

STEP 5: Document everything
────────────────────────────
Found something? 
  → Screenshot the request in Repeater
  → Screenshot the response showing the bug
  → Copy raw request (right-click → Copy as curl)
  → Write it up
```

---

### Burp Keyboard Shortcuts (Save These)

| Shortcut | Action |
|----------|--------|
| `Ctrl+R` | Send request to Repeater |
| `Ctrl+I` | Send request to Intruder |
| `Ctrl+Enter` | Send request in Repeater |
| `Ctrl+Z` | Undo changes in request |
| `Ctrl+F` | Search in request/response |
| `Ctrl+Shift+F` | Search across all history |

---

## 6. Vulnerability Classes

### IDOR — Insecure Direct Object Reference

**What it is:** The server gives you access to an object (user data, invoice, order, file) using an ID, but doesn't check if YOU own that object.

**Simple example:**
```
You are logged in as User 123.
You visit: https://target.com/api/invoice/555
The server returns your invoice.

Now change the URL to: https://target.com/api/invoice/556
The server returns ANOTHER USER'S invoice.

This is IDOR. The server never checked: "Does user 123 own invoice 556?"
```

**Where to look:**
- Any URL with a number: `/users/123`, `/orders/456`, `/invoices/789`
- Any POST body with an ID: `{"user_id": 123}`, `{"account_id": "abc"}`
- Any hidden form field with an ID
- File downloads: `/download?file_id=123`

**How to test:**
```
1. Create two accounts (Account A and Account B)
2. Log in as Account A
3. Create something (a post, order, invoice)
4. Note the ID of what you created
5. Copy the request to Repeater
6. Replace your session cookie with Account B's session cookie
7. Send — do you still get Account A's data?
8. YES = IDOR confirmed
```

---

### XSS — Cross-Site Scripting

**What it is:** You inject JavaScript into a web page. When other users visit that page, your script runs in their browser — as if the site wrote it.

**Why it's dangerous:**
- Steal session cookies → take over accounts
- Redirect users to phishing pages
- Log keystrokes
- Make requests on behalf of the user (CSRF bypass)

**Three types:**

| Type | How it works | Severity |
|------|-------------|---------|
| **Reflected** | Your script is in the URL, reflected in the response. Only affects people who click your link | Medium |
| **Stored** | Your script is saved in the database. Every user who views the page runs your script | High |
| **DOM-based** | JavaScript on the page reads your input and writes it to the DOM unsafely | Medium |

**How to test:**
```
1. Find every input field, URL parameter, and header that appears in the response
2. Enter a basic payload: <script>alert(1)</script>
3. If an alert box appears — XSS confirmed
4. If it doesn't — the input is being encoded/filtered
5. Try bypass payloads:
   "><script>alert(1)</script>
   <img src=x onerror=alert(1)>
   <svg onload=alert(1)>
   javascript:alert(1)
```

**Where to look:**
- Search bars (your query appears on the page)
- Comment sections
- Profile name/bio fields
- Error messages (the error reflects your input)
- URL parameters that appear in the page

---

### SQL Injection

**What it is:** Your input gets inserted directly into a database SQL query. You break out of the intended query and run your own SQL commands.

**Simple example:**
```sql
Normal query:
SELECT * FROM users WHERE username = 'john' AND password = 'pass123'

You enter as username: ' OR 1=1 --
Query becomes:
SELECT * FROM users WHERE username = '' OR 1=1 -- ' AND password = '...'
                                            ↑         ↑
                                      1=1 always   -- comments out the rest
                                        true

Result: Logs you in as the first user in the database (usually admin)
```

**How to test:**
```
1. Find any input that queries a database (search, login, filters)
2. Enter a single quote: '
3. If you get a 500 error or a database error message → SQLi likely
4. If the page behaves differently → investigate more
5. Try: ' OR '1'='1
6. Try: ' OR 1=1 --
```

**Tools:** `sqlmap` automates SQLi detection and exploitation.

---

### CSRF — Cross-Site Request Forgery

**What it is:** You trick a logged-in user into making a request to the target app without them knowing. Their browser sends the request with their real cookies automatically.

**Example:**
```
User is logged into bank.com
You send them a link to evil.com
evil.com has: <img src="https://bank.com/transfer?to=attacker&amount=10000">
Browser loads the image → automatically sends request to bank.com with user's cookie
Bank transfers money
```

**How to test:**
```
1. Find a sensitive action (change email, change password, transfer money)
2. Look at the request in Burp — is there a CSRF token?
   - CSRF token: a random secret value in the form that the server verifies
3. If NO CSRF token → potentially vulnerable
4. If CSRF token → try:
   - Remove the token entirely — does it still work?
   - Use an old/expired token — does it still work?
   - Use a token from a different session — does it still work?
```

---

### SSRF — Server-Side Request Forgery

**What it is:** You make the SERVER send a request to a URL you control. You can point it at internal services that aren't accessible from the internet.

**Example:**
```
Target app has a feature: "Enter a URL to preview a website"
https://target.com/preview?url=https://example.com

You change the URL to: https://target.com/preview?url=http://169.254.169.254/latest/meta-data/
(169.254.169.254 is the AWS metadata endpoint — returns cloud server credentials)
```

**Where to look:**
- Any feature that fetches a URL: webhooks, URL previews, PDF generators, image importers
- Parameters named: `url`, `fetch`, `request`, `redirect`, `link`, `webhook`, `callback`, `target`, `dest`

**What to try:**
```
http://127.0.0.1          (localhost)
http://localhost           (localhost)
http://169.254.169.254    (AWS metadata)
http://192.168.0.1        (internal network)
http://[::1]              (IPv6 localhost)
```

---

### File Upload Vulnerabilities

**What it is:** The app lets you upload files but doesn't properly validate what you're uploading. You upload a malicious file that the server then executes.

**How to test:**
```
1. Upload a normal image file — does it work?
2. Try uploading a PHP file: shell.php with content <?php system($_GET['cmd']); ?>
3. If blocked, try:
   - Change filename to: shell.php.jpg
   - Change Content-Type header to: image/jpeg (while keeping .php extension)
   - Use null byte: shell.php%00.jpg
   - Try other extensions: .phtml, .php5, .phar, .shtml
4. If upload succeeds, find where the file is stored and access it
```

---

### Open Redirect

**What it is:** The app redirects you to a URL specified in a parameter, without checking if it's safe.

**Why it matters:** Phishing. Send someone a legitimate-looking link to target.com — they get redirected to evil.com.

**How to find:**
```
Look for these parameters:
?next=https://...
?redirect=https://...
?url=https://...
?return=https://...
?goto=https://...
?destination=https://...
?redir=https://...
?back=https://...
```

**How to test:**
```
Change the URL to your own domain:
?next=https://evil.com

Or try bypasses if simple redirect is blocked:
?next=https://evil.com%0d%0a (CRLF)
?next=//evil.com
?next=https://target.com.evil.com
```

---

### Business Logic Vulnerabilities

**What it is:** The app's code works exactly as written — but the logic itself is flawed. These bugs can't be found with scanners. Only a human who thinks creatively finds them.

**Examples:**

| Scenario | What to try |
|---------|------------|
| E-commerce discount code | Apply the same code twice |
| Free trial period | Manipulate dates in the request |
| Multi-step checkout | Skip step 2, go directly to step 3 |
| Transfer limits | Use a negative value to reverse a transfer |
| Rate-limited action | Use the API endpoint instead of the web form |
| Feature behind paywall | Change `"plan":"free"` to `"plan":"premium"` in request |

---

### Information Disclosure

**What it is:** The app reveals sensitive information it shouldn't.

**Where to find it:**
```
/robots.txt           → may reveal hidden paths
/.git/                → exposed git repository (huge finding)
/.env                 → environment variables with API keys and passwords
/api/swagger.json     → API documentation revealing all endpoints
Error messages        → stack traces revealing server details, file paths
JavaScript files      → hardcoded API keys, internal URLs, secrets
HTTP response headers → server version, framework version
```

---

## 7. Recon Methodology

### Phase 1: Before Touching the Target

```bash
# Step 1: Find all subdomains
subfinder -d target.com -o subdomains.txt

# Step 2: Check which subdomains are alive
httpx -l subdomains.txt -o alive.txt

# Step 3: Take screenshots of all alive subdomains
gowitness file -f alive.txt

# Step 4: Find old URLs from wayback machine
waybackurls target.com -o wayback.txt

# Step 5: Find endpoints in JS files
gau target.com | grep ".js" > jsfiles.txt
```

### Phase 2: Manual Recon on Each Subdomain

**Files to check on every subdomain:**
```
/robots.txt
/sitemap.xml
/.well-known/security.txt
/.git/HEAD           ← if this returns 200, huge finding
/.env                ← if this returns 200, huge finding
/backup.zip          ← backup files
/admin               ← admin panel
/debug               ← debug interface
```

### Phase 3: JavaScript File Mining

JS files are goldmines. They often contain:
- Hidden API endpoints
- Hardcoded API keys
- Internal service URLs
- Developer comments with passwords

```bash
# Download a JS file and search it
curl https://target.com/app.js | grep -E "(api_key|secret|password|token|internal|admin|/api/)"
```

**In Burp:** HTTP History → filter by `.js` → right-click → Send to Repeater → read the response

---

### Phase 4: Application Mapping

Browse the entire app with Burp running. Map every feature:
```
✓ Registration and login
✓ Password reset
✓ Profile management
✓ File uploads
✓ Search functionality
✓ Any payment or checkout
✓ Admin features (try to access even if you're not admin)
✓ API endpoints (visible in Network tab and Burp)
✓ Mobile API endpoints (often /api/mobile/ or /api/app/)
```

---

## 8. Attacker Mindset

### The Core Principle

**Developers build apps assuming users will behave normally.**  
**Attackers don't behave normally.**

Every time you see something in an app — a form, a URL parameter, a file upload, an ID — your question is: *"What happens if I send something unexpected here?"*

---

### The 3 Questions for Every Feature

```
1. Can I access something I don't own?
   → Test IDOR on every ID you see

2. Can I send something the app didn't expect?
   → Test XSS, SQLi, file upload on every input

3. Can I make the app do something it wasn't designed to do?
   → Test business logic — what is NOT supposed to be possible?
```

---

### Where Developers Make Mistakes

| Mistake | Resulting bug |
|---------|--------------|
| Trust the user_id sent in the request | IDOR |
| Validate input only on the frontend | SQLi, XSS |
| Protect the HTML page but not the API endpoint | Auth bypass |
| Forget that old API versions (/v1/) still exist | Unprotected endpoints |
| Leave debug/admin endpoints in production | Information disclosure |
| Store secrets in JS files | API key exposure |
| Not checking file type server-side | File upload RCE |
| Not verifying CSRF tokens | CSRF |

---

### The Two-Account Technique

**Every single time you hunt — create two accounts.**

```
Account A: attacker@gmail.com   (your main hunting account)
Account B: victim@gmail.com     (the "victim" account)

Workflow:
1. Log in as Account B → create a resource → note the ID
2. Log in as Account A
3. Try to access Account B's resource using Account A's session
4. In Burp Repeater:
   - Use Account A's session cookie in the Cookie header
   - Try to access Account B's resource ID
5. If you can read/edit/delete Account B's data as Account A → IDOR
```

---

### Think Like a Developer (to Find Their Mistakes)

- **If you were building this feature, what shortcuts would you take?**
- **What would you forget to check?**
- "I'll just trust the user_id in the request — saves a database lookup"  → IDOR
- "The frontend validates the file type, no need to check on server too" → File upload
- "This admin endpoint is not linked anywhere, nobody will find it"  → Security by obscurity

---

## 9. Writing Bug Reports

### A Good Report Gets Paid. A Bad Report Gets Closed.

**Title format:**
```
[Vulnerability Type] in [Feature/Endpoint] allows [Impact]

Examples:
✓ IDOR in /api/v1/invoices allows unauthorized access to any user's invoice
✓ Stored XSS in profile bio field allows session hijacking
✓ Missing authentication on /api/admin/users exposes all user data
```

---

### Report Template

```markdown
## Summary
[2-3 sentences. What is the bug, where is it, what's the impact]

The /api/v1/invoices endpoint does not verify that the requesting user owns the 
invoice ID provided in the URL. An attacker can access any user's invoice by 
changing the invoice ID, exposing sensitive financial data.

## Severity
High

## Steps to Reproduce
1. Create Account A (attacker) and Account B (victim)
2. Log in as Account B and create an invoice. Note the invoice ID (e.g., 4521)
3. Log in as Account A
4. Send the following request:
   GET /api/v1/invoices/4521 HTTP/1.1
   Host: target.com
   Cookie: session=[Account A's session token]
5. The response returns Account B's full invoice data

## Expected Behavior
Server should return 403 Forbidden — Account A does not own invoice 4521

## Actual Behavior
Server returns 200 OK with Account B's complete invoice including:
- Full name, address, payment details

## Impact
An attacker can access the private financial data of any user by iterating 
invoice IDs. This exposes PII and violates user privacy.

## Evidence
[Screenshot of Burp request]
[Screenshot of Burp response showing victim's data]

## Proof of Concept
curl -H "Cookie: session=ATTACKER_SESSION" https://target.com/api/v1/invoices/VICTIM_ID
```

---

### Severity Levels

| Level | CVSS Score | Examples |
|-------|-----------|---------|
| **Critical** | 9.0 – 10.0 | RCE, SQLi on auth bypass, account takeover without interaction |
| **High** | 7.0 – 8.9 | IDOR exposing sensitive data, stored XSS, auth bypass |
| **Medium** | 4.0 – 6.9 | CSRF, reflected XSS, open redirect |
| **Low** | 0.1 – 3.9 | Missing headers, clickjacking, verbose errors |
| **Informational** | N/A | Best practice suggestions |

---

### Before Submitting — Checklist

```
✓ Can you reproduce the bug consistently?
✓ Is the endpoint in scope for this program?
✓ Have you blurred or removed real user PII from screenshots?
✓ Is your title clear and specific?
✓ Are your steps to reproduce exact and complete?
✓ Have you included proof (screenshots, curl command)?
✓ Is your impact statement realistic — not overstated?
```

---

## Quick Reference Card

### Parameters to Always Test

```
?id=          → change number → IDOR
?user_id=     → change number → IDOR
?file=        → path traversal: ../../etc/passwd
?url=         → SSRF: http://127.0.0.1 or open redirect: https://evil.com
?redirect=    → open redirect: https://evil.com
?search=      → XSS: <script>alert(1)</script>  SQLi: '
?format=      → change to XML: XXE
?token=       → decode in Burp Decoder, modify, re-encode
```

### First 10 Minutes on Any Target

```
1. Open Burp, turn Intercept OFF
2. Browse the entire app (login, profile, every feature)
3. Visit /robots.txt and /.env
4. Go to HTTP History in Burp — review all requests
5. Filter for /api/ — look for IDs in URLs
6. Send 3 most interesting requests to Repeater
7. On each: remove auth header, change IDs, add '
8. Check response size — different size = interesting
```

### XSS Quick Payload List

```html
<script>alert(1)</script>
"><script>alert(1)</script>
<img src=x onerror=alert(1)>
<svg onload=alert(1)>
<body onload=alert(1)>
javascript:alert(1)
'-alert(1)-'
\"-alert(1)//
```

### SQLi Quick Test List

```sql
'
''
`
')
'))
' OR '1'='1
' OR 1=1 --
admin'--
' UNION SELECT NULL--
```

---

*Last updated: June 2026 | Use only on authorized targets*
