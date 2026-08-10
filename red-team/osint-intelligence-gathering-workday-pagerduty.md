# Intelligence Gathering – Pre-Engagement OSINT Reconnaissance

> **Disclaimer:** This writeup was completed as an educational lab exercise. All information was gathered passively using publicly available tools and data sources (WHOIS, DNS, Shodan/InternetDB, BuiltWith, Wappalyzer, public LinkedIn/social profiles, etc.). No active scanning, exploitation, or unauthorized access to any target system was performed. Employee names and identifying details have been redacted to protect individual privacy. This document is for educational and portfolio purposes only, is not affiliated with or endorsed by the named organizations, and does not constitute an authorized security assessment of either company.

## Overview

This lab demonstrates passive open-source intelligence (OSINT) reconnaissance techniques used during the pre-engagement phase of a penetration test. Two mid-size enterprise targets — Workday (HR/ERP SaaS) and PagerDuty (incident response platform) — were profiled using a range of publicly available tools covering IP infrastructure, subdomain enumeration, mail server identification, technology fingerprinting, device discovery via Shodan, employee identification, and vulnerability cross-referencing. No active scanning or direct interaction with target systems was performed.


---

## Environment

| Component | Details |
|---|---|
| Attacker Machine | Analyst workstation (browser-based OSINT — no lab VM required) |
| Target 1 | Workday, Inc. — workday.com |
| Target 2 | PagerDuty, Inc. — pagerduty.com |
| Tools Used | CentralOps, SecurityTrails, Shodan/InternetDB, BuiltWith, Wappalyzer, TinEye, theHarvester, portscanner.online, HackerTarget, RecruitEm |

---

## Walkthrough

### Target 1 — Workday

#### Infrastructure Enumeration

| Field | Data | Source |
|---|---|---|
| HQ Address | 6110 Stoneridge Mall Rd, Pleasanton, CA | Waze |
| Website | https://www.workday.com | — |
| IP Range | 13.35.0.0 – 13.35.255.255 | CentralOps (WHOIS) |
| Additional Domains | workday.dk, workday.com.vn, workday.li | BuiltWith redirect lookup |
| Subdomains | design., blog., canvas., forms.workday.com | SecurityTrails |
| Mail Servers | mxa-001ee601.gslb.pphosted.com, mxb-001ee601.gslb.pphosted.com | MX lookup |
| Mail Self-Hosted? | No — mail routed through Proofpoint, Inc. | MX PTR resolution |

**Assessment:** Workday uses Proofpoint for email security, which is a significant defensive control. The IP range resolves to AWS CloudFront (13.35.0.0/16 is an AWS edge prefix), indicating heavy CDN usage — direct server IP identification is obscured.

#### Shodan Device Discovery

Searching by domain name returned two AWS EC2 instances:

```
ec2-52-24-215-127.us-west-2.compute.amazonaws.com
ec2-52-88-61-225.us-west-2.compute.amazonaws.com
```

Both devices displayed HTTP 301 redirects to `https://www.workday.com/en-us/homepage.html`, and their SSL certificates were issued to `workday.com` by Amazon RSA 2048. Open ports: 80 (AWS ELB redirect), 443 (HTTPS). These are confirmed Workday-owned infrastructure.

**Identified Vulnerability:** CVE-2025-0693 — a brute-force enumeration weakness allowing discovery of valid AWS IAM usernames. If Workday's backend IAM is exposed, this could be used to enumerate valid user accounts as a precursor to credential stuffing.

#### Technology Fingerprinting

| Tool | Technologies Identified |
|---|---|
| BuiltWith | Marketo (marketing automation), Adobe Dynamic Tag Management, Adobe Launch |
| Wappalyzer | Coveo (search/AI), Amazon ALB (load balancer) |

**Attack surface implication:** Marketo and Adobe DTM are client-side JavaScript frameworks with known XSS and data-exfiltration histories. Each identified technology should be searched in CVE databases for known public exploits before an engagement.

#### Employee Profiling (Social Engineering Surface)

| Employee (Anonymized) | Evidence | Employer Confirmation |
|---|---|---|
| Employee A | LinkedIn profile photo matched a separate Facebook profile | Self-stated Workday employee |
| Employee B | LinkedIn cover photo location matched Facebook | Software Engineer at Workday |

Cross-referencing LinkedIn with other social networks (Facebook, Twitter/X) allows an attacker to build a fuller picture of a target for spear-phishing or pretexting. Profile photos are a reliable identity anchor across platforms.

> *Note: Employee names, photos, and direct profile links have been redacted from this writeup. Only the methodology and its implications for attack-surface mapping are documented.*

#### Additional OSINT Tools

**portscanner.online** — identified open ports on workday.com's web servers. Open ports reveal running services; each service version can be cross-referenced against known CVEs.

**TinEye** (reverse image search) — used to locate employees' social media profiles by searching with their LinkedIn profile photos. Useful for finding accounts on platforms not indexed by name.

**theHarvester** — gathered additional employee email addresses and hostnames associated with workday.com by querying public search engines and data sources. Harvested emails feed directly into phishing campaign setup.

---

### Target 2 — PagerDuty

#### Infrastructure Enumeration

| Field | Data | Source |
|---|---|---|
| HQ Address | 600 Townsend St., Suite 200, San Francisco, CA 94103 | SEC EDGAR filing |
| Website | https://www.pagerduty.com | — |
| IP Range | 104.16.0.0 – 104.31.255.255 | CentralOps (WHOIS) |
| Additional Domains | pagerduty.org, pagerduty.community, pagerduty.cloud | BuiltWith |
| Subdomains | support., blog., app., investor.pagerduty.com | SecurityTrails |
| Mail Servers | aspmx.l.google.com, aspmx2.googlemail.com | MX lookup |
| Mail Self-Hosted? | No — Google Workspace | MX record resolution |

**Assessment:** PagerDuty's IP range (104.16.0.0/12) belongs to Cloudflare — confirming CDN and DDoS-mitigation infrastructure. Direct origin IP is masked behind Cloudflare's edge network. Attempts to scan or access origin IPs directly may be blocked or logged.

#### Shodan Device Discovery

Domain search returned `blog.pagerduty.com` (104.16.118.10) and `pagerduty.com` (104.16.117.10), both served through Cloudflare with certificates issued by Google Trust Services to `blog.pagerduty.com` and `www.pagerduty.com`. Supported TLS: 1.0, 1.1, 1.2, 1.3.

IP-based search on 104.16.117.10 showed no additional devices but revealed several open ports (2053, 2082, 2083 TCP) — Cloudflare's alternative HTTPS port range, blocking direct HTTP access while serving HTTPS. A 403 Forbidden response on port 2082 indicates direct IP access is disabled at the Cloudflare layer.

**Identified Vulnerability:** CVE-2024-8603 — potential use of a weak or deprecated cryptographic algorithm. TLS 1.0 and 1.1 support detected in Shodan is notable; both are deprecated by RFC 8996 and can expose sessions to BEAST and POODLE-class downgrade attacks.

#### Technology Fingerprinting

| Tool | Technologies Identified |
|---|---|
| BuiltWith | Marketo, Visual Website Optimizer (A/B testing), Rapleaf |
| Wappalyzer | MySQL (backend), CMS |

#### Employee Profiling (Social Engineering Surface)

| Employee (Anonymized) | Role | Evidence | Employer Confirmation |
|---|---|---|---|
| Employee C | Executive leadership | LinkedIn profile | Active posts about company projects |
| Employee D | Security role (held CISA/CDPSE-level certifications) | LinkedIn profile | Role and employer listed |

Identifying security staff (e.g., a CISA-certified employee) is particularly valuable — they are targets for authority-based pretexting ("I'm calling from your external auditor…") and can be used to understand the organization's security posture and tooling by reviewing their public certifications and posts.

> *Note: Employee names and profile links have been redacted from this writeup. Only the methodology and its implications for attack-surface mapping are documented.*

#### Additional OSINT Tools

**HackerTarget Nmap Online Scanner** — identified open TCP ports on pagerduty.com's web server. Service banners can reveal software versions for CVE lookups.

**HackerTarget SSL Check** — returned full TLS configuration including certificate chain, cipher suites, protocol versions, and configuration errors. Identified a potentially misconfigured certificate that could allow a machine-in-the-middle attack under certain downgrade conditions.

**RecruitEm** (LinkedIn X-Ray search) — used Google dork syntax to enumerate PagerDuty employee LinkedIn profiles by department and title without requiring a LinkedIn account. Provides direct access to role names, tenure, and technology mentions useful for crafting targeted spear-phishing lures.

---

## Skills Demonstrated

- Passive IP range and WHOIS enumeration (CentralOps)
- Subdomain discovery (SecurityTrails)
- Mail server identification and third-party mail provider detection
- Device and port discovery via Shodan / InternetDB
- SSL certificate ownership verification
- Technology fingerprinting (BuiltWith, Wappalyzer)
- CVE cross-referencing from device/technology discoveries
- Employee OSINT for social engineering surface mapping
- Reverse image search for cross-platform identity correlation (TinEye)
- Email and hostname harvesting (theHarvester)
- Remote port scanning for service enumeration
- TLS/SSL configuration auditing

---

## References

- [Shodan.io](https://www.shodan.io/)
- [SecurityTrails](https://securitytrails.com/)
- [BuiltWith Technology Profiler](https://builtwith.com/)
- [Wappalyzer](https://www.wappalyzer.com/)
- [CentralOps.net WHOIS](https://centralops.net/co/)
- [theHarvester – GitHub](https://github.com/laramies/theHarvester)
- [HackerTarget Online Tools](https://hackertarget.com/)
- [CVE-2025-0693 – NVD](https://nvd.nist.gov/vuln/detail/CVE-2025-0693)
- [CVE-2024-8603 – NVD](https://nvd.nist.gov/vuln/detail/CVE-2024-8603)
- [OSINT Framework](https://osintframework.com/)
- MITRE ATT&CK: [T1589 – Gather Victim Identity Information](https://attack.mitre.org/techniques/T1589/), [T1596 – Search Open Technical Databases](https://attack.mitre.org/techniques/T1596/), [T1593 – Search Open Websites/Domains](https://attack.mitre.org/techniques/T1593/)
