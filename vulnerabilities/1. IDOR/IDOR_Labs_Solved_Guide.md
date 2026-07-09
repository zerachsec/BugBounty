# PortSwigger Access Control Labs — Complete Solved Guide
### All 13 Labs, Step-by-Step Solutions
**Zerach | Day 1 Practice | July 9, 2026**

**URL:** portswigger.net/web-security/access-control
**Login for most labs:** `wiener:peter` (your account) — target victim: `carlos`

> ⚠️ Each lab spins up a fresh, unique instance when you click "Access the lab" — your URL will have a random subdomain like `0a1b00c9...web-security-academy.net`. Replace that in every URL below with your own instance's domain.

---

## Lab 1: Unprotected Admin Functionality
**Difficulty:** Apprentice
**Goal:** Access the unprotected admin panel and delete user `carlos`

### Solution
1. Open the lab and go to `/robots.txt`
2. Look for a `Disallow:` line — it'll reveal something like `/administrator-panel`
3. Navigate directly to that path (e.g. `https://[lab-id].web-security-academy.net/administrator-panel`)
4. The admin panel loads with zero login required
5. Click **Delete** next to `carlos`

**Why it works:** The developer "hid" the admin panel by disallowing it in `robots.txt` (meant to stop search engines from indexing it) — but that file is public and readable by anyone, and there's no actual authentication on the panel itself.

---

## Lab 2: Unprotected Admin Functionality with Unpredictable URL
**Difficulty:** Apprentice
**Goal:** Find the hidden admin panel URL and delete `carlos`

### Solution
1. `robots.txt` won't help this time — the URL is random, not disclosed there
2. Load the home page and view page source, or check any linked JS file
3. Look for a script that references the admin panel path (often in a JS file loaded on the page, e.g. `analytics.js` or similar containing a comment or a hardcoded link)
4. Navigate to the disclosed random path (e.g. `/admin-a3f9c2`)
5. Delete `carlos`

**Why it works:** Security through obscurity — the developer assumed nobody would find a random URL, but it's still referenced somewhere in client-side code that any user can read.

---

## Lab 3: User Role Controlled by Request Parameter
**Difficulty:** Apprentice
**Goal:** Access `/admin` (protected by a forgeable cookie) and delete `carlos`

### Solution
1. Log in as `wiener:peter`
2. Open Burp Suite, turn on Intercept, browse to `/admin` — you'll get access denied
3. Look at your cookies — you'll see something like `Admin=false`
4. Send the request to Repeater, change the cookie value to `Admin=true`
5. Resend — you now have admin access
6. Delete `carlos`

**Why it works:** The server trusts a client-controlled cookie to decide admin status instead of checking it server-side against the actual logged-in user's role.

---

## Lab 4: User Role Can Be Modified in User Profile
**Difficulty:** Apprentice
**Goal:** Escalate your own account to admin via the profile update feature

### Solution
1. Log in as `wiener:peter`, go to your account/profile page
2. Intercept the "update profile" request in Burp (usually a `PUT` or `POST` to `/my-account/change-email` or similar)
3. Look at the full request body — there's often a hidden `roleid` or `role` parameter alongside the visible fields
4. Add or modify that parameter to an admin value (e.g. `"roleid":2`) — you may need to inspect the response of a GET request to your profile first to learn the exact field name and admin value
5. Resend — your account is now admin
6. Go to `/admin` and delete `carlos`

**Why it works:** The client sends more fields than the UI displays, and the server blindly trusts and applies all fields in the request instead of ignoring role-related ones from user-editable requests.

---

## Lab 5: User ID Controlled by Request Parameter
**Difficulty:** Apprentice
**Goal:** Exploit horizontal privilege escalation to access `carlos`'s account and steal his API key

### Solution
1. Log in as `wiener:peter`
2. Go to "My account" — note the URL: `/my-account?id=wiener`
3. Simply change the URL to `/my-account?id=carlos`
4. The page now shows `carlos`'s account, including his API key
5. Copy the API key and submit it as the solution

**Why it works:** The server uses the `id` parameter directly to fetch account data with zero check that it matches the logged-in session. This is the purest, most classic IDOR pattern.

---

## Lab 6: User ID Controlled by Request Parameter, with Unpredictable User IDs
**Difficulty:** Apprentice
**Goal:** Find `carlos`'s GUID, then use it to steal his API key

### Solution
1. Log in as `wiener:peter` — note your own account URL uses a GUID: `/my-account?id=83a...`
2. Go to the blog/home page and find a post authored by `carlos` (click "View post")
3. Look at the author link or comment section — `carlos`'s GUID is disclosed there (often in an author profile link or a comment's "author" metadata)
4. Copy that GUID, then navigate to `/my-account?id=[carlos's GUID]`
5. Steal his API key and submit it

**Why it works:** The ID is "unguessable" on its own, but the app leaks it elsewhere (a blog author reference) — proving that GUIDs alone aren't a security control if they're exposed anywhere else in the app.

---

## Lab 7: User ID Controlled by Request Parameter with Data Leakage in Redirect
**Difficulty:** Practitioner
**Goal:** Exploit a redirect that leaks `carlos`'s data before redirecting away

### Solution
1. Log in as `wiener:peter`, send the `/my-account?id=carlos` request to Burp Repeater
2. The app returns a `302 Found` redirect (because you're "not allowed" to view `carlos`'s account) — but check the **response body**, not just the redirect header
3. The full account page HTML — including `carlos`'s API key — is actually included in the body of that redirect response, even though the browser would normally just follow the redirect and never show it to you
4. Read the API key directly from the raw response in Repeater and submit it

**Why it works:** The server generates the full page (with sensitive data) before deciding to redirect, and sends both — a classic case of "the check happens, but too late."

---

## Lab 8: User ID Controlled by Request Parameter with Password Disclosure
**Difficulty:** Practitioner
**Goal:** Retrieve the administrator's password, then log in and delete `carlos`

### Solution
1. Log in as `wiener:peter`, go to `/my-account?id=wiener` and observe the response includes a (masked) password field in the HTML
2. Change the `id` parameter to `administrator`: `/my-account?id=administrator`
3. View the page source / raw response in Burp — the administrator's password is present in the HTML (often in a pre-filled but "hidden" input field), even though it's not rendered visibly in the browser
4. Log out, log back in as `administrator` using the disclosed password
5. Go to `/admin` and delete `carlos`

**Why it works:** The account page template includes the password field for every account it renders, relying on CSS/JS to hide it visually — but the raw HTTP response still contains it in plaintext.

---

## Lab 9: Insecure Direct Object References
**Difficulty:** Apprentice
**Goal:** Exploit chat logs stored with static, predictable URLs to find another user's password, then log in as them

### Solution
1. Log in as `wiener:peter`, find the live chat feature and send a message
2. Note the URL pattern for your chat log download (e.g. `/download-transcript/1.txt`)
3. Try adjacent numbers (`/download-transcript/2.txt`, `/download-transcript/3.txt`, etc.) — one of these belongs to another user's chat transcript
4. Read through the leaked transcript for a password the user typed into the chat (chat support logs often contain users pasting their own credentials by mistake)
5. Log in as that user with the stolen password

**Why it works:** Chat transcripts are stored as flat files on the server's filesystem and served via sequential/predictable static URLs with no ownership check at all.

---

## Lab 10: URL-Based Access Control Can Be Circumvented
**Difficulty:** Practitioner
**Goal:** Bypass access control enforced only at the front-end/proxy layer

### Solution
1. Try accessing `/admin` directly — you'll be blocked (redirected to `/login` or similar) because a front-end proxy blocks direct requests to `/admin` based on the URL path
2. In Burp Repeater, keep the request path as something allowed (e.g. `/`) but add the header: `X-Original-URL: /admin`
3. Send the request — the backend application (unlike the proxy) reads `X-Original-URL` to determine the real route, and grants access since the proxy only inspected the visible path
4. Use the same technique to reach the admin delete-user function: set `X-Original-URL: /admin/delete?username=carlos`

**Why it works:** Access control is enforced by a separate front-end component (reverse proxy/router) that only looks at the literal request path, while the actual backend app trusts a header the proxy forgot to strip or validate.

---

## Lab 11: Method-Based Access Control Can Be Circumvented
**Difficulty:** Practitioner
**Goal:** Bypass access control that only checks the HTTP method

### Solution
1. Log in as `wiener:peter`, try to promote yourself to admin via `POST /admin-roles?username=wiener&action=upgrade` — you'll get blocked (403) because POST to that admin function is restricted to admins only
2. In Repeater, change the request method from `POST` to `GET` (move the parameters into the query string) — or try `POST` with the method overridden via `X-HTTP-Method-Override: GET`
3. The backend still processes the request and grants the action, because it only checks "is this a POST request from an admin" and doesn't equally protect other methods
4. Repeat this to run the admin delete action against `carlos`

**Why it works:** The developer added access control logic only for the specific method they expected the form to use, forgetting that the same route accepts other HTTP methods with identical behavior.

---

## Lab 12: Multi-Step Process with No Access Control on One Step
**Difficulty:** Practitioner
**Goal:** Exploit a checkout-style multi-step admin role upgrade flow

### Solution
1. Log in as `wiener:peter`, navigate to the admin panel's "upgrade user privileges" flow — it's a 3-step wizard: (1) select user, (2) confirm role, (3) apply change
2. As a non-admin, step 1 correctly blocks you
3. In Burp, capture the request for **step 3** (the "confirm"/"apply" action) directly — note its endpoint and parameters
4. Send step 3's request directly (skipping steps 1 and 2 entirely), setting `username=wiener` and the target role to admin
5. The server applies the role change because it only verified admin permissions on step 1, not on step 3
6. Reload — you're now admin. Go delete `carlos`

**Why it works:** Developers often protect the *entry point* of a multi-step flow but assume later steps are "safe" because they expect users to only reach them after the earlier steps — an assumption attackers ignore entirely by calling the final step directly.

---

## Lab 13: Referer-Based Access Control
**Difficulty:** Practitioner
**Goal:** Bypass access control that trusts the `Referer` header

### Solution
1. Log in as `wiener:peter`, try to access the admin panel at `/admin` — blocked
2. Notice that the legitimate admin-panel-link page (e.g. a staff-only nav page) can access `/admin` fine — the app is checking whether the `Referer` header shows you came from that internal page
3. In Burp Repeater, craft your request to `/admin`, and manually set the `Referer` header to the internal staff page's URL (e.g. `Referer: https://[lab-id].web-security-academy.net/admin-role-selector`)
4. Send — access is granted because the check only validates the `Referer` string, which the client fully controls and can forge
5. Use the granted access to delete `carlos`

**Why it works:** `Referer` is a client-supplied, easily spoofed header — never a valid substitute for actual session-based authorization.

---

## Quick Reference — Solve Order & Time Estimate

| # | Lab | Time | Core Trick |
|---|---|---|---|
| 1 | Unprotected admin functionality | 3 min | Check `robots.txt` |
| 2 | ...unpredictable URL | 5 min | Find URL in JS/page source |
| 3 | User role via request param | 5 min | Edit cookie value |
| 4 | Role modifiable in profile | 8 min | Add hidden `roleid` param |
| 5 | User ID via request param | 3 min | Swap `?id=` value |
| 6 | ...unpredictable user IDs | 6 min | Find GUID via blog post |
| 7 | ...data leakage in redirect | 7 min | Read redirect response body |
| 8 | ...password disclosure | 7 min | Read hidden password field in raw HTML |
| 9 | Insecure direct object references | 8 min | Enumerate static transcript URLs |
| 10 | URL-based bypass | 10 min | `X-Original-URL` header |
| 11 | Method-based bypass | 8 min | Swap HTTP method |
| 12 | Multi-step, no check on one step | 10 min | Call final step directly |
| 13 | Referer-based bypass | 8 min | Forge `Referer` header |

**Total: ~90 minutes for all 13.**

---

## After You Finish

Every technique above maps directly to something you'll test on real VDP targets today:
- Labs 1–2 → check for exposed admin panels/hidden routes via `robots.txt` and JS files
- Labs 3–4 → check cookies/hidden params for role/permission fields
- Labs 5–9 → the core IDOR sweep: swap every ID you see, check redirects and raw responses carefully
- Labs 10–13 → header/method bypass tricks for when direct access is blocked

Go run this same checklist against your Dyson/GMX (or current VDP) target now, starting with labs 5, 6, and 9's techniques — they're the highest-yield in real bug bounty programs.
