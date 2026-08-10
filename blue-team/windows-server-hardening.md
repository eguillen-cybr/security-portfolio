# Windows Server Hardening
### Active Directory Domain Controller — Security Audit & Remediation


---

## Overview

A Windows Server 2019 domain controller for the `is3650.com` Active Directory environment was identified as critically misconfigured following the departure of a previous IT administrator under suspicious circumstances. As the contracted security consultant, the objective was to perform a full security audit and harden the server while preserving all legitimate business services: Active Directory/DNS, DHCP, IIS web server, and a read-only SMB file share (CORPDOCS).

The audit revealed five distinct vulnerability categories — firewall misconfiguration, over-permissioned file shares, unnecessary and potentially malicious software, excessive user accounts with weak credentials, and a complete absence of endpoint protection. Every finding was remediated using a least-privilege, defense-in-depth approach consistent with CIS Benchmark guidance for Windows Server 2019 and NIST SP 800-123 (Guide to General Server Security).

Notably, the server was found running **HoneyBOT 0.1.8** — honeypot software with no legitimate production purpose — which may indicate that the departing administrator was conducting unauthorized network monitoring or intelligence gathering against internal users. This finding was flagged as requiring further investigation.

---

## Environment

| Component | Detail | Role |
|-----------|--------|------|
| Target OS | Windows Server 2019 Standard | Domain Controller (AD DS, DNS, DHCP, IIS) |
| Domain | is3650.com | Active Directory forest root |
| IP Address | 10.0.5.4/24 | Static — DHCP server for the segment |
| Gateway | 10.0.5.1 | NAT network egress |
| Hypervisor | Oracle VirtualBox 6.1.16 | MultimachineNAT (10.0.5.0/24) |
| Scan/Verify VM | Bodhi Linux | Used to verify web server and network reach |
| Hardware | Intel Core i7-9700F @ 3.0 GHz | 4 GB RAM, 101 GB disk |
| Tools Used | Windows Defender, WF.msc, ADUC, Server Manager | Native OS hardening toolset |

---

## Findings Summary

| Finding | Severity | Remediation |
|---------|----------|-------------|
| Firewall completely disabled | 🔴 Critical | Enabled Windows Defender Firewall; scoped inbound/outbound rules |
| No antivirus / endpoint protection | 🔴 Critical | Enabled Windows Defender with real-time and cloud protection |
| HoneyBOT 0.1.8 installed (unauthorized) | 🟠 High | Removed; flagged as possible insider threat indicator |
| Stale AD accounts (Karina Feder, Oleg Vasilov) | 🟠 High | Removed all unauthorized accounts; retained only Karl and Frida |
| Weak/default user passwords | 🟠 High | Reset with temp passwords; forced change on next login; 1-year expiry |
| OS not patched (multiple missing updates) | 🟠 High | Applied all available Windows Updates to current patch level |
| FileZilla (FTP server) installed + FTP rules enabled | 🟠 High | Uninstalled FileZilla; disabled all FTP firewall rules |
| CORPDOCS share — Users group had write/modify rights | 🟡 Medium | Removed inheritance; set IS3650\\Users to Read-only |
| Unnecessary software (Winamp, Winamp Plug-in) | 🟡 Medium | Uninstalled; no legitimate server function |
| Unnecessary inbound rules (Cast, Teredo, DIAL, LLMNR) | 🟡 Medium | Disabled; not deleted (preserves rollback option) |
| LibreOffice 3.4 — outdated, EOL office suite on DC | 🟢 Low | Removed; no legitimate domain controller function |
| Google Chrome — outdated/broken installation | 🟢 Low | Uninstalled old version; reinstalled current release |

---

## Walkthrough

### Part 1 — Firewall Audit and Hardening

The first and most critical finding was that **Windows Defender Firewall was completely disabled**. A production domain controller with no host-based firewall is exposed to the full threat surface of the network — any service listening on any port is reachable by any machine on the segment.

**Action — Enable Firewall:**

```
Control Panel > System and Security > Windows Defender Firewall
> Turn Windows Firewall on or off
> Enable for all profiles: Domain, Private, Public
```

---

#### Inbound Rules — Kept (required for legitimate services)

| Rule Group | Reason Kept |
|------------|-------------|
| Active Directory Domain Controller — all rules | Required for AD DS replication, LDAP, Kerberos, RPC |
| Active Directory Web Services (TCP-In) | Required for ADWS / PowerShell AD module access |
| Core Networking — DHCP, IPHTTPS, IGMP, ICMPv6 subset | Necessary for network functionality and DC health monitoring |
| DHCP Server — all rules | Server is the DHCP provider for the 10.0.5.0/24 segment |
| DNS (TCP/UDP Incoming) | Required for AD-integrated DNS |
| File and Printer Sharing (SMB-In) | Required for CORPDOCS share access |
| File Server Remote Management — all rules | Required for Server Manager remote management |
| Kerberos Key Distribution Center — all rules | KDC is a core AD authentication component |
| World Wide Web Services (HTTP/HTTPS Traffic-In) | Required for the IIS web server role |
| Google Chrome (mDNS-In) | Retained as the modern browser replacement for IE |

---

#### Inbound Rules — Disabled (attack surface reduction)

All rules below were **disabled rather than deleted** to preserve the ability to re-enable them in the future without reinstallation. This follows a conservative change-management approach.

| Rule Disabled | Reason |
|---------------|--------|
| Core Networking — Teredo | IPv6 tunneling; not in use; unnecessary exposure |
| Core Networking — Multicast Listener Discovery, Neighbor Discovery, Router Advertisement/Solicitation | Not required for a single-segment DC with a static IP |
| Cortana — all rules | Consumer feature; no server function |
| Delivery Optimization (TCP/UDP-In) | Peer-to-peer update delivery; inappropriate for a production DC |
| Desktop App Web Viewer — all rules | No legitimate server use |
| DIAL Protocol Server | Cast-to-device / streaming protocol; no server function |
| FTP Server — all rules | FTP is not a required service per scope |
| mDNS | Not required in a small single-segment AD environment |
| File and Printer Sharing — LLMNR-UDP-In, NB-Datagram/Name/Session-In | LLMNR and NetBIOS are known attack vectors (LLMNR poisoning, NBT-NS spoofing); disabled per CIS Benchmark recommendation |

> **Why disable vs. delete?** Disabling preserves the rule configuration and allows for a clean rollback if a business requirement changes. Deleting a built-in Windows firewall rule group requires reinstallation of the associated server role to restore it.

---

### Part 2 — Services and Share Permissions

#### CORPDOCS Share — Least-Privilege Hardening

The CORPDOCS share at `C:\Shares\CORPDOCS` was intended to provide read-only access to authorized domain users. The initial configuration allowed inherited permissions that granted the Users group modify or write access.

**Steps taken:**

```
Right-click CORPDOCS > Properties > Security > Advanced
> Disable inheritance
> Convert inherited permissions into explicit permissions

Remove: Users (Modify), Users (Write), any other non-read entries
Retain:
  SYSTEM                          Full Control  (This folder, subfolders and files)
  IS3650\Administrators           Full Control  (This folder, subfolders and files)
  IS3650\Users                    Read          (This folder, subfolders and files)
  CREATOR OWNER                   Full Control  (Subfolders and files only)
```

**Rationale:** Granting only Read access to the Users group means that even if a domain account is compromised, the attacker cannot modify, delete, or overwrite the contents of CORPDOCS. This directly limits the blast radius of a credential compromise — a common attack path in AD environments.

---

#### Web Server Verification

IIS was confirmed operational by navigating to `http://10.0.5.4` from the Bodhi Linux VM on the MultimachineNAT segment. The IS3650 Homepage loaded successfully, confirming that the IIS firewall rules and web server bindings were correctly configured.

---

#### Windows Defender — Initial Scan

Before proceeding with software removal, Windows Defender was enabled and an immediate Quick Scan was run to establish a clean baseline:

```
Windows Security > Virus & threat protection > Quick scan

Result: 29,401 files scanned — 0 threats detected
Scan duration: 2 minutes 20 seconds
```

Real-time protection and cloud-delivered protection were both enabled before any further changes were made.

---

### Part 3 — Software Audit and OS Patching

#### Unauthorized / Unnecessary Software Removed

| Software | Finding | Action |
|----------|---------|--------|
| FileZilla | FTP client/server software. FTP is not a required service; its presence creates unnecessary attack surface and contradicts the principle of least functionality. | Uninstalled via Programs and Features. All FTP inbound firewall rules also disabled. |
| Winamp (media player) | Audio/media playback software. No legitimate domain controller function. | Uninstalled. |
| Winamp Detector Plug-in | Browser companion plug-in for Winamp. No server function. | Uninstalled. |
| **HoneyBOT 0.1.8** ⚠️ | Honeypot software that simulates vulnerable services to attract and log attacker connections. **Its presence on a production DC with no documented IT policy is a significant anomaly.** The most likely explanations are: (1) the departing admin was conducting unauthorized surveillance of internal network users, or (2) the machine was being used as an unauthorized research platform. Either scenario warrants a security investigation. | Removed immediately. Flagged as a potential insider threat indicator for further review. |
| LibreOffice 3.4 | Office suite — version 3.4 reached end-of-life in 2012 and carries numerous known CVEs. No legitimate DC function. | Uninstalled. |
| Google Chrome (outdated) | Installation path was broken/corrupted. Old version could not receive security updates. | Old version uninstalled; current version reinstalled. |

---

#### OS Patching

Windows Update showed multiple missing patches at the start of the audit.

```
Settings > Update & Security > Windows Update > Check for updates

Action: Downloaded and installed all available updates
Result:  "You're up to date" — Last checked: Today, 12:25 AM
Server restarted to apply kernel-level patches
```

**Rationale:** Unpatched systems are the most commonly exploited entry point in network intrusions. A domain controller running outdated OS code is particularly dangerous — a single exploited vulnerability on a DC can result in full domain compromise. This maps directly to NIST SP 800-123 patch management guidance.

---

#### DNS Forwarder Configuration

A DNS resolution issue was identified — the server lacked a working external DNS forwarder, blocking internet access needed for Windows Update.

```
Network adapter properties > IPv4 > DNS servers

Added: 8.8.8.8 (Google Public DNS) as secondary forwarder
Primary remains: 127.0.0.1 (local AD-integrated DNS)

Result: Internet connectivity restored. Windows Update and browser functional.
```

---

### Part 4 — User Account Audit and Password Policy

#### Pre-Remediation User Inventory

Opening Active Directory Users and Computers (`dsa.msc`) revealed the following non-built-in user accounts:

| Account | Status | Action |
|---------|--------|--------|
| Frida Antonov | ✅ Authorized — active employee | Retained; password reset |
| Karl Ivanovich | ✅ Authorized — active employee | Retained; password reset |
| Karina Feder | ❌ Unauthorized — no longer employed | **Removed** |
| Oleg Vasilov | ❌ Unauthorized — no longer employed | **Removed** |

> **Why stale accounts matter:** Orphaned accounts are a primary lateral movement vector in AD environments. A threat actor who obtains credentials for Karina or Oleg — through phishing, credential dumps, or password spray — would have a valid domain logon with no legitimate owner monitoring it. This is especially concerning given the circumstances of the previous admin's departure.

---

#### Remediation Actions

```
Active Directory Users and Computers (dsa.msc)
> is3650.com > Users

Deleted: Karina Feder, Oleg Vasilov

For Karl Ivanovich and Frida Antonov:
  > Right-click > Properties > Account tab
  > Check: "User must change password at next logon"
  > Password expiration: 365 days
  > Kerberos pre-authentication: Enabled (default — verified)
```

**Temporary passwords assigned (change required on first login):**

| Account | Temporary Password |
|---------|-------------------|
| Frida Antonov | `GeneralUserFrida` |
| Karl Ivanovich | `GeneralUserKarl` |

> These are intentionally generic placeholder credentials. Both accounts are configured to force an immediate password change on first login. The permanent passwords set by each user are governed by the domain password policy.

**Kerberos pre-authentication** was verified enabled for both accounts. Disabling pre-auth exposes accounts to AS-REP Roasting — an offline hash-cracking attack that does not require any interaction with the target account after the initial AS-REQ.

---

### Part 5 — Endpoint Protection

Windows Defender Antivirus was not enabled at the start of the audit — the server had **no active endpoint protection** whatsoever. This is a critical gap on any server, and especially serious on a DC that was already found running unauthorized software.

```
Windows Security > Virus & threat protection > Manage settings

Enabled: Real-time protection       → ON
Enabled: Cloud-delivered protection → ON

Initial scan result:
  Files scanned: 29,401
  Threats found: 0
  Duration: 2 minutes 20 seconds
```

**Real-time protection** continuously monitors for malware installation or execution events. **Cloud-delivered protection** provides access to Microsoft's live threat intelligence feed, enabling detection of novel threats before local signature updates are available.

Server Manager properties panel confirmed: `Windows Defender Antivirus: Real-Time Protection: On`

---

## Skills Demonstrated

- **Windows Server 2019 hardening** — DC-specific hardening while maintaining Active Directory, DNS, DHCP, IIS, and SMB service integrity
- **Windows Defender Firewall administration** — inbound/outbound rule auditing, attack surface reduction, rule scoping by protocol and network profile
- **NTFS and SMB permission hardening** — inheritance removal, explicit ACL configuration, least-privilege enforcement for domain user groups
- **Active Directory user account management** — stale account identification and removal, Kerberos pre-auth hardening, forced password rotation
- **Malware/threat detection** — endpoint protection deployment, initial scan, real-time protection configuration
- **Software inventory auditing** — unauthorized software identification and removal, including anomalous honeypot software (HoneyBOT)
- **OS patch management** — Windows Update audit and full patching cycle on a production server
- **Security risk reasoning** — prioritizing remediations by severity, explaining rationale for each decision, distinguishing between disable and delete for firewall rules
- **CIS Benchmark alignment** — LLMNR/NetBIOS/Teredo disable rationale maps directly to CIS Windows Server 2019 Benchmark recommendations

---

## Framework Mapping

| Framework | Control | Application |
|-----------|---------|-------------|
| NIST SP 800-123 | General Server Security | Patch management, service minimization, firewall configuration |
| NIST SP 800-53 — AC-2 | Account Management | Removal of stale accounts; least-privilege user provisioning |
| NIST SP 800-53 — CM-7 | Least Functionality | Disable unnecessary services, protocols, and software |
| NIST SP 800-53 — SI-3 | Malicious Code Protection | Endpoint protection deployment and real-time scanning |
| CIS Benchmark — Windows Server 2019 | Multiple | LLMNR disable, NetBIOS disable, Teredo disable, password policy |
| MITRE ATT&CK — T1171 | LLMNR/NBT-NS Poisoning | Mitigated by disabling LLMNR and NetBIOS inbound firewall rules |
| MITRE ATT&CK — T1078 | Valid Accounts | Mitigated by removing stale accounts and enforcing password rotation |
| MITRE ATT&CK — T1558.004 | AS-REP Roasting | Mitigated by verifying Kerberos pre-authentication on all user accounts |

---

## References

- NIST SP 800-123: Guide to General Server Security — https://csrc.nist.gov/publications/detail/sp/800-123/final
- NIST SP 800-53 Rev 5: Security and Privacy Controls — https://csrc.nist.gov/publications/detail/sp/800-53/rev-5/final
- CIS Benchmark for Windows Server 2019 — https://www.cisecurity.org/benchmark/microsoft_windows_server
- MITRE ATT&CK T1171 — LLMNR/NBT-NS Poisoning and SMB Relay — https://attack.mitre.org/techniques/T1171/
- MITRE ATT&CK T1078 — Valid Accounts — https://attack.mitre.org/techniques/T1078/
- MITRE ATT&CK T1558.004 — AS-REP Roasting — https://attack.mitre.org/techniques/T1558/004/
- Microsoft Docs — Windows Defender Firewall with Advanced Security — https://docs.microsoft.com/en-us/windows/security/threat-protection/windows-firewall/
- Microsoft Docs — Active Directory Security Best Practices — https://docs.microsoft.com/en-us/windows-server/identity/ad-ds/plan/security-best-practices/
- HoneyBOT by Atomic Software Solutions — https://www.atomicsoftwaresolutions.com/honeybot.php
