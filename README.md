# 🛡️ BugBounty

> My personal bug bounty journey — notes, methodologies, writeups, and tools.  
> Built in public. Updated as I learn and hunt.

---

## 👋 About This Repo

I'm a security researcher transitioning from blockchain security into web application penetration testing and bug bounty hunting. This repository is my public knowledge base — everything I learn, test, and discover gets documented here.

**Current focus:** Web application security — HackerOne & Bugcrowd  
**Background:** 1 year in blockchain/smart contract security  
**Goal:** Build a career in web app pentesting and bug bounty hunting

---

## 📂 Repository Structure

```
BugBounty/
│
├── README.md                    ← You are here
│
├── notes/
│   ├── bug-bounty-bible.md      ← Complete beginner reference (web, HTTP, Burp, vulns)
│   └── ...                      ← More notes added as I learn
│
├── writeups/
│   └── ...                      ← Bug reports and findings (sanitized)
│
├── recon/
│   └── ...                      ← Recon scripts and checklists
│
└── tools/
    └── ...                      ← Custom scripts and tool configs
```

---

## 📒 Notes

| File | Description | Status |
|------|-------------|--------|
| [bug-bounty-bible.md](./notes/bug-bounty-bible.md) | Complete reference — HTTP, Burp Suite, all vuln classes, methodology | ✅ Done |

> More notes added weekly as I work through Hacker101, PortSwigger Web Academy, and live targets.

---

## 🔬 Writeups

> Bug reports and findings will be added here as I discover and report them.  
> All writeups are sanitized — no real user data, no sensitive program details until disclosed.

---

## 🧰 Tools I Use

| Tool | Purpose |
|------|---------|
| [Burp Suite Community](https://portswigger.net/burp) | Proxy, Repeater, Intruder — core testing tool |
| [subfinder](https://github.com/projectdiscovery/subfinder) | Subdomain enumeration |
| [httpx](https://github.com/projectdiscovery/httpx) | Probe live subdomains |
| [ffuf](https://github.com/ffuf/ffuf) | Directory and endpoint fuzzing |
| [nuclei](https://github.com/projectdiscovery/nuclei) | Automated vulnerability scanning |
| [waybackurls](https://github.com/tomnomnom/waybackurls) | Find old URLs from Wayback Machine |
| [gau](https://github.com/lc/gau) | Fetch known URLs from multiple sources |

---

## 📚 Learning Resources

| Resource | What it covers |
|----------|---------------|
| [Hacker101](https://hacker101.com) | Video lessons + CTFs → earns private program invites |
| [PortSwigger Web Academy](https://portswigger.net/web-security) | Hands-on labs for every vuln class |
| [HackerOne](https://hackerone.com) | Primary bug bounty platform |
| [Bugcrowd](https://bugcrowd.com) | Secondary bug bounty platform |

---

## 📈 Progress Tracker

| Topic | Hacker101 | PortSwigger Labs | Hunted on real target |
|-------|-----------|-----------------|----------------------|
| Burp Suite setup | ⬜ | ⬜ | ⬜ |
| XSS | ⬜ | ⬜ | ⬜ |
| SQL Injection | ⬜ | ⬜ | ⬜ |
| IDOR / Access Control | ⬜ | ⬜ | ⬜ |
| Authentication flaws | ⬜ | ⬜ | ⬜ |
| CSRF | ⬜ | ⬜ | ⬜ |
| SSRF | ⬜ | ⬜ | ⬜ |
| File upload bugs | ⬜ | ⬜ | ⬜ |
| Business logic | ⬜ | ⬜ | ⬜ |
| XXE | ⬜ | ⬜ | ⬜ |
| Cryptography / JWT | ⬜ | ⬜ | ⬜ |
| Mobile (iOS) | ⬜ | ⬜ | ⬜ |

---

## 🤝 Connect

- **HackerOne:** *[Zer4ch](https://hackerone.com/zer4ch)*
- **LinkedIn:** *[Zer4ch](https://www.linkedin.com/in/zer4ch/)*
- **Twitter/X:** *[Zer4ch](https://twitter.com/zer4ch)* 

---

> ⚠️ **Disclaimer:** Everything in this repo is for educational purposes and authorized security research only. All testing is performed on programs with explicit permission via HackerOne and Bugcrowd. Never test without authorization.
