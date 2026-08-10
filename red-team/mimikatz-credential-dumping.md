# Credential Dumping & Advanced Reconnaissance — Mimikatz & Legion

**Environment:** Kali Linux → Metasploitable 3 Windows Server 2008 R2  
**Tools Used:** Mimikatz, Metasploit (Meterpreter), Legion, Hydra, Nikto  

---

## Overview

This lab covers two critical post-exploitation and reconnaissance skill sets:

1. **Credential Dumping with Mimikatz** — extracting plaintext passwords and hashes directly from Windows memory, both locally and remotely via Meterpreter
2. **Advanced Network Reconnaissance with Legion** — automated multi-tool scanning, service enumeration, and credential brute-forcing against a live target

These techniques are foundational knowledge for both red team operators and blue team defenders. A SOC analyst who understands *how* credentials are stolen is far better equipped to detect and respond to it.

---

## Part 1 — Mimikatz: Windows Credential Extraction

### What is Mimikatz?

Mimikatz is a post-exploitation tool that extracts passwords, NTLM/LM hashes, Kerberos tickets, and PINs directly from **LSASS (Local Security Authority Subsystem Service)** memory — the Windows process responsible for enforcing security policy and storing authentication data.

Specifically, Mimikatz targets:

| Source | Data Extracted |
|--------|---------------|
| `msv` | NTLM and LM password hashes |
| `wdigest` | Cleartext passwords (cached in older Windows versions) |
| `kerberos` | Kerberos tickets and plaintext credentials |
| `tspkg` | Terminal Services credentials |
| `credman` | Windows Credential Manager entries |

> **Why wdigest matters:** Windows versions prior to Server 2012 R2 (and unpatched later versions) store credentials in cleartext in memory via the WDigest authentication protocol. This is why Mimikatz is so dangerous on legacy systems and why patching and disabling WDigest is a critical hardening step.

---

### 1A — Local Execution on Metasploitable 3 Win2K8

Mimikatz was downloaded and executed directly on the Windows Server 2008 R2 target VM. The following command sequence was used:

```
mimikatz # privilege::debug
mimikatz # sekurlsa::logonpasswords
```

`privilege::debug` is required to gain SeDebugPrivilege, which allows Mimikatz to access LSASS process memory.

**Credentials Extracted:**

```
Authentication Id : 0 ; 229157 (00000000:00037f25)
Session          : Interactive from 1
User Name        : vagrant
Domain           : METASPLOITABLE3
Logon Server     : METASPLOITABLE3
Logon Time       : 4/14/2025 8:19:13 PM
SID              : S-1-5-21-358291975-4152007756-2235311849-1000

  msv:
    [00000003] Primary
    * Username : vagrant
    * Domain   : METASPLOITABLE3
    * LM       : 5229b7f52540641daad3b435b51404ee
    * NTLM     : e02bc503339d51f71d913c245d35b50b
    * SHA1     : c805f88436bcd9ff534ee86c59ed2304375050ecf
  wdigest:
    * Username : vagrant
    * Domain   : METASPLOITABLE3
    * Password : vagrant          ← CLEARTEXT
  kerberos:
    * Username : vagrant
    * Password : vagrant          ← CLEARTEXT
```

**Source:** Credentials were captured from **WDigest authentication** cached in LSASS memory. Because WDigest is enabled by default on Windows Server 2008 R2, the plaintext password is available without cracking any hashes.

---

### 1B — Remote Execution via Meterpreter

After exploiting the Win2K8 machine remotely from Kali (using MS09-050 SMBv2 exploit as documented in the penetration testing lab), Mimikatz was executed directly through the Meterpreter session using the `kiwi` extension:

```
meterpreter > load kiwi
meterpreter > creds_all
```

**Remote Credential Dump Results:**

| Username | Domain | Password | Protocol |
|----------|--------|----------|----------|
| sshd_server | METASPLOITABLE3 | D@rj33l1ng | wdigest |
| vagrant | METASPLOITABLE3 | vagrant | wdigest |
| sshd_server | METASPLOITABLE3 | D@rj33l1ng | tspkg |
| vagrant | METASPLOITABLE3 | vagrant | tspkg |
| sshd_server | METASPLOITABLE3 | D@rj33l1ng | kerberos |
| vagrant | METASPLOITABLE3 | vagrant | kerberos |

**Key finding:** The `sshd_server` service account credential `D@rj33l1ng` was recovered — a non-obvious password that would not appear in a standard wordlist, yet was exposed in plaintext via WDigest. This demonstrates why **memory-resident credential exposure** is more dangerous than hash-based attacks — no cracking required.

---

### Defensive Implications (Blue Team Perspective)

Understanding Mimikatz from the attacker's side directly informs defensive detection:

| Attack Action | Detection Opportunity |
|--------------|----------------------|
| `privilege::debug` (SeDebugPrivilege) | Alert on non-admin processes requesting SeDebugPrivilege |
| LSASS memory access | Windows Defender Credential Guard blocks this on modern systems; enable it |
| `sekurlsa::logonpasswords` | LSASS process access events (Event ID 4656, 10 in Sysmon) |
| Kiwi/Mimikatz loaded in Meterpreter | EDR signature detection; behavioral: unusual LSASS read patterns |
| WDigest cleartext exposure | **Mitigation:** Set `HKLM\SYSTEM\CurrentControlSet\Control\SecurityProviders\WDigest\UseLogonCredential = 0` |

---

## Part 2 — Legion: Advanced Reconnaissance & Post-Exploitation

### What is Legion?

Legion is a GUI-based network reconnaissance framework built into Kali Linux that orchestrates multiple security tools (Nmap, Nikto, Hydra, and others) against a target. It provides a centralized interface for running parallel scans and aggregating results.

**Target:** Metasploitable 3 Windows Server 2008 R2 (`10.0.5.4`)

---

### Full Service Enumeration Scan

Legion's automated scan revealed an exceptionally large attack surface on the target — over 40 open ports across multiple high-risk services:

**High-Priority Findings:**

| Port | Service | Version | Risk |
|------|---------|---------|------|
| 21 | FTP | Microsoft ftpd | Credential brute-force target |
| 22 | SSH | OpenSSH 7.1 | Credential brute-force target |
| 445 | SMB | Windows Server 2008 R2 | Multiple known CVEs |
| 3306 | MySQL | 5.5.20 | Unauthorized access potential |
| 4848 | HTTPS | Oracle GlassFish 4.0 | Multiple known vulnerabilities |
| 8032 | HTTP | ManageEngine Desktop Central | Known RCE vulnerabilities |
| 8080 | HTTP | Oracle GlassFish 4.0 | Publicly exploitable |
| 8585 | HTTP | Apache 2.2.21 / PHP 5.3.10 | Outdated, multiple CVEs |
| 49276 | TCP | Jenkins TcpSlaveAgentListener | Jenkins exploitation surface |

**Analyst Note:** The sheer number of exposed services on this machine represents a critical misconfiguration in a real environment. Each running service is a potential entry point. Defense-in-depth would require disabling all non-essential services, applying patches, and placing the machine behind a properly configured firewall.

---

### Hydra — FTP Credential Brute Force

Legion's integrated Hydra module was used to brute-force the FTP service on port 21:

```
[21][ftp] host: 10.0.5.4   login: vagrant   password: vagrant
1 of 1 target successfully completed, 1 valid password found
```

**Finding:** Default credentials (`vagrant:vagrant`) were valid on the FTP service — a critical misconfiguration that would represent an immediate critical finding in a real penetration test.

---

### Tool 1 — SSH Default Credential Check (Hydra via Legion)

Legion ran Hydra against port 22 (SSH) using a default credential wordlist:

```
[22][ssh] host: 10.0.5.4   login: vagrant   password: vagrant
1 of 1 target successfully completed, 1 valid password found
```

**Significance:** The same default credential pair (`vagrant:vagrant`) was valid for both FTP and SSH — meaning an attacker who compromised one service could immediately pivot to the other. This is a credential reuse vulnerability, one of the most common and preventable findings in real engagements.

---

### Tool 2 — Nikto Web Vulnerability Scan (Port 80/IIS)

Nikto was run against Microsoft IIS 7.5 on port 80, revealing several security misconfigurations:

| Finding | Risk |
|---------|------|
| `X-Frame-Options` header missing | Clickjacking vulnerability |
| `X-Content-Type-Options` header missing | MIME-type sniffing attacks possible |
| ASP.NET version exposed via `x-powered-by` header | Version disclosure aids targeted attacks |
| `TRACE` HTTP method enabled | Cross-Site Tracing (XST) attacks possible |
| ASP.NET version: `2.0.50727` | Outdated, end-of-life framework |

**Analyst Note:** None of these findings alone represent a critical vulnerability, but collectively they indicate **security headers are not configured** — a systemic gap that suggests the web server was never hardened. In a real engagement, these would be listed as informational/medium findings with recommended remediations for each header.

---

## Key Takeaways

| Concept | Lesson |
|---------|--------|
| **WDigest is dangerous on legacy systems** | Cleartext passwords in memory require no cracking — disable WDigest immediately on any Windows system |
| **Default credentials are critical findings** | `vagrant:vagrant` worked on FTP, SSH simultaneously — credential reuse amplifies impact |
| **Service exposure = attack surface** | 40+ open ports on one machine is indefensible; every service must be justified |
| **Memory-based attacks bypass hashing** | Mimikatz proves that strong password hashing doesn't protect against LSASS dumping |
| **Legion aggregates efficiently** | Running Nmap + Hydra + Nikto manually takes hours; Legion parallelizes and centralizes results |
| **Defenders need attacker knowledge** | Knowing *how* Mimikatz works makes Event ID 4656 alerts meaningful instead of noise |

---

## Tools Reference

| Tool | Purpose |
|------|---------|
| Mimikatz | Windows credential extraction from LSASS memory |
| Metasploit `kiwi` | Mimikatz integration within Meterpreter sessions |
| Legion | GUI reconnaissance framework orchestrating multiple tools |
| Hydra | Credential brute-forcing (FTP, SSH, HTTP, etc.) |
| Nikto | Web server misconfiguration and vulnerability scanning |
| Nmap | Port and service enumeration (run automatically by Legion) |
