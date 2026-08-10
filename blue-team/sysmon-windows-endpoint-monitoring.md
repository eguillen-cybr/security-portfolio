# Windows Endpoint Monitoring — Sysinternals Sysmon

## Overview

This lab demonstrates deployment and operational use of Sysinternals Sysmon (System Monitor) as a host-based telemetry agent on Windows. Sysmon extends the Windows Event Log with high-fidelity process, network, DNS, and file system events that are not captured by default Windows logging. The lab covers Sysmon installation, default configuration review, XML-based configuration authoring for DNS query logging and file deletion monitoring, event analysis in Windows Event Viewer, and practical threat detection scenarios for two additional event types. This workflow mirrors the initial deployment and tuning phase of Sysmon in a real SOC environment, where the tool is used to feed a SIEM with endpoint telemetry for detection and investigation.

---

## Environment

| Component | Details |
|---|---|
| Target OS | Windows 10 (VirtualBox VM) — DESKTOP-CN295K6 |
| Tool | Sysmon v15.14 (System Monitor) / SysmonDrv kernel driver |
| Deployment Path | `D:\Sysmon\` |
| Schema Version | 4.90 |
| Hash Algorithm | SHA256 |
| Event Log Path | `Microsoft-Windows-Sysmon/Operational` |

---

## Walkthrough

### Step 1 — Installation and Default Configuration

Sysmon64 was downloaded from the Microsoft Sysinternals site and installed from an Administrator command prompt. After installation, the current configuration was verified:

```
D:\>sysmon64 -c

System Monitor v15.14 - System activity monitor
By Mark Russinovich and Thomas Garnier
Copyright (C) 2014-2024 Microsoft Corporation
Sysinternals - www.sysinternals.com

Current configuration:
- Service name:        Sysmon64
- Driver name:         SysmonDrv
- Config file:         D:\Sysmon\sysmon64.exe -accepteula -i
- HashingAlgorithms:   SHA256
- Network connection:  disabled
- Archive Directory:   -
- Image loading:       disabled
- CRL checking:        enabled
- DNS lookup:          enabled

No rules installed
```

By default, Sysmon logs three event types: **Event ID 1** (Process Create), **Event ID 5** (Process Terminate), and **Event ID 16** (Sysmon configuration change). Network connections and image loading are disabled by default to reduce log volume. DNS lookup is enabled, meaning Sysmon resolves IP addresses in network events to hostnames.

---

### Step 2 — Process Creation Logging (Event ID 1)

Notepad was opened to generate a Process Create event. In Windows Event Viewer under `Applications and Services Logs > Microsoft > Windows > Sysmon > Operational`, Event ID 1 was located:

```
Process Create:
RuleName: -
UtcTime: 2024-05-19 23:44:45.143
ProcessGuid: {dc705821-8eed-664a-810e-00000000ed00}
ProcessId: 13520
Image: C:\Windows\System32\notepad.exe

Log Name:     Microsoft-Windows-Sysmon/Operational
Source:       Sysmon
Event ID:     1
Task Category: Process Create (rule: ProcessCreate)
Level:        Information
User:         SYSTEM
Computer:     DESKTOP-CN295K6
```

The Process Create event captures the full image path (`C:\Windows\System32\notepad.exe`), a unique ProcessGuid for correlation across events, the ProcessId, and a UTC timestamp. In a threat hunting context, this allows an analyst to identify when a suspicious process (e.g., `powershell.exe` spawning from `winword.exe`) was created, correlate it with network connections or file writes using the ProcessGuid, and build a complete timeline of attacker activity.

---

### Step 3 — DNS Query Logging (Event ID 22)

A custom Sysmon configuration file was authored to enable DNS query logging:

```xml
<Sysmon schemaversion="4.72">
    <EventFiltering>
        <DnsQuery onmatch="exclude" />
    </EventFiltering>
</Sysmon>
```

The `onmatch="exclude"` directive with no filter conditions means "exclude nothing" — effectively logging all DNS queries. The configuration was saved as `dns_config.xml` and loaded:

```
D:\>Sysmon64 -c Sysmon\dns_config.xml

System Monitor v15.14 - System activity monitor
Loading configuration file with schema version 4.72
Sysmon schema version: 4.90
Configuration file validated.
Configuration updated.
```

After browsing to several websites, Event ID 22 (DNS Query) entries appeared in Event Viewer. A representative entry:

```
Dns query:
RuleName: -
UtcTime: 2024-05-20 00:15:10.694
ProcessGuid: {dc705821-960e-664a-e70f-00000000ed00}
ProcessId: 19540
QueryName: login.live.com

Log Name:     Microsoft-Windows-Sysmon/Operational
Event ID:     22
Task Category: Dns query (rule: DnsQuery)
User:         SYSTEM
Computer:     DESKTOP-CN295K6
```

DNS logging is particularly valuable for detecting:
- **Domain Generation Algorithm (DGA) malware** — unusual, high-entropy domain names that don't match any known service
- **C2 beaconing** — repeated DNS queries to the same domain at regular intervals from a non-browser process
- **DNS tunneling** — unusually long DNS query strings used to exfiltrate data
- **Suspicious TLDs** — queries for `.onion`, `.bit`, or newly registered domains associated with phishing or C2

The `ProcessId` field is critical — it links the DNS query to a specific process. A DNS query from `svchost.exe` is normal; the same query from `notepad.exe` or a process running from `%TEMP%` is a major red flag.

---

### Step 4 — File Deletion Logging (Event ID 23/26)

A configuration file was authored to detect file deletion events:

```xml
<Sysmon schemaversion="4.72">
    <EventFiltering>
        <FileDelete onmatch="exclude"/>
    </EventFiltering>
</Sysmon>
```

This configuration logs all file deletion and overwrite events system-wide. After loading the config, a test file was created and deleted. Windows Event Viewer showed Event ID 23 (File Delete archived) entries at high volume — the system generates many file deletion events during normal operation (prefetch cache management, temp file cleanup, etc.).

A representative file deletion event:

```
File Delete logged:
RuleName: -
UtcTime: 2021-09-17 17:34:50.571
ProcessGuid: {447ac445-d869-6144-1400-000000000400}
ProcessId: 988
User: NT AUTHORITY\SYSTEM
Image: C:\Windows\System32\svchost.exe
TargetFilename: C:\Windows\Prefetch\NOTEPAD.EXE-D8414F97.pf
Hashes: SHA256=719DAF5536FEF9307FC83E301F451E6E7BDC10D62BC50D6D0FBB56201BB32AA2
IsExecutable: false
```

File deletion monitoring is particularly relevant for detecting:
- **Ransomware activity** — ransomware typically deletes shadow copies and original files after encryption; Sysmon captures both the deletion events and the process responsible
- **Log tampering** — an attacker deleting Windows Event Logs or application logs to cover tracks will generate file deletion events in Sysmon, which writes to its own protected log channel
- **Malware cleanup** — droppers that delete themselves after deploying a payload

The hash values in file deletion events are especially useful — if the deleted file's hash matches a known malware signature, the deletion event itself becomes evidence even if the file is gone.

---

### Step 5 — Additional Event IDs for Security Monitoring

#### Event ID 4624 — Successful Account Logon

Event ID 4624 logs every successful authentication event on a Windows system, including local interactive logins, network logons, and service account authentications. Each event records the account name, logon type, source IP address, authentication package, and timestamp.

**Security scenario:** In a corporate environment with defined working hours (09:00–17:00), a 4624 event for a domain administrator account at 03:47 from an IP address in a foreign country is an immediate high-priority alert. Combined with a SIEM correlation rule that baselines each account's normal logon hours and source IPs, Event ID 4624 enables detection of credential theft, pass-the-hash attacks, and unauthorized remote access. The logon type field further distinguishes interactive logins (type 2), network logins (type 3), and service logins (type 5) — a service account that suddenly generates a type 2 (interactive) logon is highly anomalous.

#### Event ID 4625 — Failed Account Logon

Event ID 4625 logs every failed authentication attempt, capturing the target account name, failure reason code, source IP, and timestamp.

**Security scenario:** A single 4625 event is noise. But 50 failed logon attempts against `Administrator` from the same source IP within 60 seconds is a textbook brute-force attack. A SIEM rule correlating 4625 events by source IP and account target, with a threshold of N failures within a time window, automatically generates a brute-force alert and can trigger an automated block via firewall API or Active Directory account lockout. Combining 4625 (failed attempts) with 4624 (eventual success) reveals whether a brute-force attack succeeded — a 4624 following a burst of 4625 events from the same source is a critical incident indicator.

---

## Skills Demonstrated

- **Sysmon deployment and configuration** — Installed Sysmon64, verified default event coverage (Event IDs 1, 5, 16), and understood the default logging posture
- **XML configuration authoring** — Wrote Sysmon configuration XML files for DNS query logging and file deletion monitoring; understood the `onmatch="exclude"` filter logic for catch-all logging
- **Process creation analysis** — Located and interpreted Event ID 1 entries in Windows Event Viewer; understood how ProcessGuid enables cross-event correlation for threat hunting
- **DNS telemetry** — Enabled Event ID 22 DNS query logging; interpreted entries showing QueryName and ProcessId for detection of DGA, C2 beaconing, and DNS tunneling
- **File deletion monitoring** — Enabled Event ID 23/26 file deletion logging; understood the security value of hash-enriched deletion events for ransomware and log tampering detection
- **Event Viewer navigation** — Sorted by Event ID, located specific events by timestamp, and read both the General and Details panes for complete event context
- **Threat detection scenario design** — Articulated detection scenarios for Event IDs 4624 and 4625 that demonstrate understanding of SIEM correlation logic, behavioral baselines, and incident escalation triggers
- **SOC telemetry pipeline** — Understood Sysmon as a data source for SIEM ingestion; recognized the relationship between endpoint event generation, SIEM rule thresholds, and analyst workflow

---

## References

- [Sysinternals Sysmon Documentation](https://docs.microsoft.com/en-us/sysinternals/downloads/sysmon)
- [Sysmon Community Guide — TrustedSec](https://github.com/trustedsec/SysmonCommunityGuide)
- [SwiftOnSecurity Sysmon Config](https://github.com/SwiftOnSecurity/sysmon-config)
- [Blumira — How to Enable Sysmon](https://www.blumira.com/enable-sysmon)
- [MITRE ATT&CK — Defense Evasion: Indicator Removal](https://attack.mitre.org/techniques/T1070/)
- [Windows Security Event Log Reference — Microsoft](https://docs.microsoft.com/en-us/windows/security/threat-protection/auditing/security-auditing-overview)
