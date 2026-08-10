# Web Application Hacking – Burp Suite & OWASP Juice Shop

**Tools:** Burp Suite Community Edition, OWASP Juice Shop (Docker), Kali Linux, Chromium  
**Skills Demonstrated:** HTTP proxy interception, SQL injection, broken access control exploitation, cryptographic failure discovery, security misconfiguration

---

## Overview

Performed web application penetration testing against OWASP Juice Shop — a deliberately vulnerable Node.js e-commerce application — using Burp Suite as an intercepting HTTP proxy. Completed four challenge categories from the OWASP Top 10, finishing with 8/171 challenges solved (7% hacking challenges).

---

## Environment

| Component | Details |
|-----------|---------|
| Attacker Machine | Kali Linux (VirtualBox) |
| Target Application | OWASP Juice Shop (Docker container) |
| Proxy Tool | Burp Suite Community Edition v2024.10.3 |
| Browser | Chromium (via Burp's built-in browser, correct proxy settings pre-configured) |
| Network | VirtualBox NAT Network — `MultimachineNAT` (10.0.5.0/24) |

### Setup Notes

- Docker installed on Kali; Juice Shop pulled and run as a container
- Used Burp Suite's **Target → Site Map → Open Browser** to launch Chromium with proxy already configured (bypassed broken FoxyProxy integration)
- Burp Suite JRE warning acknowledged; out-of-date warning dismissed

---

## Challenge 1 – Injection Attack (SQL Injection Login Bypass)

**OWASP Category:** Injection  
**Difficulty:** ★★

### Approach

1. Navigated to the Juice Shop login page in Chromium with Burp intercept **ON**
2. Entered test credentials and captured the POST request in Burp's **Proxy → Intercept** tab
3. Identified the `email` parameter as the injection point
4. Modified the email field to a classic SQL injection bypass payload: `' OR 1=1--`
5. Forwarded the modified request through Burp
6. Application authenticated as the **admin** user

### Result

Challenge completed — Juice Shop displayed the fireworks/green bar confirmation: *"Login Admin – Log in with the administrator's user account."*

**Key takeaway:** Unsanitized SQL query parameters allow attackers to bypass authentication entirely by injecting logic that always evaluates to true.

---

## Challenge 2 – Broken Access Control (Admin Panel Access)

**OWASP Category:** Broken Access Control  
**Difficulty:** ★★

### Approach

1. While authenticated as admin from Challenge 1, examined the Juice Shop's client-side JavaScript source files via Burp's Site Map
2. Discovered a reference to the hidden `/administration` route in the bundled JS
3. Navigated directly to `http://localhost:3000/#/administration`
4. Accessed the full **Administration panel** showing all registered users and customer feedback

### Result

Challenge completed — gained unauthorized access to the admin panel revealing all user accounts (admin@juice-sh.op, jim@juice-sh.op, bender@juice-sh.op, and others) and all customer feedback entries.

**Key takeaway:** Security through obscurity is not access control. Hidden URLs exposed in client-side code are accessible to any user who inspects the source.

---

## Challenge 3 – Cryptographic Failure (Nested Easter Egg)

**OWASP Category:** Cryptographic Failures  
**Difficulty:** ★★★

### Approach

1. Located the Score Board page by finding its route in client-side JS (`/#/score-board`)
2. Followed the cryptographic challenge chain — discovered an encoded easter egg path in the application's FTP directory
3. Applied Base64 decoding and ROT13 to decode the obfuscated path
4. Navigated to the decoded URL to retrieve the nested easter egg

### Result

Challenge completed — Juice Shop confirmed: *"Nested Easter Egg – Apply some advanced cryptanalysis to find the real easter egg."*

**Key takeaway:** Weak or absent encryption of sensitive paths/data allows attackers to recover hidden endpoints through simple decoding techniques.

---

## Challenge 4 – Security Misconfiguration (Deprecated B2B Interface)

**OWASP Category:** Security Misconfiguration  
**Difficulty:** ★★

### Approach

1. Identified a legacy B2B complaint submission endpoint (`/api/complaints`) still active in the application
2. The endpoint accepted XML file uploads — a deprecated interface that was never properly disabled
3. Submitted a request to the endpoint using the Complaint form with a crafted XML invoice attachment
4. The server processed the request, confirming the deprecated interface remained functional

### Result

Challenge completed — Juice Shop confirmed: *"Deprecated Interface – Use a deprecated B2B interface that was not properly shut down."*

**Key takeaway:** Forgotten or deprecated API endpoints that aren't disabled during application updates represent a common and serious misconfiguration risk.

---

## Final Scoreboard

| Metric | Value |
|--------|-------|
| Challenges Solved | 8 / 171 |
| Hacking Progress | 7% |
| ★1 Challenges | 3/28 |
| ★2 Challenges | 4/23 |
| ★3 Challenges | 0/44 |
| ★4 Challenges | 1/37 |
| ★5 Challenges | 0/25 |
| ★6 Challenges | 0/14 |

---

## Skills Demonstrated

- HTTP proxy interception and request manipulation (Burp Suite)
- SQL injection for authentication bypass
- Client-side source code analysis to discover hidden routes
- Broken access control exploitation (forced browsing)
- Cryptographic decoding (Base64, ROT13) for path discovery
- Security misconfiguration identification (deprecated endpoints)
- OWASP Top 10 vulnerability identification and exploitation
- Docker-based lab environment setup on Kali Linux

---

## References

- [OWASP Juice Shop](https://owasp.org/www-project-juice-shop/)
- [Burp Suite Community Edition](https://portswigger.net/burp/communitydownload)
- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [The Ultimate Kali Linux Book (O'Reilly)](https://learning.oreilly.com/library/view/the-ultimate-kali/9781801818933/)
