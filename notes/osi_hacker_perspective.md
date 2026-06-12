# OSI Layers — A Hacker's Perspective

> Think of the OSI stack as attack surface by layer. When scoping a target, ask: "What layers can I reach?" The further down the stack you go, the more fundamental the impact.

---

## Layer 7 — Application
**Protocols:** HTTP, DNS, SMTP, FTP

This is where you spend most of your time as a web pentester. User-facing logic, authentication, session handling, APIs — all here. Bugs at this layer have direct business impact.

**Attack techniques:**
- SQL Injection (SQLi)
- Cross-Site Scripting (XSS)
- Insecure Direct Object Reference (IDOR)
- Server-Side Request Forgery (SSRF)
- XML External Entity (XXE)
- Server-Side Template Injection (SSTI)
- Broken authentication
- Cross-Site Request Forgery (CSRF)

**Tools:** `Burp Suite`, `sqlmap`, `ffuf`, `nuclei`, `nikto`

---

## Layer 6 — Presentation
**Protocols:** TLS/SSL, encoding, MIME

Handles encryption, encoding, and data format. Weak TLS configs, expired certs, or insecure cipher suites live here. Also where you intercept and decode traffic.

**Attack techniques:**
- SSL stripping
- Downgrade attack
- Padding oracle
- BEAST/POODLE
- Certificate pinning bypass

**Tools:** `SSLyze`, `testssl.sh`, `Burp Suite (TLS intercept)`, `mitmproxy`

> Layer 6 is frequently forgotten. Most people skip straight from HTTP to TLS as an afterthought, but weak cipher suites, expired certs, and missing HSTS headers have cost real companies real breaches. Run `testssl.sh` on every target, always.

---

## Layer 5 — Session
**Protocols:** Cookies, WebSockets, RPC

Manages persistent connections and session state. Weak session tokens, missing expiry, insecure WebSocket handling, and session fixation attacks target this layer.

**Attack techniques:**
- Session fixation
- Session hijacking
- Cookie theft
- WebSocket injection

**Tools:** `Burp Suite`, `cookie-editor`, `wscat`

---

## Layer 4 — Transport
**Protocols:** TCP, UDP

Handles ports, connections, and reliability. Port scanning lives here. You map open services, fingerprint them, and find misconfigured firewalls. TCP handshake abuse enables SYN floods.

**Attack techniques:**
- SYN flood (DoS)
- Port scanning
- Firewall evasion
- TCP session hijacking

**Tools:** `nmap`, `masscan`, `hping3`, `netcat`

---

## Layer 3 — Network
**Protocols:** IP, ICMP, routing

IP addresses and routing. ICMP recon, IP spoofing, routing manipulation. In internal engagements you pivot across subnets here. BGP hijacking operates at this layer at scale.

**Attack techniques:**
- IP spoofing
- ICMP tunneling
- BGP hijacking
- Route poisoning
- Subnet pivoting

**Tools:** `nmap`, `traceroute`, `scapy`, `iodine (DNS tunnel)`

---

## Layer 2 — Data Link
**Protocols:** Ethernet, ARP, MAC, Wi-Fi

Local network frame delivery. ARP is unauthenticated — you can poison it and become the man-in-the-middle for an entire LAN segment. MAC spoofing bypasses access controls.

**Attack techniques:**
- ARP poisoning / MitM
- MAC spoofing
- VLAN hopping
- Evil twin Wi-Fi

**Tools:** `arpspoof`, `ettercap`, `bettercap`, `aircrack-ng`

> A layer 2 attack like ARP poisoning makes you the MitM for every device on that subnet — you intercept all traffic, not just one user.

---

## Layer 1 — Physical
**Protocols:** Cables, Wi-Fi signals, hardware

Bits on wire or radio. Physical access bypasses nearly all software controls. Rogue devices, USB drops, Wi-Fi sniffing, and hardware implants are physical-layer attacks. Often overlooked in assessments.

**Attack techniques:**
- Rogue access point
- USB drop (BadUSB)
- Hardware implant
- Wi-Fi sniffing
- Keylogger

**Tools:** `Rubber Ducky`, `Wi-Fi Pineapple`, `aircrack-ng`, `Wireshark`

> Layer 1 is the "game over" layer. Physical access = full compromise in most environments. A USB Rubber Ducky dropped in a parking lot or a rogue Pi plugged into an open port bypasses firewalls, EDR, and everything above it.

---

## The Hacker's Workflow

```
Recon (L3–L4)  →  TLS check (L6)  →  App testing (L7)  →  Internal pivot (L2)
   nmap/masscan      testssl.sh         Burp Suite           bettercap/arpspoof
```

- **Layers 3–4** — recon and mapping territory. Scan ports, trace routes, fingerprint services.
- **Layer 6** — TLS/SSL audit. Weak ciphers and missing HSTS are frequently missed.
- **Layer 7** — the application. Where most bug bounty and web pentest work happens.
- **Layer 2** — lateral movement on internal engagements. ARP poisoning, VLAN hopping.
- **Layer 1** — physical assessments. Nuclear option. Bypasses everything above it.
