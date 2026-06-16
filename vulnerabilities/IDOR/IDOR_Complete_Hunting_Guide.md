# 🔐 IDOR Complete Hunting Guide — Beginner to Advanced

> **"Today all I'm gonna do is find as many IDOR bugs as possible."**
> This guide is your single reference. Read it top to bottom, then go hunt.

---

## Table of Contents

1. [What is IDOR?](#1-what-is-idor)
2. [Why IDOR Matters](#2-why-idor-matters)
3. [All Types of IDOR](#3-all-types-of-idor)
4. [Where IDORs Hide — Attack Surface Map](#4-where-idors-hide)
5. [Hunting Methodology — Step by Step](#5-hunting-methodology)
6. [Burp Suite IDOR Workflow](#6-burp-suite-idor-workflow)
7. [Beginner Techniques](#7-beginner-techniques)
8. [Intermediate Techniques](#8-intermediate-techniques)
9. [Advanced Techniques](#9-advanced-techniques)
10. [Chaining IDOR with Other Bugs](#10-chaining-idor-with-other-bugs)
11. [Real HackerOne Reports (Disclosed)](#11-real-hackerone-reports)
12. [Tools & Extensions](#12-tools--extensions)
13. [IDOR Testing Checklist](#13-idor-testing-checklist)
14. [Writing a Good Report](#14-writing-a-good-report)
15. [Quick Reference Cheat Sheet](#15-quick-reference-cheat-sheet)

---

## 1. What is IDOR?

**IDOR = Insecure Direct Object Reference**

It is a type of **Broken Access Control** vulnerability (OWASP Top 10 — A01:2021).

IDOR happens when an application uses **user-controllable input** to access objects (files, database records, user data) **directly**, without verifying that the requesting user is **authorized** to access that specific object.

### Simple Example

```
Normal request:
GET /api/user/profile?user_id=1001   (your ID)
Response: { "name": "You", "email": "you@email.com" }

IDOR attack:
GET /api/user/profile?user_id=1002   (another user's ID)
Response: { "name": "Victim", "email": "victim@email.com" }  ← should be blocked!
```

The server returned data it shouldn't have — because it only checked "are you logged in?" not "does this data belong to you?"

### The Root Cause

```
❌ Vulnerable code logic:
   1. Is user authenticated? → YES → return the data

✅ Secure code logic:
   1. Is user authenticated? → YES
   2. Does this object belong to the authenticated user? → only then return data
```

---

## 2. Why IDOR Matters

| Impact | Description |
|--------|-------------|
| **Data theft** | Read other users' PII, emails, orders, medical records |
| **Account takeover** | Change another user's password/email |
| **Financial fraud** | View/modify invoices, payments, orders |
| **Privilege escalation** | Access admin functions as a regular user |
| **Data destruction** | Delete other users' files, accounts, records |

IDOR is consistently one of the **highest-paid** bug classes on HackerOne, with bounties ranging from $150 to $30,000+ depending on impact.

---

## 3. All Types of IDOR

### 3.1 — Classic / Numeric IDOR

The most common. An integer ID is directly exposed and incrementable.

```
/api/orders/1001  →  try /api/orders/1002
/download?file_id=5  →  try /download?file_id=6
/invoice/download/9873  →  try /invoice/download/9874
```

**What to look for:** Sequential integers, auto-increment database IDs.

---

### 3.2 — UUID / GUID IDOR

GUIDs look random but may still be leaked elsewhere (in emails, other API responses, referrer headers).

```
/api/document/3f2504e0-4f89-11d3-9a0c-0305e82c3301
```

**How to exploit:**
- Check if GUIDs are leaked in other endpoints (e.g., `/api/activity-feed`)
- Check email notifications, shareable links, referrer headers
- Sometimes GUIDs are predictable (time-based UUIDs v1)

---

### 3.3 — Hashed / Encoded IDOR

IDs are Base64, MD5, or otherwise encoded — but the underlying value is still guessable.

```
/profile?id=MTAwMQ==  →  base64 decode → 1001
/file?token=5f4dcc3b5aa765d61d8327deb882cf99  →  MD5 of "password"
```

**How to exploit:**
- Decode the value, increment, re-encode
- Burp Suite Decoder tab is your friend here

---

### 3.4 — Parameter Pollution IDOR

Adding a second parameter overrides or confuses the server's authorization check.

```
Original:  POST /api/update?user_id=1001
Attack:    POST /api/update?user_id=1001&user_id=1002

Or in body:
{"user_id": 1001, "user_id": 1002}
```

---

### 3.5 — Path Traversal IDOR

Object references hidden inside the URL path.

```
/users/me/documents  →  /users/1002/documents
/api/v1/accounts/self/settings  →  /api/v1/accounts/[other_id]/settings
```

---

### 3.6 — Indirect IDOR (Reference via another object)

You don't directly control the victim's ID, but you can reference an object that belongs to them.

```
POST /api/messages/send
{"thread_id": 9999}   ← thread belonging to another user
```

---

### 3.7 — File/Resource IDOR

Direct access to files by predictable names or paths.

```
/uploads/invoices/invoice_1001.pdf  →  try invoice_1002.pdf
/exports/report_user_5583.csv  →  try report_user_5584.csv
/avatars/user_1001.jpg  →  try user_1002.jpg
```

---

### 3.8 — GraphQL IDOR

GraphQL queries/mutations that accept IDs without checking ownership.

```graphql
query {
  user(id: "1002") {
    email
    privateNotes
    paymentInfo
  }
}
```

Also look for **introspection** to discover hidden fields you can query.

---

### 3.9 — JSON / Body Parameter IDOR

IDs hidden inside POST/PUT/PATCH request bodies.

```json
PATCH /api/profile/update
{
  "user_id": 1002,       ← change this
  "email": "attacker@evil.com"
}
```

---

### 3.10 — HTTP Header IDOR

Some APIs use custom headers to identify the user/resource.

```
X-User-ID: 1001  →  change to X-User-ID: 1002
X-Account-ID: abc123
X-Org-ID: 5001
```

---

### 3.11 — State-Based / Blind IDOR

You can't read the response, but you can infer whether the action succeeded (e.g., deleting another user's resource returns HTTP 200 vs 404).

```
DELETE /api/posts/9999
200 OK  →  you deleted someone else's post (blind IDOR)
404 Not Found  →  post doesn't exist
403 Forbidden  →  access control working
```

---

### 3.12 — Batch / Mass Assignment IDOR

APIs that accept arrays of IDs to perform bulk actions.

```json
POST /api/messages/read
{"message_ids": [101, 102, 9999]}   ← 9999 belongs to another user
```

---

### 3.13 — Export / Report IDOR

Download/export features that generate reports using an ID.

```
GET /export/user-data?account_id=1001
POST /api/generate-report {"user_id": 1002}
```

---

### 3.14 — WebSocket IDOR

Real-time data leaks via WebSocket message manipulation.

```
ws://app.com/ws
Send: {"type": "subscribe", "room_id": "9999"}
Receive: private messages from room 9999
```

---

### 3.15 — Second-Order / Stored IDOR

Your malicious reference is stored and later used by the server in a privileged context.

```
Step 1: Set your profile's "linked_account_id" to victim's ID
Step 2: Server later uses that ID to fetch data on your behalf
```

---

## 4. Where IDORs Hide

```
Application Areas to Test:
├── User Profile (view, edit, delete)
├── Account Settings
├── Password Reset / Email Change
├── Orders & Invoices
├── File Uploads & Downloads
├── Messages & Notifications
├── Admin Panels (even limited ones)
├── API Endpoints (REST, GraphQL, WebSocket)
├── Export / Download Features
├── Sharing & Collaboration Features
├── Payment & Billing
├── Search Results
├── Comments & Posts
├── Team / Organization Management
└── Mobile App API Calls
```

**URL patterns that scream IDOR:**
```
/api/users/{id}
/api/accounts/{id}/settings
/documents/{id}/download
/invoices/{id}
/orders/{id}
/messages/thread/{id}
/admin/users/{id}
/v2/profile/{username_or_id}
```

---

## 5. Hunting Methodology

### Phase 1: Reconnaissance

**Step 1 — Map the application**
- Browse every feature as a normal user
- Note every URL, parameter, and ID you encounter
- Create two test accounts: **Account A** (attacker) and **Account B** (victim)

**Step 2 — Identify all object references**
- URL path parameters: `/api/user/1001`
- Query parameters: `?id=1001&order_id=555`
- POST body fields: `{"user_id": 1001}`
- Headers: `X-User-ID: 1001`
- Cookies: `session_user=1001`
- Hidden form fields

**Step 3 — Collect IDs from Account B**
- Log into Account B, copy its IDs (user ID, order IDs, file IDs, etc.)
- Keep a note of all resources that belong to B

---

### Phase 2: Testing

**Step 4 — Switch to Account A**
- Log out of B, log into A
- Replay every request from A, but substitute B's IDs

**Step 5 — Test all HTTP methods**
```
GET    /api/user/B_ID    (read)
POST   /api/user/B_ID    (create in their namespace)
PUT    /api/user/B_ID    (update their data)
PATCH  /api/user/B_ID    (partial update)
DELETE /api/user/B_ID    (delete their resource)
HEAD   /api/user/B_ID    (check existence)
```

**Step 6 — Test unauthenticated access**
- Remove session cookie entirely
- Some endpoints have no authentication at all

---

### Phase 3: Bypass Techniques

**Step 7 — If you get a 403/401, try these bypasses:**

```
# Method override
POST /api/delete/1002
X-HTTP-Method-Override: DELETE

# Wildcard
GET /api/users/*

# Version switching
/api/v1/user/1002  →  /api/v2/user/1002  (older version may lack checks)
/api/v1/user/1002  →  /api/user/1002     (unversioned)

# Case variation
/API/User/1002
/api/USER/1002

# Extension
/api/user/1002.json
/api/user/1002.xml

# Trailing slash
/api/user/1002/

# Parameter pollution
?user_id=1001&user_id=1002

# Type juggling (especially in PHP/JS)
?id=1002&id[]=1002
?id=1002 (string vs int)
```

---

### Phase 4: Automation

**Step 8 — Fuzz ID ranges**
- Use Burp Intruder or ffuf to enumerate IDs
- Target range: current_user_id ± 500
- Look for different response sizes/codes

**Step 9 — Check API documentation**
- `/swagger.json`, `/api-docs`, `/openapi.json`
- GraphQL introspection: `POST /graphql { __schema { types { name fields { name } } } }`

---

## 6. Burp Suite IDOR Workflow

### Setup

1. Launch Burp Suite → Proxy → Intercept ON
2. Set browser to use Burp's proxy (127.0.0.1:8080)
3. Install Burp's CA certificate in browser

---

### Technique 1: Manual Testing with Repeater

```
Step 1: Browse as Account A → capture request in Proxy
Step 2: Right-click request → Send to Repeater
Step 3: In Repeater, change the ID to Account B's resource ID
Step 4: Click Send → check response for B's data
```

**Signs of successful IDOR:**
- Response contains B's data (name, email, files)
- Response is 200 OK instead of 403
- Response size differs significantly (data vs error message)

---

### Technique 2: Burp Intruder for Enumeration

```
Step 1: Capture request → Send to Intruder
Step 2: Highlight the ID value → click Add § (payload marker)
Step 3: Payloads tab → Payload Type: Numbers
        From: 1000  To: 2000  Step: 1
Step 4: Options → uncheck "URL-encode these characters"
Step 5: Start Attack
Step 6: Sort by Response Length — different length = potential IDOR
```

**Pro tip:** Set Grep-Match to extract specific strings (like email patterns) to auto-flag hits.

---

### Technique 3: Burp Comparer

```
Step 1: Make request as Account A → Send response to Comparer
Step 2: Change ID to Account B's → Send response to Comparer
Step 3: Comparer → Word/Byte comparison
→ Differences show what data you shouldn't be seeing
```

---

### Technique 4: Burp Match & Replace (Passive IDOR)

```
Proxy → Options → Match and Replace
Add Rule:
  Type: Request Header
  Match: your_account_b_id
  Replace: account_a_id
→ Burp auto-replaces IDs in ALL requests, helping find hidden references
```

---

### Technique 5: Authorization Extension (must-have)

Install **AuthMatrix** or **Autorize** extension:

```
Step 1: Install from BApp Store → Extensions → BApp Store
Step 2: Autorize:
  - Add Account B's session token as "low privilege"
  - Browse as Account A
  - Autorize automatically replays every request with B's token
  - Flags requests where B gets the same response as A
```

**Autorize is the single most powerful tool for IDOR hunting.**

---

### Technique 6: GraphQL Testing in Burp

```
Step 1: Intercept GraphQL request
Step 2: Send to Repeater
Step 3: Modify the query:

# Introspection
{"query": "{ __schema { types { name } } }"}

# Field discovery
{"query": "{ user(id: \"B_USER_ID\") { email phone creditCard } }"}

# Mutation IDOR
{"query": "mutation { deleteUser(id: \"B_USER_ID\") { success } }"}
```

---

### Technique 7: Automated Scanning with Burp Active Scanner

```
Step 1: Add target to scope
Step 2: Right-click → Actively scan this host
Step 3: Look for "Insecure direct object references" in Issues
Step 4: Manually verify any flagged issues (scanner has false positives)
```

---

## 7. Beginner Techniques

These are the easiest IDORs to find — start here.

### 7.1 — The Classic ID Swap

1. Create two accounts (A and B)
2. With Account B: create an order, upload a file, post a comment — note the ID
3. Switch to Account A
4. Replace your ID with B's ID in the URL
5. Check if you can read/edit/delete B's resource

### 7.2 — Profile Edit IDOR

```
PUT /api/users/1001/profile
{"email": "me@test.com"}

→ Change 1001 to 1002
→ Can you update another user's email?
```

### 7.3 — Order/Invoice Access

```
GET /orders/1234/invoice.pdf   (your order)
→ Try GET /orders/1233/invoice.pdf
→ Try GET /orders/1235/invoice.pdf
```

### 7.4 — Password Reset Token in URL

```
GET /reset-password?token=ABC123&user_id=1001
→ Change user_id to 1002 (does the token work for B?)
```

### 7.5 — Hidden Fields in HTML

Inspect page source for hidden inputs:
```html
<input type="hidden" name="user_id" value="1001">
<input type="hidden" name="account_id" value="5583">
```
Modify these with Burp and replay the form submission.

---

## 8. Intermediate Techniques

### 8.1 — API Version Downgrade

Newer API versions fix authorization, but old ones still exist:
```
/api/v2/user/1002  →  403 Forbidden
/api/v1/user/1002  →  200 OK ← IDOR!
/api/user/1002     →  200 OK ← IDOR!
```

### 8.2 — IDOR in File Downloads

```
POST /api/export
{"report_type": "activity", "user_id": 1002}
→ Returns a download URL
GET /exports/activity_report_1002.csv  ← IDOR
```

### 8.3 — Referrer-Leaked UUIDs

1. Share a resource from Account B
2. Check the referrer/location headers in burp for GUIDs
3. Use that GUID to access B's resource from Account A

### 8.4 — IDOR in Search / Filter Parameters

```
GET /api/messages?sender_id=1001  (yours)
→ GET /api/messages?sender_id=1002  (theirs)

GET /api/transactions?account=ACC-001
→ GET /api/transactions?account=ACC-002
```

### 8.5 — Role Parameter Manipulation

```
POST /api/users/update
{"user_id": 1001, "role": "user"}
→ Change role to "admin"

POST /api/organization/invite
{"email": "me@test.com", "role": "member"}
→ Change role to "owner"
```

### 8.6 — IDOR via Object Reference in Request Body

```
POST /api/transfer
{
  "from_account": "ACC-YOURS",
  "to_account": "ACC-YOURS",
  "amount": 100
}
→ Change "from_account" to another user's account
```

### 8.7 — IDOR in Notifications / Activity Feeds

```
GET /api/notifications?user_id=1002
GET /api/activity?account_id=5583
→ These often forget authorization checks
```

---

## 9. Advanced Techniques

### 9.1 — Race Condition IDOR

Some access control checks happen before the actual operation, and a race can bypass them:

```
Thread 1: POST /api/share-document {"doc_id": B_DOC_ID}
Thread 2: GET /api/documents/B_DOC_ID  (simultaneously)
→ Thread 2 may succeed during the share window
```

Use Burp Turbo Intruder or `ffuf` with parallel threads.

### 9.2 — JWT / Token IDOR

JWT tokens sometimes encode the user_id:
```
eyJhbGciOiJIUzI1NiJ9.eyJ1c2VyX2lkIjoxMDAxfQ...
→ Decode payload: {"user_id": 1001}
→ Change to {"user_id": 1002}
→ If signature isn't verified (alg:none), this works
```

Tools: jwt.io, jwt_tool

### 9.3 — GraphQL IDOR via Alias Batching

```graphql
{
  user1: user(id: "1001") { email }
  user2: user(id: "1002") { email }
  user3: user(id: "1003") { email }
}
```
Batch queries can bypass per-request rate limits while enumerating users.

### 9.4 — IDOR via HTTP Parameter Pollution (HPP)

```
GET /api/user?id=1001&id=1002
POST body: user_id=1001&user_id=1002

# PHP takes the last value, ASP.NET takes all, Express takes first
# Test all combinations to find what the backend does
```

### 9.5 — Indirect IDOR Through Relationships

```
# You can't access /users/1002 directly
# But you can access /teams/5/members which references user 1002's data
GET /teams/5/members  →  leaks all member data including user 1002
```

### 9.6 — IDOR via Mass Assignment (Object Property Injection)

```
PATCH /api/profile
{"name": "Hacker"}

→ Add extra fields:
{"name": "Hacker", "user_id": 1002, "is_admin": true, "account_balance": 99999}
→ Server may assign all provided fields (mass assignment)
```

### 9.7 — Cache Poisoning IDOR

```
GET /api/user/profile?user_id=1001
X-Forwarded-For: 1.2.3.4
→ If response is cached by user_id, another user requesting the same ID
  may get your cached response
```

### 9.8 — IDOR in PDF/Report Generation

```
POST /api/generate-statement
{"account_id": "B_ACCOUNT"}
→ Server generates PDF and returns a link
GET /statements/generated_B_ACCOUNT.pdf
→ Even if the POST fails, the GET may succeed with a predictable filename
```

### 9.9 — IDOR via Redirect Parameter

```
GET /auth/callback?redirect_uri=/dashboard&user=1002
→ Application may redirect and load user 1002's session
```

### 9.10 — IDOR Chains (Multi-Step)

Some IDORs require multiple steps:
```
Step 1: GET /api/users/1002/api-key  →  returns key (IDOR #1)
Step 2: Use that API key to authenticate as user 1002
Step 3: Now authenticated, perform privileged actions (IDOR #2)
Result: Full account takeover
```

---

## 10. Chaining IDOR with Other Bugs

### IDOR + XSS = Account Takeover

```
1. Find IDOR that lets you update another user's profile
2. Inject XSS payload in the name/bio field
3. When victim views their own profile → XSS fires → steals session cookie
```

### IDOR + CSRF

```
1. Find IDOR on a POST endpoint (no CSRF token)
2. Craft CSRF payload with victim's ID
3. Victim visits your page → their data is changed
```

### IDOR + SSRF

```
1. Find IDOR that reads a "URL" field from another user's record
2. Set your own "URL" to an internal server
3. Access victim's ID → server fetches your internal URL
```

### IDOR + Race Condition = Privilege Escalation

```
1. Find IDOR on a "transfer ownership" endpoint
2. Race condition allows you to transfer ownership of an org you don't own
3. You become admin
```

---

## 11. Real HackerOne Reports

These are all publicly disclosed reports on HackerOne's Hacktivity — real bugs, real bounties.

---

### Report 1 — IDOR on HackerOne's Own Platform: Private Report Leak
- **Program:** HackerOne
- **Endpoint:** `POST /bugs.json`
- **Bug:** Any private bug report's details could be accessed by sending a POST request to `/bugs.json` with another report's ID — even from private programs. This exposed program handler info, about sections, and report metadata.
- **Impact:** Disclosure of private vulnerability reports across all programs
- **Status:** Resolved (May 2024)
- **Report URL:** https://hackerone.com/reports/2487889
- **Lesson:** Internal platforms are also in scope. Try the platform you're hunting on.

---

### Report 2 — IDOR to Add Tags to Other Users' Assets (HackerOne)
- **Program:** HackerOne
- **Operation:** `AddTagToAssets` GraphQL mutation
- **Bug:** An authenticated user could call the GraphQL mutation with another user's asset ID and add custom tags to assets they don't own. The mutation lacked ownership verification.
- **Impact:** Disclosure of victim's custom tag structure; potential data manipulation
- **Bounty:** $175 (retest bounty)
- **Status:** Resolved (December 2024)
- **Report URL:** https://hackerone.com/reports/2633771
- **Lesson:** GraphQL mutations are goldmines for IDOR — always test with another user's IDs.

---

### Report 3 — IDOR: Mozilla Firefox Account Deletion via Another User's Session
- **Program:** Mozilla
- **Endpoint:** `DELETE https://api.accounts.firefox.com/v1/account/destroy`
- **Bug:** An attacker using SSO (Google login) could delete another user's Firefox account by providing the victim's email address. The server failed to verify that the session making the deletion request actually belonged to the account being deleted.
- **Impact:** Any authenticated attacker could delete another user's Firefox/Mozilla account
- **Severity:** High
- **Report URL:** https://hackerone.com/reports/3154983
- **Lesson:** Account deletion endpoints often skip the "does this belong to you?" check. Always test destructive actions.

---

### Report 4 — IDOR: SingleStore Scheduled Data Leak via `projectID` Parameter
- **Program:** SingleStore
- **Endpoint:** `GetNotebookScheduledPaginatedJobs`
- **Bug:** By modifying the `projectID` parameter in API requests, an authenticated user could access scheduled job information belonging to other users' projects. No ownership check on the project ID.
- **Impact:** Cross-user data exposure of notebook schedules and job configurations
- **Report URL:** https://hackerone.com/reports/3219944
- **Lesson:** Data-intensive SaaS platforms often have many endpoints that accept project/workspace IDs without proper authorization.

---

### Report 5 — IDOR in Revive Adserver: Delete Another Manager's Banners
- **Program:** Revive Adserver
- **Endpoint:** Banner deletion endpoint
- **Bug:** The code validated access to the parent campaign but failed to check if the banner itself belonged to the requesting manager. Any Manager could delete banners owned by other Managers.
- **Impact:** Unauthorized deletion of advertising assets
- **Status:** Disclosed October 2025
- **Report URL:** https://hackerone.com/reports/3401612
- **Lesson:** Multi-tenant systems with roles often check the parent object but forget to check child objects. Go deep into the object hierarchy.

---

### Report 6 — Chain of IDORs Between Uber's U4B and Vouchers APIs (PII Leak)
- **Program:** Uber
- **Bug:** A chain of IDOR vulnerabilities across Uber's business (U4B) and vouchers APIs allowed attackers to view and modify program/voucher policies and obtain organization employees' PII. One IDOR led to the next in a chain.
- **Impact:** Viewing and modifying business voucher programs; leaking employee PII
- **Upvotes:** 62 upvotes
- **Lesson:** IDORs chain. When you find one IDOR, dig into what that data can be used for to find the next.

---

### Report 7 — IDOR on HackerOne Feedback Review ("What Programs Say")
- **Program:** HackerOne
- **Bug:** A malicious program manager could submit a public review for any hacker on the platform — even hackers who never submitted to that program — by changing the hacker's username parameter in the review submission. The public review appeared on the victim's profile under "What Programs Say."
- **Impact:** Reputation damage; fake reviews on researcher profiles
- **Upvotes:** 77 upvotes
- **Report URL:** https://hackerone.com/reports/262661
- **Lesson:** IDOR doesn't always mean data theft — reputation/business logic impacts count too.

---

### Report 8 — IDOR: Bykea Hardcoded Zombie Endpoint Exposes User Data
- **Program:** Bykea (Pakistani ride-hailing)
- **Bug:** By reverse engineering the Android app, a researcher found a hardcoded legacy endpoint that was no longer actively used but remained accessible. The endpoint had no authorization and exposed user data via direct object reference.
- **Impact:** Exposure of user data via legacy endpoint
- **Report URL:** https://hackerone.com/reports/3085742
- **Lesson:** Mobile app reverse engineering reveals hidden/forgotten endpoints that still work on the backend. Decompile APKs and look for zombie endpoints.

---

### Report 9 — IDOR: Affirm Order Information Exposure
- **Program:** Affirm (Buy Now Pay Later)
- **Bug:** Broken access control allowed an attacker to view order information belonging to other users by manipulating the order identifier in the request.
- **Impact:** Financial order details exposed across users
- **Report URL:** https://hackerone.com/reports/1323406
- **Lesson:** Fintech applications are high-value IDOR targets because the impact is immediately financial.

---

### Report 10 — IDOR on Nextcloud: Access Deleted and Other Users' Photos via Direct URL
- **Program:** Nextcloud
- **Bug:** Photos (including deleted ones) were accessible via direct URL using the photo's ID, without verifying that the requesting user owned or had access to the photo.
- **Impact:** Unauthorized access to private and deleted photos
- **Upvotes:** 30 upvotes
- **Lesson:** File/media storage systems frequently have IDOR in direct download URLs. Always test predictable file paths with another user's resource IDs.

---

## 12. Tools & Extensions

### Essential Tools

| Tool | Use | Link |
|------|-----|-------|
| **Burp Suite (Community/Pro)** | Core proxy, Repeater, Intruder, Scanner | portswigger.net |
| **Autorize** (Burp extension) | Auto-test every request with another user's session | BApp Store |
| **AuthMatrix** (Burp extension) | Matrix-based access control testing | BApp Store |
| **ParamMiner** (Burp extension) | Discover hidden parameters | BApp Store |
| **ffuf** | Fuzzing IDs in URLs | github.com/ffuf/ffuf |
| **jwt_tool** | JWT manipulation for token-based IDOR | github.com/ticarpi/jwt_tool |
| **Arjun** | Discover hidden HTTP parameters | github.com/s0md3v/Arjun |
| **GraphQL Voyager** | Visualize GraphQL schema from introspection | graphql-voyager |
| **InQL** (Burp extension) | GraphQL security testing | BApp Store |

### Browser Extensions

- **HackTools** — Quick encoding/decoding toolbar
- **EditThisCookie** — Cookie editor for session switching
- **Wappalyzer** — Tech stack detection

---

## 13. IDOR Testing Checklist

Use this every time you test a new application.

### Recon
- [ ] Mapped all features and created two accounts (A and B)
- [ ] Collected all IDs from Account B (user ID, object IDs)
- [ ] Identified all ID types: integers, UUIDs, hashed, encoded
- [ ] Checked API docs (`/swagger.json`, `/api-docs`, `/openapi.json`)
- [ ] Tested GraphQL introspection

### Authentication Testing
- [ ] Tested all endpoints without any session (unauthenticated)
- [ ] Tested all endpoints with Account A using Account B's IDs
- [ ] Tested all HTTP methods (GET, POST, PUT, PATCH, DELETE)
- [ ] Tested API version differences (v1 vs v2)

### Parameter Testing
- [ ] URL path parameters
- [ ] Query string parameters
- [ ] POST/PUT body parameters
- [ ] JSON body fields
- [ ] Custom HTTP headers (X-User-ID, X-Account-ID)
- [ ] Hidden HTML form fields
- [ ] Cookie values
- [ ] JWT claims

### Bypass Attempts
- [ ] Parameter pollution (duplicate params)
- [ ] Type juggling (string vs int IDs)
- [ ] Base64/URL encoding the ID
- [ ] Trailing slash / extension manipulation
- [ ] HTTP method override headers
- [ ] Wildcard characters (`*`, `%`)

### Advanced
- [ ] Race condition testing on authorization checks
- [ ] JWT manipulation (alg:none, changing sub claim)
- [ ] GraphQL batch queries
- [ ] WebSocket message manipulation
- [ ] Mobile app decompilation for hidden endpoints
- [ ] Mass assignment on PATCH/PUT endpoints

---

## 14. Writing a Good Report

A clear report gets triaged faster and paid higher. Use this template:

```markdown
# Title: IDOR on [Endpoint] Allows [Impact]

## Summary
An IDOR vulnerability exists at [endpoint] that allows an authenticated user 
to [read/modify/delete] resources belonging to other users by manipulating 
the [parameter name] parameter.

## Severity: [Critical/High/Medium/Low]
Justification: [Why this severity? Data exposed? Financial impact? ATO?]

## Steps to Reproduce
1. Create two accounts: Account A (attacker) and Account B (victim)
2. Log into Account B and [perform action to create resource] — note the ID: [ID_VALUE]
3. Log out of Account B and log into Account A
4. Send the following request:

    GET /api/resource/[ID_VALUE] HTTP/1.1
    Host: target.com
    Cookie: session=ACCOUNT_A_SESSION

5. Observe that the response contains Account B's data

## Proof of Concept
[Screenshot or video showing the attack]
[Show Account A's session and Account B's data in the response]

## Impact
An attacker can [describe real-world impact]:
- Read [specific data type] belonging to any user
- Modify [specific data type] belonging to any user
- [Potential for account takeover / financial fraud / PII exposure]

## Mitigation
Implement server-side authorization checks that verify the requesting user 
owns or has explicit permission to access the requested resource before 
returning or modifying it.
```

### Tips for Better Reports
- Always include the **exact HTTP request and response**
- Show data from **Account B** appearing in **Account A's** context
- Calculate the real impact (how many users affected? what data type?)
- Suggest a clear fix
- Keep the PoC as simple as possible

---

## 15. Quick Reference Cheat Sheet

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
IDOR CHEAT SHEET — PRINT THIS OUT
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

ALWAYS HAVE TWO ACCOUNTS. Account A = attacker, Account B = victim.

WHERE TO LOOK:
  /api/{resource}/{id}       → swap ID
  ?id=X&order_id=Y           → swap all ID params
  POST body JSON             → swap IDs in body
  X-User-ID / X-Account-ID  → swap header values
  JWT payload (sub, id)      → decode, change, re-encode

BYPASS 403:
  Add ?user_id=own&user_id=victim (HPP)
  Try /api/v1/ instead of /api/v2/
  Try method override: X-HTTP-Method-Override: DELETE
  Try path: /api/users/victim/../victim
  Try encoding: %31%30%30%32 (URL encode the ID)
  Try type confusion: "1002" vs 1002

BURP WORKFLOW:
  1. Capture request with A's session
  2. Send to Repeater
  3. Replace ID with B's ID
  4. Send → check for B's data
  OR install Autorize → browses as A, tests with B's cookie automatically

IMPACT MULTIPLIERS (higher bounty):
  → Account takeover = CRITICAL
  → PII exposure (name, email, phone, address) = HIGH
  → Financial data (payment, orders, invoices) = HIGH
  → Read-only on non-sensitive = LOW/MEDIUM

GRAPHQL:
  Introspect: {"query":"{ __schema { types { name fields { name } } } }"}
  IDOR: {"query":"{ user(id: \"VICTIM_ID\") { email phone } }"}

MOBILE:
  Decompile APK with jadx or apktool
  Look for hardcoded endpoints and API calls
  Test all endpoints found in source code

REPORT TITLE FORMAT:
  "IDOR on /api/[endpoint] allows [action] of [impact]"

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

## Appendix: Useful IDOR Resources

- **PortSwigger Web Academy — Access Control Labs:** https://portswigger.net/web-security/access-control
- **HackerOne Hacktivity (filter by IDOR):** https://hackerone.com/hacktivity
- **Top IDOR Reports (GitHub):** https://github.com/reddelexc/hackerone-reports/blob/master/tops_by_bug_type/TOPIDOR.md
- **Nahamsec's IDOR Tips:** https://youtube.com/@NahamSec
- **InsiderPhD's IDOR Guide:** https://youtube.com/@InsiderPhD
- **PentesterLab (IDOR exercises):** https://pentesterlab.com

---

*Happy Hunting. Find bugs, stay ethical, report responsibly.*
*Remember: Only test on programs that have given permission (bug bounty programs, CTFs, your own test environments).*
```

---

**Document Version:** 1.0  
**Last Updated:** June 2026  
**Scope:** Ethical bug bounty hunting on authorized programs only
