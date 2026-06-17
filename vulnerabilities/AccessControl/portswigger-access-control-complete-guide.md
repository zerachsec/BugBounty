# PortSwigger Web Security Academy
# 🔐 Access Control Vulnerabilities — Complete Lab Guide

> **Goal:** By the end of this guide you will understand every class of access control vulnerability, know how to find them during a pentest or bug bounty, and be able to solve every lab in the Access Control module from first principles.

---

## Table of Contents

1. [Prerequisites & Setup](#1-prerequisites--setup)
2. [What Is Access Control?](#2-what-is-access-control)
3. [Vulnerability Categories — Mental Map](#3-vulnerability-categories--mental-map)
4. [Lab 01 — Unprotected Admin Functionality](#lab-01--unprotected-admin-functionality-apprentice)
5. [Lab 02 — Unprotected Admin Functionality with Unpredictable URL](#lab-02--unprotected-admin-functionality-with-unpredictable-url-apprentice)
6. [Lab 03 — User Role Controlled by Request Parameter](#lab-03--user-role-controlled-by-request-parameter-apprentice)
7. [Lab 04 — User Role Can Be Modified in User Profile](#lab-04--user-role-can-be-modified-in-user-profile-apprentice)
8. [Lab 05 — User ID Controlled by Request Parameter](#lab-05--user-id-controlled-by-request-parameter-apprentice)
9. [Lab 06 — User ID Controlled by Request Parameter with Unpredictable User IDs](#lab-06--user-id-controlled-by-request-parameter-with-unpredictable-user-ids-apprentice)
10. [Lab 07 — User ID Controlled by Request Parameter with Data Leakage in Redirect](#lab-07--user-id-controlled-by-request-parameter-with-data-leakage-in-redirect-apprentice)
11. [Lab 08 — User ID Controlled by Request Parameter with Password Disclosure](#lab-08--user-id-controlled-by-request-parameter-with-password-disclosure-apprentice)
12. [Lab 09 — Insecure Direct Object References (IDOR)](#lab-09--insecure-direct-object-references-apprentice)
13. [Lab 10 — URL-Based Access Control Can Be Circumvented](#lab-10--url-based-access-control-can-be-circumvented-practitioner)
14. [Lab 11 — Method-Based Access Control Can Be Circumvented](#lab-11--method-based-access-control-can-be-circumvented-practitioner)
15. [Lab 12 — Multi-Step Process with No Access Control on One Step](#lab-12--multi-step-process-with-no-access-control-on-one-step-practitioner)
16. [Lab 13 — Referer-Based Access Control](#lab-13--referer-based-access-control-practitioner)
17. [Bug Hunting Methodology for Access Control](#bug-hunting-methodology-for-access-control)
18. [Common Payloads & Cheat Sheet](#common-payloads--cheat-sheet)
19. [How to Prevent Access Control Vulnerabilities](#how-to-prevent-access-control-vulnerabilities)

---

## 1. Prerequisites & Setup

### Tools You Need

| Tool | Purpose | Free? |
|------|---------|-------|
| Burp Suite Community Edition | Intercept, repeat, and manipulate HTTP requests | ✅ Yes |
| Browser (Firefox recommended) | Configured to proxy through Burp | ✅ Yes |
| PortSwigger Academy Account | Access the labs | ✅ Yes |

### Burp Suite Basics

1. **Open Burp Suite** → Proxy tab → Intercept is on.
2. In Firefox, set manual proxy to `127.0.0.1:8080`.
3. Install Burp's CA certificate: browse to `http://burpsuite` and download/install it so HTTPS works.
4. **Repeater** (Ctrl+R) — resend and modify individual requests.
5. **Intruder** (Ctrl+I) — automated payload injection (rate-limited in Community Edition).
6. **HTTP History** — review every request the browser made.

> 💡 **Tip:** In Community Edition, Intruder is throttled. You can use Burp's "Send to Repeater" and manually change values, or use Python `requests` to script attacks.

---

## 2. What Is Access Control?

Access control answers the question: **"Is this user allowed to do this thing?"**

It depends on three pillars that work together:

```
Authentication  →  Session Management  →  Access Control
(Who are you?)     (Are you still you?)    (What can you do?)
```

### Three Types of Access Control

#### Vertical Access Control
Restricts *sensitive functionality* to specific user roles.
- Admin can delete users; regular users cannot.
- A bug here = **Vertical Privilege Escalation** (you gain capabilities above your role).

#### Horizontal Access Control
Restricts users to *their own resources only*.
- You can see your own bank account, not someone else's.
- A bug here = **Horizontal Privilege Escalation** (you access another user's data at the same privilege level).

#### Context-Dependent Access Control
Restricts actions based on *application state*.
- You can't modify a shopping cart after paying.
- A bug here = **Business Logic Bypass**.

---

## 3. Vulnerability Categories — Mental Map

```
ACCESS CONTROL VULNERABILITIES
│
├── VERTICAL PRIVILEGE ESCALATION
│   ├── Unprotected admin URLs (no auth check at all)
│   ├── URL discovered via robots.txt / JS source
│   ├── Parameter-based role control (admin=true, role=1)
│   ├── Platform misconfiguration
│   │   ├── X-Original-URL / X-Rewrite-URL header bypass
│   │   └── HTTP method switch (POST → GET)
│   └── URL-matching discrepancies (trailing slash, case)
│
├── HORIZONTAL PRIVILEGE ESCALATION
│   ├── IDOR via numeric ID (?id=123 → ?id=124)
│   ├── IDOR via GUID (find other users' GUIDs in app)
│   └── Data leakage in redirects (302 with sensitive body)
│
├── HORIZONTAL → VERTICAL ESCALATION
│   └── Access admin account via IDOR → steal/change password
│
├── INSECURE DIRECT OBJECT REFERENCES (IDOR)
│   ├── Direct DB object references (customer_number=132355)
│   └── Static file references (/download?file=transcript_1.txt)
│
├── MULTI-STEP PROCESS VULNERABILITIES
│   └── Skip step 1 & 2, submit step 3 directly
│
└── REFERER-BASED ACCESS CONTROL
    └── Forge Referer: https://site.com/admin header
```

---

## Lab 01 — Unprotected Admin Functionality (APPRENTICE)

**URL:** `https://portswigger.net/web-security/access-control/lab-unprotected-admin-functionality`

### Concept
The admin panel exists at a URL with no authentication check whatsoever. Any user who discovers the URL gets full admin access.

### Vulnerability Explanation
Developers sometimes assume that if they don't link to an admin page from the regular UI, users won't find it. This is **security through obscurity** — not real security. The URL can be discovered through:
- `robots.txt` (tells search bots what NOT to index — ironically revealing secret paths)
- Directory brute-forcing (tools like `gobuster`, `dirsearch`)
- Source code leakage

### Step-by-Step Solution

**Step 1:** Open the lab. Browse to the homepage.

**Step 2:** Navigate to `robots.txt`:
```
https://<YOUR-LAB-ID>.web-security-academy.net/robots.txt
```
You will see something like:
```
User-agent: *
Disallow: /administrator-panel
```

**Step 3:** Browse directly to:
```
https://<YOUR-LAB-ID>.web-security-academy.net/administrator-panel
```

**Step 4:** The admin panel loads with no login required. Click **Delete** next to `carlos`.

✅ Lab solved.

### What to Learn
- Always check `robots.txt` during recon.
- An unauthenticated admin panel is a **P1 Critical** bug on any bug bounty program.
- Fix: Enforce server-side authentication on every admin route.

---

## Lab 02 — Unprotected Admin Functionality with Unpredictable URL (APPRENTICE)

**URL:** `https://portswigger.net/web-security/access-control/lab-unprotected-admin-functionality-with-unpredictable-url`

### Concept
The admin URL is randomised (e.g., `/admin-abc123`) so it won't appear in `robots.txt` or be easily brute-forced. However, the URL is **leaked in the JavaScript source code**.

### Vulnerability Explanation
The server-side code generates the admin URL dynamically and embeds it in client-side JavaScript to conditionally show the link — but JavaScript is sent to ALL users, including non-admins. Anyone who reads the page source can find the URL.

### Step-by-Step Solution

**Step 1:** Open the lab. Right-click on the homepage → **View Page Source** (or Ctrl+U).

**Step 2:** Search (Ctrl+F) for `admin`. You will find something like:
```javascript
var isAdmin = false;
if (isAdmin) {
    var adminPanelTag = document.createElement('a');
    adminPanelTag.setAttribute('href', '/admin-7gx3ab');
    adminPanelTag.innerText = 'Admin panel';
    document.getElementById('adminpanel').appendChild(adminPanelTag);
}
```

**Step 3:** Copy the admin URL from the `href` attribute (e.g., `/admin-7gx3ab`).

**Step 4:** Browse to that URL directly:
```
https://<YOUR-LAB-ID>.web-security-academy.net/admin-7gx3ab
```

**Step 5:** Delete user `carlos`.

✅ Lab solved.

### What to Learn
- Never put sensitive URLs, tokens, or role-checks in client-side JavaScript.
- During recon: always read page source, look for JS files, use browser DevTools → Sources tab.
- Bug bounty tip: grep JS files for `admin`, `internal`, `secret`, `token`, `password`, `endpoint`.

---

## Lab 03 — User Role Controlled by Request Parameter (APPRENTICE)

**URL:** `https://portswigger.net/web-security/access-control/lab-user-role-controlled-by-request-parameter`

### Concept
After login, the application sets an `Admin` cookie to `false`. The app reads this cookie to determine whether to grant admin access. Because cookies are user-controllable, any user can change `Admin=false` to `Admin=true`.

### Vulnerability Explanation
Trust is placed on a **client-side value** (cookie/parameter) to make an authorization decision. The correct approach is to check role from the server-side session or database.

### Step-by-Step Solution

**Step 1:** Open the lab. Log in with credentials `wiener:peter`.

**Step 2:** In Burp Suite, go to **Proxy → HTTP History**. Find the request after login and observe a cookie like:
```
Cookie: session=<token>; Admin=false
```

**Step 3:** Try browsing to `/admin` — you'll be denied.

**Step 4:** Send any request to **Repeater**. Modify the cookie:
```
Cookie: session=<token>; Admin=true
```
Change the URL to `/admin` and send.

**Step 5:** You now see the admin panel. Send the delete request for `carlos`:
```
GET /admin/delete?username=carlos HTTP/1.1
Cookie: session=<token>; Admin=true
```

✅ Lab solved.

### What to Learn
- **Never** use user-controllable values (cookies, URL parameters, hidden form fields) to make authorization decisions.
- Bug bounty tip: Look for cookies or parameters named `role`, `admin`, `isAdmin`, `privilege`, `level`, `type`, `group`.
- Fix: Store role in server-side session, never in a client-visible location.

---

## Lab 04 — User Role Can Be Modified in User Profile (APPRENTICE)

**URL:** `https://portswigger.net/web-security/access-control/lab-user-role-can-be-modified-in-user-profile`

### Concept
The application has a profile update endpoint that returns the user's role in the JSON response. An attacker can add a `roleid` field to the request body to escalate privileges.

### Vulnerability Explanation
The server accepts and processes extra JSON fields from the client that it shouldn't. This is a **mass assignment** vulnerability combined with broken access control.

### Step-by-Step Solution

**Step 1:** Log in as `wiener:peter`. Go to **Account** and update your email address. Intercept the request in Burp.

**Step 2:** The request body looks like:
```json
{"email":"newemail@example.com"}
```
The response includes:
```json
{"username":"wiener","email":"newemail@example.com","roleid":1}
```

**Step 3:** Send the request to Repeater. Inject an extra field in the JSON body:
```json
{"email":"newemail@example.com","roleid":2}
```

**Step 4:** Send the request. The response confirms:
```json
{"username":"wiener","email":"newemail@example.com","roleid":2}
```
Your role is now `2` (admin).

**Step 5:** Navigate to `/admin` and delete `carlos`.

✅ Lab solved.

### What to Learn
- This is a **Mass Assignment** vulnerability — the server blindly accepts all JSON keys.
- Bug bounty tip: When you see JSON responses with role/privilege fields, try adding that field to update requests.
- Fix: Whitelist only expected fields on the server; never process `roleid` from user input.

---

## Lab 05 — User ID Controlled by Request Parameter (APPRENTICE)

**URL:** `https://portswigger.net/web-security/access-control/lab-user-id-controlled-by-request-parameter`

### Concept
The account page URL contains a `?id=wiener` parameter. Changing this to another username retrieves that user's account page and API key.

### Vulnerability Explanation
This is a classic **IDOR (Insecure Direct Object Reference)** / horizontal privilege escalation. The server uses the user-supplied `id` parameter directly to look up the account, without verifying the session belongs to that user.

### Step-by-Step Solution

**Step 1:** Log in as `wiener:peter`. Go to **My Account** — the URL is:
```
/my-account?id=wiener
```

**Step 2:** In the URL bar (or Burp Repeater), change `id=wiener` to `id=carlos`:
```
/my-account?id=carlos
```

**Step 3:** The page now shows carlos's account, including his **API key**.

**Step 4:** Submit carlos's API key as the solution.

✅ Lab solved.

### What to Learn
- Every time you see an ID, username, number, or reference in a URL or request — try changing it.
- The server must check: "Does the current session owner match the requested resource owner?"
- This class of bug is among the most common on bug bounty platforms (HackerOne reports it as one of the top vulnerability types every year).

---

## Lab 06 — User ID Controlled by Request Parameter with Unpredictable User IDs (APPRENTICE)

**URL:** `https://portswigger.net/web-security/access-control/lab-user-id-controlled-by-request-parameter-with-unpredictable-user-ids`

### Concept
The app uses GUIDs (e.g., `f47ac10b-58cc-4372-a567-0e02b2c3d479`) instead of sequential integers as user IDs. You can't guess carlos's GUID — but it's **disclosed elsewhere in the application** (in a blog post authored by carlos).

### Vulnerability Explanation
Using UUIDs/GUIDs does add guessing difficulty, but if those IDs are exposed anywhere in the application (public profiles, posts, comments, search results), the protection is bypassed. Security must not depend on the secrecy of an identifier.

### Step-by-Step Solution

**Step 1:** Browse the blog/posts on the site. Find a post authored by **carlos** and click on his username.

**Step 2:** The URL of his profile page reveals his GUID:
```
/blogs?userId=f47ac10b-58cc-4372-a567-0e02b2c3d479
```
Copy the GUID.

**Step 3:** Log in as `wiener:peter`. Go to your account page:
```
/my-account?id=<your-guid>
```

**Step 4:** Replace your GUID with carlos's GUID:
```
/my-account?id=f47ac10b-58cc-4372-a567-0e02b2c3d479
```

**Step 5:** The page shows carlos's API key. Submit it.

✅ Lab solved.

### What to Learn
- GUIDs are **not a security control** — they are just identifiers.
- During recon, look for user IDs in: profile pages, comments, posts, API responses, email notifications.
- Fix: Enforce proper authorization checks regardless of how "unguessable" the ID appears.

---

## Lab 07 — User ID Controlled by Request Parameter with Data Leakage in Redirect (APPRENTICE)

**URL:** `https://portswigger.net/web-security/access-control/lab-user-id-controlled-by-request-parameter-with-data-leakage-in-redirect`

### Concept
When you try to access another user's account page, the server redirects you (302) to the login page — but the **redirect response body still contains the sensitive data** before the browser follows the redirect.

### Vulnerability Explanation
Developers implement a redirect when access is denied, thinking the user won't see the protected content. However, the HTTP response with the 302 status code can still contain a body with sensitive data — and tools like Burp Suite show you the raw response before the redirect is followed.

### Step-by-Step Solution

**Step 1:** Log in as `wiener:peter`. Browse to your account page:
```
/my-account?id=wiener
```

**Step 2:** In Burp Repeater, change `id=wiener` to `id=carlos` and send.

**Step 3:** The response is a **302 redirect** to `/login`. However, look at the **response body** — it contains carlos's API key!
```html
HTTP/1.1 302 Found
Location: /login
...

<div>Your API Key is: abc123xyz...</div>
```

**Step 4:** Copy the API key from the response body and submit it.

✅ Lab solved.

### What to Learn
- A redirect does NOT mean the server returned no data. Always check the body of 302/301 responses in Burp.
- Bug bounty tip: Whenever a request redirects you away, intercept the redirect response in Burp and read the body.
- Fix: Perform the authorization check *before* generating the page content, not after.

---

## Lab 08 — User ID Controlled by Request Parameter with Password Disclosure (APPRENTICE)

**URL:** `https://portswigger.net/web-security/access-control/lab-user-id-controlled-by-request-parameter-with-password-disclosure`

### Concept
The account page renders the user's **current password** in a pre-filled password field. By accessing the admin's account page via IDOR, you can steal the admin's password and log in.

### Vulnerability Explanation
This is a **horizontal → vertical privilege escalation chain**:
1. Horizontal IDOR: access the admin's account page.
2. Password disclosure: steal the admin password from the rendered HTML.
3. Log in as admin: full vertical escalation achieved.

### Step-by-Step Solution

**Step 1:** Log in as `wiener:peter`. Go to your account page. Observe that the password field is pre-filled (inspect it — the `value` attribute contains your password).

**Step 2:** In Burp Repeater, send a request to:
```
GET /my-account?id=administrator HTTP/1.1
Cookie: session=<wiener's session>
```

**Step 3:** In the response, find the password field:
```html
<input type="password" name="password" value="s3cr3tAdminP4ss">
```

**Step 4:** Log out. Log in as `administrator` with the discovered password.

**Step 5:** Go to the admin panel (`/admin`) and delete `carlos`.

✅ Lab solved.

### What to Learn
- Never pre-fill password fields from the server — use empty fields or let the browser handle it via autocomplete.
- IDOR + information disclosure can chain into full account takeover.
- This exact bug pattern exists on real-world apps. Always check for pre-filled sensitive fields.

---

## Lab 09 — Insecure Direct Object References (APPRENTICE)

**URL:** `https://portswigger.net/web-security/access-control/lab-insecure-direct-object-references`

### Concept
The application stores chat transcripts as static files on the server with sequential numeric filenames (`/download-transcript/1.txt`, `2.txt`, etc.). By iterating through filenames, you can access other users' chat logs.

### Vulnerability Explanation
The server directly exposes internal storage references (filenames) to users. There is no authorization check confirming the requesting user owns the file. This is a **direct reference to a static file** IDOR.

### Step-by-Step Solution

**Step 1:** In the lab, go to the **Live chat** feature. Send a message, then click **View transcript** and download your transcript. Intercept the download request in Burp.

**Step 2:** The request is:
```
GET /download-transcript/2.txt HTTP/1.1
```

**Step 3:** In Repeater, change the filename to `1.txt`:
```
GET /download-transcript/1.txt HTTP/1.1
```

**Step 4:** The response contains another user's chat transcript with their **password** visible in the conversation.

**Step 5:** Log out. Log in as `carlos` with the password found in the transcript.

**Step 6:** Go to **My Account** to complete the lab.

✅ Lab solved.

### What to Learn
- Sequential/predictable filenames are a classic IDOR vector.
- Bug bounty tip: Any time you download a file with a numeric/predictable name — decrement or increment the number.
- Fix: Use access-controlled download endpoints that verify ownership; store files with random/unguessable names (UUIDs) in non-web-accessible directories.

---

## Lab 10 — URL-Based Access Control Can Be Circumvented (PRACTITIONER)

**URL:** `https://portswigger.net/web-security/access-control/lab-url-based-access-control-can-be-circumvented`

### Concept
The front-end firewall/middleware blocks direct access to `/admin`. However, the back-end application supports the non-standard header `X-Original-URL`, which overrides the URL it processes — allowing you to bypass the front-end restriction.

### Vulnerability Explanation
Some reverse proxies, CDNs, and load balancers enforce access controls based on the URL path in the HTTP request line. But some back-end frameworks (e.g., certain Java EE, PHP frameworks) support override headers (`X-Original-URL`, `X-Rewrite-URL`) that tell them to process a *different* path. The front-end sees `/`, passes it through — but the back-end processes `/admin`.

### Step-by-Step Solution

**Step 1:** Try to browse to `/admin` — you get blocked (403 Forbidden).

**Step 2:** In Burp Repeater, send:
```
GET / HTTP/1.1
Host: <YOUR-LAB-ID>.web-security-academy.net
X-Original-URL: /invalid
```
Response: 404 Not Found → confirms the back-end is reading `X-Original-URL`.

**Step 3:** Change the header value to `/admin`:
```
GET / HTTP/1.1
Host: <YOUR-LAB-ID>.web-security-academy.net
X-Original-URL: /admin
```
Response: 200 OK with the admin panel.

**Step 4:** To delete `carlos`, you need to pass the query parameter via the actual request line (the back-end framework reads query strings from the real request line):
```
GET /?username=carlos HTTP/1.1
Host: <YOUR-LAB-ID>.web-security-academy.net
X-Original-URL: /admin/delete
```

✅ Lab solved.

### What to Learn
- Test `X-Original-URL`, `X-Rewrite-URL`, `X-Forwarded-For`, `X-Host` on all protected endpoints.
- This is particularly relevant when you see a "plain" block page — a simple response likely comes from a front-end WAF/proxy, not the application itself.
- Fix: Perform access control in the application layer, not only at the proxy/CDN layer.

---

## Lab 11 — Method-Based Access Control Can Be Circumvented (PRACTITIONER)

**URL:** `https://portswigger.net/web-security/access-control/lab-method-based-access-control-can-be-circumvented`

### Concept
The admin function to promote a user to admin is a `POST` request to `/admin-roles`. The platform-level access control blocks `POST` for non-admins — but not `GET`. By converting the request to `GET`, you bypass the access control.

### Vulnerability Explanation
Some frameworks and middleware allow you to configure per-method access controls: "DENY POST to /admin for non-admins." If the developer forgets to restrict `GET` (or other methods) on the same endpoint, and the application processes `GET` parameters too, the restriction is bypassed.

### Step-by-Step Solution

**Step 1:** Log in as `administrator:admin`. Go to the admin panel and promote a user. Intercept the request in Burp — it looks like:
```
POST /admin-roles HTTP/1.1
Cookie: session=<admin-session>

username=carlos&action=upgrade
```
Send this to Repeater.

**Step 2:** Open a new incognito browser. Log in as `wiener:peter`.

**Step 3:** In the Repeater tab (with the admin's promote request), replace the admin session cookie with wiener's session cookie. Send it — you get `401 Unauthorized`.

**Step 4:** Change the method from `POST` to `POSTX` — the response changes to "missing parameter", which means it's hitting a different code path.

**Step 5:** Right-click the request in Repeater → **Change request method** (converts to GET with parameters in the URL):
```
GET /admin-roles?username=wiener&action=upgrade HTTP/1.1
Cookie: session=<wiener-session>
```

**Step 6:** Send the GET request — it succeeds! Wiener is now promoted to admin.

**Step 7:** Use wiener's browser to navigate to `/admin` and delete `carlos`.

✅ Lab solved.

### What to Learn
- Access control rules must cover **all HTTP methods** — not just `POST`.
- Always try `GET`, `PUT`, `PATCH`, `DELETE`, `HEAD`, `OPTIONS`, `TRACE` on protected endpoints.
- Try also custom verbs: `POSTX`, `FOO` — some frameworks fall through to default handlers.
- Fix: Enforce method-specific controls for every sensitive action, or better: don't rely on method-based controls at all — enforce role checks in the application code.

---

## Lab 12 — Multi-Step Process with No Access Control on One Step (PRACTITIONER)

**URL:** `https://portswigger.net/web-security/access-control/lab-multi-step-process-with-no-access-control-on-one-step`

### Concept
The admin user promotion workflow has three steps. Steps 1 and 2 check for admin privileges, but Step 3 (the confirmation step) does not. You can skip straight to Step 3.

### Vulnerability Explanation
Multi-step workflows often have access control on the first step, with the assumption that if you reach later steps, you already passed the earlier checks. This is wrong — HTTP is stateless, and any step can be accessed directly by crafting the right request.

### Step-by-Step Solution

**Step 1:** Log in as `administrator:admin`. Navigate through the full promotion workflow for any user. Capture **all steps** in Burp HTTP History.

The steps typically look like:
- **Step 1:** `GET /admin` → Admin clicks "upgrade" button
- **Step 2:** `POST /admin-roles` with `action=upgrade&username=carlos` → Confirmation page shown
- **Step 3:** `POST /admin-roles` with `action=upgrade&username=carlos&confirmed=true` → Action executed

**Step 2:** Now log in as `wiener:peter` in an incognito browser. Copy wiener's session cookie.

**Step 3:** In Burp Repeater, take the **Step 3** request (the final confirmation POST) and replace the admin session with wiener's session cookie:
```
POST /admin-roles HTTP/1.1
Cookie: session=<wiener-session>

action=upgrade&confirmed=true&username=wiener
```

**Step 4:** Send. Wiener is now an admin.

**Step 5:** Use wiener's browser to visit `/admin` and delete `carlos`.

✅ Lab solved.

### What to Learn
- **Every step** in a multi-step process must independently enforce access control.
- Never trust that the user "got to this step legitimately" — check permissions at every step server-side.
- Bug bounty tip: In any multi-step flow (checkout, password reset, profile update, admin action) — try skipping to the last step directly with a low-privilege session.

---

## Lab 13 — Referer-Based Access Control (PRACTITIONER)

**URL:** `https://portswigger.net/web-security/access-control/lab-referer-based-access-control`

### Concept
The admin panel at `/admin` is properly protected. However, admin sub-pages like `/admin-roles` only check the `Referer` header — if it says `Referer: https://.../admin`, the request is allowed. Since `Referer` is a client-supplied header, you can forge it.

### Vulnerability Explanation
The `Referer` header indicates which page the user came from. Developers sometimes use it as a proxy for "the user legitimately navigated here from the admin panel." But `Referer` is just another HTTP header — any client can set it to any value.

### Step-by-Step Solution

**Step 1:** Log in as `administrator:admin`. Promote a user via the admin panel. In Burp HTTP History, find the request:
```
POST /admin-roles?username=carlos&action=upgrade HTTP/1.1
Referer: https://<YOUR-LAB-ID>.web-security-academy.net/admin
Cookie: session=<admin-session>
```

**Step 2:** Open incognito, log in as `wiener:peter`. Copy wiener's session.

**Step 3:** In Burp Repeater, use the promotion request but replace the session cookie:
```
GET /admin-roles?username=wiener&action=upgrade HTTP/1.1
Referer: https://<YOUR-LAB-ID>.web-security-academy.net/admin
Cookie: session=<wiener-session>
```

**Step 4:** Send — it succeeds because the `Referer` header passes the check.

**Step 5:** Wiener is now admin. Delete `carlos`.

✅ Lab solved.

### What to Learn
- **Never** use the `Referer` header as a security control. It can be forged by any client.
- Same applies to `Origin`, `X-Forwarded-For` (for IP allowlisting), and similar client-controlled headers.
- Bug bounty tip: If you see a 403 on a sub-page but the parent page is also protected, try adding a `Referer` pointing to the parent.
- Fix: Use proper server-side session-based role checking on every endpoint.

---

## Bug Hunting Methodology for Access Control

Use this systematic approach on any real target:

### Phase 1: Recon & Mapping

```
1. Spider the app thoroughly (Burp Target → Site Map → Spider)
2. Check robots.txt
3. Check JS files for hardcoded paths, role checks, admin URLs
4. Look at HTML source for hidden fields, comments
5. Review all API endpoints in Burp HTTP History
6. Note every parameter that looks like an ID, role, or permission flag
```

### Phase 2: Authentication & Role Testing

```
For each endpoint / function:
│
├── Test as: unauthenticated (no cookie)
├── Test as: low-privilege user
├── Test as: different low-privilege user
└── Test as: admin (to understand what's supposed to be restricted)

Compare responses — look for differences in status codes AND content
```

### Phase 3: Parameter Tampering

```
Cookies:    admin=false → admin=true
            role=user → role=admin
            isAdmin=0 → isAdmin=1

URL params: ?id=123 → ?id=124 (IDOR)
            ?user=wiener → ?user=carlos
            ?admin=false → ?admin=true

JSON body:  Add {"roleid":2} or {"admin":true}
            Try {"role":"admin"}

Path:       /my-account/wiener → /my-account/carlos
            /api/v1/users/123 → /api/v1/users/124
```

### Phase 4: HTTP Method & Header Testing

```
For every restricted endpoint, try:
- GET instead of POST (and vice versa)
- HEAD, PUT, PATCH, DELETE, OPTIONS
- POSTX (custom non-standard method)

Try override headers:
- X-Original-URL: /admin
- X-Rewrite-URL: /admin
- X-Forwarded-For: 127.0.0.1
- Referer: https://target.com/admin

Try URL variations:
- /Admin (capitalisation)
- /admin/ (trailing slash)
- /admin.json (suffix)
- /admin%2F (URL encoding)
```

### Phase 5: Multi-Step & State Testing

```
For every multi-step process:
1. Map all steps and their requests
2. Try accessing later steps directly with a low-priv session
3. Try reordering steps
4. Try skipping confirmation steps
5. Check 302 redirect response bodies for sensitive data
```

### Phase 6: IDOR Deep Dive

```
For every object reference (ID, filename, slug, GUID):
1. Note the format (numeric, UUID, hash, username)
2. If numeric: try ±1, 0, negative numbers, very large numbers
3. If UUID: search the rest of the app for other UUIDs
4. If hash/slug: try to decode or find the source
5. Test in ALL HTTP methods
6. Test in ALL request locations (URL, body, headers, cookies)
7. Check response for other users' data
8. Check 302 redirect bodies
```

---

## Common Payloads & Cheat Sheet

### Headers to Try
```http
X-Original-URL: /admin
X-Rewrite-URL: /admin
X-Custom-IP-Authorization: 127.0.0.1
X-Forwarded-For: 127.0.0.1
X-Remote-IP: 127.0.0.1
X-Remote-Addr: 127.0.0.1
X-Host: localhost
Referer: https://TARGET/admin
```

### Parameter Names Indicating Roles/Privileges
```
admin, isAdmin, is_admin, Admin
role, roleid, role_id, userRole, user_role
privilege, privilegeLevel
level, accessLevel, access_level
type, userType, user_type
group, userGroup
permission, perms
```

### Common IDOR Parameter Names
```
id, userId, user_id, uid
accountId, account_id
customerId, customer_id
orderId, order_id
fileId, file_id, doc, document
profileId, profile_id
postId, post_id
```

### HTTP Methods to Test on Restricted Endpoints
```
GET, POST, PUT, PATCH, DELETE, HEAD, OPTIONS, TRACE, CONNECT
POSTX, GETX, FOO (custom/invalid — may bypass some controls)
```

---

## How to Prevent Access Control Vulnerabilities

Understanding the fix makes you a better attacker AND prepares you for interviews:

| Vulnerability | Fix |
|--------------|-----|
| Unprotected admin URL | Enforce authentication middleware on ALL admin routes |
| URL leaked in JS | Never embed sensitive paths in client-side code; generate URLs server-side per role |
| Parameter-based role control | Store role in server-side session, never in cookies or URL params |
| Mass assignment (roleid in JSON) | Whitelist accepted fields; never bind untrusted input directly to user model |
| IDOR (numeric IDs) | Check resource ownership server-side on every request |
| IDOR (GUIDs) | Still enforce authorization — don't rely on secrecy of IDs |
| Redirect leaking data | Perform auth check BEFORE generating page content |
| Password in profile | Never render passwords in HTML — use masked fields, never pre-fill from server |
| X-Original-URL bypass | Do access control in the application layer, not just at the proxy |
| HTTP method bypass | Restrict ALL methods on sensitive endpoints; check role in app code |
| Multi-step bypass | Every step must independently verify session role |
| Referer-based control | Never use Referer as a security check — forge-able by any client |

### Defence-in-Depth Principles

1. **Deny by default** — if access isn't explicitly granted, it's denied.
2. **Single mechanism** — use one consistent access-control framework across the app.
3. **Server-side enforcement** — access decisions are never delegated to the client.
4. **Least privilege** — users get only the minimum permissions needed.
5. **Audit and test** — automated and manual testing of every protected route.

---

## Quick Reference: Lab → Vulnerability Mapping

| # | Lab Name | Difficulty | Category |
|---|----------|-----------|----------|
| 01 | Unprotected admin functionality | 🟢 Apprentice | Vertical escalation — unprotected URL |
| 02 | Unprotected admin with unpredictable URL | 🟢 Apprentice | Vertical escalation — URL in JS |
| 03 | User role controlled by request parameter | 🟢 Apprentice | Vertical escalation — cookie tampering |
| 04 | User role modifiable in profile | 🟢 Apprentice | Vertical escalation — mass assignment |
| 05 | User ID via request parameter | 🟢 Apprentice | Horizontal escalation — IDOR (username) |
| 06 | User ID via parameter + unpredictable IDs | 🟢 Apprentice | Horizontal escalation — IDOR (GUID) |
| 07 | User ID via parameter + data leak in redirect | 🟢 Apprentice | Horizontal escalation — 302 body leak |
| 08 | User ID via parameter + password disclosure | 🟢 Apprentice | Horizontal → Vertical escalation |
| 09 | Insecure direct object references | 🟢 Apprentice | IDOR — static file reference |
| 10 | URL-based access control circumvented | 🟡 Practitioner | Platform misconfig — X-Original-URL |
| 11 | Method-based access control circumvented | 🟡 Practitioner | Platform misconfig — HTTP method switch |
| 12 | Multi-step process with missing control | 🟡 Practitioner | Multi-step — skip to final step |
| 13 | Referer-based access control | 🟡 Practitioner | Referer header forgery |

---

## After Completing These Labs — What You Can Do

✅ Find unprotected admin panels through `robots.txt`, JS source, and directory brute-forcing  
✅ Exploit parameter-based role control (cookies, JSON fields, URL params)  
✅ Perform IDOR attacks on numeric IDs, GUIDs, and filenames  
✅ Read sensitive data from redirect response bodies  
✅ Chain horizontal + vertical escalation to gain admin access  
✅ Bypass front-end access controls using X-Original-URL  
✅ Bypass method-based controls by switching HTTP verbs  
✅ Skip multi-step workflow steps to execute privileged actions  
✅ Forge the Referer header to bypass referrer-based controls  
✅ Report these issues clearly and accurately in bug bounty reports  

---

*Guide compiled from PortSwigger Web Security Academy — https://portswigger.net/web-security/access-control*  
*Last verified: June 2026*  
*For educational and authorised testing purposes only.*
