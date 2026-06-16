# Zer4ch — Web Hacking Roadmap

## One-Time Setup (Day 1)

- Install **Burp Suite Community Edition**
- Create accounts: **Intigriti** + **Bugcrowd** + **YesWeHack**
- Bookmark these 3 resource hubs:
  - Intigriti Hackademy → `intigriti.com/researchers/hackademy`
  - PortSwigger Academy → `portswigger.net/web-security`
  - Intigriti YouTube Playlist → `youtube.com/playlist?list=PLmqenIp2RQciV955S2rqGAn2UOrR2NX-v`
- Pick **1 target per platform** (public program, web scope, wildcard domains)

---

## The Loop — Repeat for Every Vuln

```
Intigriti Hackademy article on the vuln
          ↓
Intigriti YouTube Hackademy video on same vuln
          ↓
PortSwigger theory page
          ↓
PortSwigger Apprentice + Practitioner labs
          ↓
Read 2–3 disclosed reports → hackerone.com/hacktivity (filter by vuln)
          ↓
Hunt that specific vuln on your target (Intigriti / Bugcrowd / YesWeHack)
          ↓
Move to next vuln only after hunting
```

---

## Vuln Order + Full Resources

---

### SQL Injection

| Resource | Link |
|---|---|
| Hackademy article | `intigriti.com/researchers/hackademy/sql-injection` |
| PortSwigger theory | `portswigger.net/web-security/sql-injection` |
| PortSwigger labs | Apprentice + Practitioner (skip Expert for now) |
| YouTube | Search "Intigriti SQL injection" on their channel |
| Disclosed reports | HackerOne Hacktivity → filter SQLi |

**Hunt:** Look for login forms, search bars, any input that hits a DB. Use `sqlmap` to assist.

---

### Authentication Flaws

| Resource | Link |
|---|---|
| PortSwigger theory | `portswigger.net/web-security/authentication` |
| PortSwigger labs | Apprentice + Practitioner |
| YouTube | Search "authentication bypass bug bounty" on Intigriti channel |

**Hunt:** Password reset flows, login rate limiting, MFA bypass, account enumeration via error messages.

---

### IDOR / Access Control

| Resource | Link |
|---|---|
| Hackademy article | `intigriti.com/researchers/hackademy/idor` |
| PortSwigger theory | `portswigger.net/web-security/access-control` |
| PortSwigger labs | Apprentice + Practitioner |

**Hunt:** Create 2 accounts. Swap IDs in every API request. Test export/download endpoints. Check GUIDs — sometimes predictable underneath.

---

### XSS (Reflected → Stored → DOM)

| Resource | Link |
|---|---|
| Hackademy XSS | `intigriti.com/researchers/hackademy/cross-site-scripting-xss` |
| Hackademy Stored XSS | `.../stored-cross-site-scripting` |
| Hackademy Reflected XSS | `.../reflected-cross-site-scripting` |
| Hackademy DOM XSS | `.../dom-based-cross-site-scripting` |
| How to test XSS | `.../how-to-test-for-cross-site-scripting` |
| PortSwigger labs | All XSS labs — Apprentice + Practitioner |
| Intigriti XSS Challenges | `intigriti.com/researchers/hackademy/guides/xss-challenges` |

**Hunt:** Every input field, URL param, search box, comment section, profile fields.

---

### SSRF

| Resource | Link |
|---|---|
| Hackademy article | `intigriti.com/researchers/hackademy/server-side-request-forgery-ssrf` |
| PortSwigger theory | `portswigger.net/web-security/ssrf` |
| PortSwigger labs | Apprentice + Practitioner |
| Free Burp Collaborator alt | `interactsh` (open source) |
| Intigriti SSRF payload gen | `tools.intigriti.io/redirector/` |

**Hunt:** Webhooks, URL fetcher features, PDF generators, image importers — anywhere the server makes an outbound request.

---

### File Upload Vulnerabilities

| Resource | Link |
|---|---|
| Hackademy article | `intigriti.com/researchers/hackademy/file-upload-vulnerabilities` |
| PortSwigger theory | `portswigger.net/web-security/file-upload` |
| PortSwigger labs | All labs |

**Hunt:** Profile pictures, document uploads, any file input. Try SVG with XSS payload, PHP shell with renamed extension.

---

### Business Logic

| Resource | Link |
|---|---|
| PortSwigger theory | `portswigger.net/web-security/logic-flaws` |
| PortSwigger labs | All labs |

**Hunt:** Price manipulation, discount abuse, workflow skipping (checkout without payment), quantity as negative number. No tool finds these — pure thinking.

---

### XXE

| Resource | Link |
|---|---|
| Hackademy article | `intigriti.com/researchers/hackademy/xml-external-entity-processing-xxe` |
| PortSwigger theory | `portswigger.net/web-security/xxe` |
| PortSwigger labs | Apprentice + Practitioner |

**Hunt:** Any endpoint accepting XML input — SOAP APIs, document parsers, data imports.

---

### CSRF

| Resource | Link |
|---|---|
| Hackademy article | `intigriti.com/researchers/hackademy/cross-site-request-forgery-csrf` |
| PortSwigger theory | `portswigger.net/web-security/csrf` |
| PortSwigger labs | Apprentice + Practitioner |

**Hunt:** State-changing actions without CSRF tokens — password change, email change, account settings.

---

### Directory Traversal + Open Redirect + HTTP Parameter Pollution

| Resource | Link |
|---|---|
| Hackademy Dir Traversal | `intigriti.com/researchers/hackademy/directory-traversal` |
| Hackademy Open Redirect | `intigriti.com/researchers/hackademy/open-redirect` |
| Hackademy HPP | `intigriti.com/researchers/hackademy/http-parameter-pollution` |
| PortSwigger Path Traversal | `portswigger.net/web-security/file-path-traversal` |

---

## Picking One Vuln and Hunting — The Rule

Pick one vuln, learn it fully, then test **only** that vuln on your target for 3 focused days. No find → move on. New vuln class every new topic. Don't get stuck.

**3 days on one vuln per target. No find → next vuln. New topic → new vuln class to learn.**

---

## Recon Stack (Set Up Day 1, Use from IDOR Week)

```bash
# Install tools
go install github.com/projectdiscovery/subfinder/v2/cmd/subfinder@latest
go install github.com/projectdiscovery/httpx/cmd/httpx@latest
go install github.com/lc/gau/v2/cmd/gau@latest
go install github.com/projectdiscovery/nuclei/v2/cmd/nuclei@latest

# Basic recon flow
subfinder -d target.com -o subs.txt
cat subs.txt | httpx -o live.txt
cat live.txt | gau | tee endpoints.txt
nuclei -l live.txt -t ~/nuclei-templates/
```

---

## Hacking Tools Reference

Full list maintained by Intigriti: `intigriti.com/researchers/hackademy/guides/hacking-tools`

---

## How to Write Reports

Read before your first submission: `intigriti.com/researchers/hackademy/guides/how-to-write-a-good-report`

A bad report on a valid bug gets rejected or downgraded. This matters.

---

## Weekly Schedule

| Day | Task |
|---|---|
| Monday | Hackademy article + PortSwigger theory |
| Tuesday | PortSwigger labs (Apprentice + Practitioner) |
| Wednesday | Read 2–3 disclosed HackerOne reports on that vuln |
| Thursday | Hunt on Intigriti |
| Friday | Hunt on Bugcrowd or YesWeHack |
| Saturday | Write notes, document what you found/didn't find |
| Sunday | Rest or recon on next week's target |

---

## Milestones

| Milestone | Target |
|---|---|
| Setup done, first target chosen | Day 1 |
| 3 vulns learned + hunted | Week 3 |
| First report submitted | Week 4 |
| First valid finding | Week 6–8 |
| First bounty paid | Week 8–12 |
