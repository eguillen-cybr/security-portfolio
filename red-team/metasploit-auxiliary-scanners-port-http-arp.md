# Metasploit Auxiliary Scanners - Port Scanning, HTTP Enumeration & ARP Sweep

## Overview

This lab demonstrates Metasploit's built-in auxiliary scanner modules for active reconnaissance against a Metasploitable 3 Linux target. Four specialized scanners are used in sequence: a TCP port scanner to map the attack surface, an HTTP directory scanner to enumerate web application paths, an HTTP file scanner to identify interesting files within discovered directories, and an ARP sweep to discover all live hosts on the local subnet. Together these modules replicate the reconnaissance and scanning phase of a penetration test entirely within the Metasploit Framework - no external tools required.


---

## Environment

| Component | Details |
|---|---|
| Attacker OS | Kali Linux (kali / kali) |
| Target OS | Metasploitable 3 Linux - 10.0.5.6 |
| Network | VirtualBox NAT Network - MultimachineNAT (10.0.5.0/24) |
| Framework | Metasploit Framework 6 (msfconsole) |
| Modules Used | `auxiliary/scanner/portscan/tcp`, `auxiliary/scanner/http/dir_scanner`, `auxiliary/scanner/http/files_dir`, `auxiliary/scanner/discovery/arp_sweep` |

---

## Walkthrough

### Task 1 - TCP Port Scanner

```
msf6 > use auxiliary/scanner/portscan/tcp
msf6 auxiliary(scanner/portscan/tcp) > set RHOSTS 10.0.5.5
msf6 auxiliary(scanner/portscan/tcp) > run
```

**Open ports discovered on Metasploitable 3 (10.0.5.5):**

```
[+] 10.0.5.5:21    - TCP OPEN   (FTP)
[+] 10.0.5.5:22    - TCP OPEN   (SSH)
[+] 10.0.5.5:80    - TCP OPEN   (HTTP)
[+] 10.0.5.5:135   - TCP OPEN   (RPC)
[+] 10.0.5.5:139   - TCP OPEN   (NetBIOS)
[+] 10.0.5.5:445   - TCP OPEN   (SMB)
[+] 10.0.5.5:1617  - TCP OPEN
[+] 10.0.5.5:3306  - TCP OPEN   (MySQL)
[+] 10.0.5.5:3389  - TCP OPEN   (RDP)
[+] 10.0.5.5:3700  - TCP OPEN
[+] 10.0.5.5:3820  - TCP OPEN
[+] 10.0.5.5:3920  - TCP OPEN
[+] 10.0.5.5:4848  - TCP OPEN   (GlassFish Admin)
[+] 10.0.5.5:5985  - TCP OPEN   (WinRM)
[+] 10.0.5.5:7676  - TCP OPEN
[+] 10.0.5.5:8009  - TCP OPEN   (Tomcat AJP)
[+] 10.0.5.5:8019  - TCP OPEN
[+] 10.0.5.5:8020  - TCP OPEN
[+] 10.0.5.5:8022  - TCP OPEN
[+] 10.0.5.5:8027  - TCP OPEN
[+] 10.0.5.5:8028  - TCP OPEN
[+] 10.0.5.5:8031  - TCP OPEN
[+] 10.0.5.5:8032  - TCP OPEN
[+] 10.0.5.5:8080  - TCP OPEN   (HTTP Alt)
[+] 10.0.5.5:8181  - TCP OPEN
[+] 10.0.5.5:8282  - TCP OPEN
[+] 10.0.5.5:8383  - TCP OPEN
[+] 10.0.5.5:8443  - TCP OPEN   (HTTPS Alt)
[+] 10.0.5.5:8444  - TCP OPEN
[+] 10.0.5.5:8484  - TCP OPEN
[+] 10.0.5.5:8585  - TCP OPEN
[+] 10.0.5.5:8686  - TCP OPEN
[+] 10.0.5.5:9200  - TCP OPEN   (Elasticsearch)
[+] 10.0.5.5:9300  - TCP OPEN   (Elasticsearch Transport)
[*] Scanned 1 of 1 hosts (100% complete)
```

Metasploitable 3 exposes an intentionally large attack surface. High-value targets for further exploitation include MySQL (3306), RDP (3389), GlassFish (4848), WinRM (5985), and Elasticsearch (9200/9300). FTP (21) and SMB (139/445) are also common initial access vectors.

---

### Task 2 - HTTP Directory Scanner

```
msf6 > use auxiliary/scanner/http/dir_scanner
msf6 auxiliary(scanner/http/dir_scanner) > set RHOSTS 10.0.5.5
msf6 auxiliary(scanner/http/dir_scanner) > set RPORT 80
msf6 auxiliary(scanner/http/dir_scanner) > run

[\*] Detecting error code
[*] Using code '404' as not found for 10.0.5.5
[+] Found http://10.0.5.5:80/aspnet_client/ 403 (10.0.5.5)
[*] Scanned 1 of 1 hosts (100% complete)
```

The `dir_scanner` module performs a dictionary-based HTTP directory brute-force, probing for common web application paths. The `/aspnet_client/` directory was discovered with a 403 Forbidden response, confirming it exists but has restricted access. A 403 is still meaningful - it confirms the directory is present and that the server is enforcing access controls, which itself may be misconfigured or bypassable.

The scan was repeated against port 8181 (an alternate HTTP server on the target), which revealed additional directories including `/chat`, `/phpmyadmin`, and `/uploads` - all returning 301 redirects:

```
[+] Found http://10.0.5.6:80/chat        301
[+] Found http://10.0.5.6:80/phpmyadmin  301
[+] Found http://10.0.5.6:80/uploads     301
```

`/phpmyadmin` exposes a database administration interface - a high-priority target for credential attacks. `/uploads` suggests a file upload mechanism that may be exploitable for web shell deployment.

---

### Task 3 - HTTP Interesting File Scanner

With `/chat` identified in the directory scan, the `files_dir` module was used to enumerate specific files within that path using a built-in wordlist of common backup, configuration, and script file extensions:

```
msf6 > use auxiliary/scanner/http/files_dir
msf6 auxiliary(scanner/http/files_dir) > set RHOSTS 10.0.5.6
msf6 auxiliary(scanner/http/files_dir) > set PATH /chat
msf6 auxiliary(scanner/http/files_dir) > run

[*] Using code '404' as not found for files with extension .null
[*] Using code '404' as not found for files with extension .backup
[*] Using code '404' as not found for files with extension .bak
[*] Using code '404' as not found for files with extension .cfg
[*] Using code '404' as not found for files with extension .php
[+] Found http://10.0.5.6:80/chat/index.php  200
[+] Found http://10.0.5.6:80/chat/post.php   200
[*] Scanned 1 of 1 hosts (100% complete)
```

Two PHP files were discovered in the `/chat` directory: `index.php` (the chat front-end) and `post.php` (likely the form handler for chat message submission). PHP form handlers like `post.php` are common targets for injection attacks - if user input is not sanitized, they may be vulnerable to SQL injection, XSS, or remote code execution via file upload parameters.

The module tested extensions including `.backup`, `.bak`, `.cfg`, `.conf`, `.log`, `.old`, `.orig`, `.tar`, `.zip` and others - none of which were present, indicating no exposed backup or configuration files in this directory.

---

### Task 4 - ARP Sweep (Host Discovery)

```
msf6 > use auxiliary/scanner/discovery/arp_sweep
msf6 auxiliary(scanner/discovery/arp_sweep) > set RHOSTS 10.0.5.0/24
msf6 auxiliary(scanner/discovery/arp_sweep) > run

[+] 10.0.5.1  appears to be up (Realtek (UpTech? also reported)).
[+] 10.0.5.2  appears to be up (Realtek (UpTech? also reported)).
[+] 10.0.5.3  appears to be up (CADMUS COMPUTER SYSTEMS).
[+] 10.0.5.5  appears to be up (CADMUS COMPUTER SYSTEMS).
[+] 10.0.5.6  appears to be up (CADMUS COMPUTER SYSTEMS).
[*] Scanned 256 of 256 hosts (100% complete)
```

ARP sweep sent ARP requests to all 256 addresses in the 10.0.5.0/24 subnet and identified five live hosts. MAC address OUI resolution reveals the vendor: "CADMUS COMPUTER SYSTEMS" is the OUI registered to VirtualBox virtual network adapters - confirming these are VM targets. 10.0.5.1 and 10.0.5.2 are the NAT gateway and DHCP server respectively. This technique works at Layer 2 and cannot be blocked by host-based firewalls, making it reliable for local subnet discovery even when ICMP is disabled.

---

## Skills Demonstrated

- TCP auxiliary port scanning within Metasploit Framework
- HTTP directory brute-forcing with `dir_scanner` across multiple ports
- HTTP interesting file enumeration with `files_dir` using extension wordlists
- ARP-based Layer 2 host discovery across a full /24 subnet
- Web application attack surface identification (phpMyAdmin, uploads, PHP handlers)
- MAC OUI analysis for VM and vendor fingerprinting
- Correlation of scan results to exploitable service vectors

---

## References

- [Metasploit Auxiliary Modules Documentation](https://docs.metasploit.com/docs/using-metasploit/basics/using-metasploit.html)
- [Metasploit Unleashed - Scanning](https://www.offensive-security.com/metasploit-unleashed/port-scanning/)
- MITRE ATT&CK: [T1046 - Network Service Discovery](https://attack.mitre.org/techniques/T1046/), [T1595 - Active Scanning](https://attack.mitre.org/techniques/T1595/), [T1018 - Remote System Discovery](https://attack.mitre.org/techniques/T1018/)
