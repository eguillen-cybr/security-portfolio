# Metasploit Database & Workspaces - PostgreSQL Integration, Host Management & Loot

## Overview

This lab demonstrates Metasploit's built-in PostgreSQL database integration for structured penetration test data management. Rather than running modules ad hoc and losing results, the database backend enables persistent storage of discovered hosts, open ports, services, credentials, and post-exploitation loot - all organized into isolated workspaces per engagement. A multi-target environment (Windows 7 and Metasploitable 3 Win2k8) is scanned and exploited, with results queried, filtered, and cross-referenced directly from the msfconsole database interface.

---

## Environment

| Component | Details |
|---|---|
| Attacker OS | Kali Linux (kali / kali) |
| Target 1 | Windows 7 Professional SP1 x64 - 10.0.5.8 |
| Target 2 | Metasploitable 3 Windows Server 2008 R2 - 10.0.5.5 |
| Network | VirtualBox NAT Network - MultimachineNAT (10.0.5.0/24) |
| Database | PostgreSQL (managed via `msfdb`) |
| Framework | Metasploit Framework 6 (msfconsole) |
| Key Commands | `db_status`, `workspace`, `db_nmap`, `hosts`, `services`, `loot` |
| Post Module | `post/windows/gather/enum_services` |

---

## Walkthrough

### Phase 1 - Database Initialization

Modern Kali Linux with MSF6 manages the PostgreSQL backend automatically. No manual service start is needed:

```bash
$ sudo msfdb init
# or to initialize and launch msfconsole in one step:
$ sudo msfdb run
```

```
msf6 > db_status
[*] Connected to msf. Connection type: postgresql.
```

Database connectivity confirmed. All subsequent scan results, host data, service records, and loot will be automatically persisted.

---

### Phase 2 - Workspace Management

Workspaces isolate engagement data - each can hold a separate set of hosts, services, credentials, and loot. This mirrors real-world pentest organization (one workspace per client, subnet, or assessment phase).

```
msf6 > workspace -a labA
[*] Added workspace: labA
[*] Workspace: labA

msf6 > workspace -a labB
[*] Added workspace: labB
[*] Workspace: labB

msf6 > workspace labB
[*] Workspace: labB
```

The active workspace is now `labB`. Any scans, exploits, and loot from this point are stored under `labB` and will not appear when switching to `labA` or the default workspace.

---

### Phase 3 - Database-Integrated Nmap Scan (`db_nmap`)

`db_nmap` runs Nmap with the same syntax as the standalone tool, but automatically imports all discovered hosts, ports, services, and OS fingerprints into the Metasploit database:

```
msf6 > db_nmap -A 10.0.5.5 10.0.5.8
```

The `-A` flag enables OS detection, service version detection, script scanning, and traceroute. Results are stored immediately - no manual import needed.

```
msf6 > hosts

Hosts
=====

address    mac                name  os_name       os_flavor  os_sp  purpose  info  comments
-------    ---                ----  -------       ---------  -----  -------  ----  --------
10.0.5.5   08:00:27:97:5e:5a        Windows 2008                    server
10.0.5.8   08:00:27:64:be:80        Windows 2008                    server
```

Both targets are identified as Windows Server 2008 hosts with their MAC addresses recorded. OS fingerprinting correctly identifies both (Windows 7 and Server 2008 R2 share the same NT 6.1 kernel, which Nmap reports as Windows 2008).

---

### Phase 4 - OS-Filtered Host Query and RHOSTS Population

The `hosts` command supports search and automatic variable population - one of the most efficient features for targeting multiple machines:

```
msf6 > hosts -S windows -R

Hosts
=====

address    mac                name  os_name       os_flavor  os_sp  purpose  info  comments
-------    ---                ----  -------       ---------  -----  -------  ----  --------
10.0.5.5   08:00:27:97:5e:5a        Windows 2008                    server
10.0.5.8   08:00:27:64:be:80        Windows 2008                    server

RHOSTS => 10.0.5.5 10.0.5.8
```

`-S windows` filters by OS name substring; `-R` automatically sets the `RHOSTS` variable to all matching IPs. This enables targeting every Windows host in the database with a single command - no manual IP entry needed.

---

### Phase 5 - TCP Port Scanner Against All Windows Hosts

With RHOSTS already populated from the previous step, the auxiliary TCP scanner runs against both targets simultaneously:

```
msf6 > use auxiliary/scanner/portscan/tcp
msf6 auxiliary(scanner/portscan/tcp) > run

[+] 10.0.5.5:  - 10.0.5.5:22   - TCP OPEN
[+] 10.0.5.5:  - 10.0.5.5:21   - TCP OPEN
[+] 10.0.5.5:  - 10.0.5.5:80   - TCP OPEN
[+] 10.0.5.5:  - 10.0.5.5:135  - TCP OPEN
[+] 10.0.5.5:  - 10.0.5.5:139  - TCP OPEN
[+] 10.0.5.5:  - 10.0.5.5:445  - TCP OPEN
[+] 10.0.5.5:  - 10.0.5.5:1617 - TCP OPEN
[+] 10.0.5.5:  - 10.0.5.5:3306 - TCP OPEN
[+] 10.0.5.5:  - 10.0.5.5:3389 - TCP OPEN
[+] 10.0.5.5:  - 10.0.5.5:3700 - TCP OPEN
[+] 10.0.5.5:  - 10.0.5.5:4848 - TCP OPEN
[+] 10.0.5.5:  - 10.0.5.5:5985 - TCP OPEN
[+] 10.0.5.5:  - 10.0.5.5:7676 - TCP OPEN
[+] 10.0.5.5:  - 10.0.5.5:8009 - TCP OPEN
[+] 10.0.5.5:  - 10.0.5.5:8080 - TCP OPEN
[+] 10.0.5.5:  - 10.0.5.5:8181 - TCP OPEN
[+] 10.0.5.5:  - 10.0.5.5:8443 - TCP OPEN
[+] 10.0.5.5:  - 10.0.5.5:9200 - TCP OPEN
[+] 10.0.5.5:  - 10.0.5.5:9300 - TCP OPEN
... (30+ open ports total on MSP3)
[+] 10.0.5.8:  - 10.0.5.8:135  - TCP OPEN
[+] 10.0.5.8:  - 10.0.5.8:139  - TCP OPEN
[+] 10.0.5.8:  - 10.0.5.8:445  - TCP OPEN
[+] 10.0.5.8:  - 10.0.5.8:5357 - TCP OPEN
[*] Scanned 2 of 2 hosts (100% complete)
```

Metasploitable 3 exposes an intentionally large attack surface: MySQL (3306), RDP (3389), WinRM (5985), Elasticsearch (9200/9300), GlassFish (4848/8080/8181), FTP (21), SSH (22), and HTTP (80). All discovered ports are stored in the database automatically.

---

### Phase 6 - Service-Filtered Database Query

The `services` command queries the database for specific ports or protocols - no re-scanning required:

```
msf6 > services -p 139

Services
========

host       port  proto  name         state  info
----       ----  -----  ----         -----  ----
10.0.5.5   139   tcp    netbios-ssn  open   Microsoft Windows netbios-ssn
10.0.5.8   139   tcp    netbios-ssn  open   Microsoft Windows netbios-ssn
```

Both hosts have NetBIOS Session Service (port 139) open - confirming SMB is active and both are viable targets for SMB-based attacks including EternalBlue, relay attacks, and credential capture.

---

### Phase 7 - Post-Exploitation Service Enumeration and Loot Storage

After exploiting MSP3 Win2k8 with EternalBlue and obtaining a Meterpreter session:

```
msf6 > use post/windows/gather/enum_services
msf6 post(windows/gather/enum_services) > set SESSION 1
msf6 post(windows/gather/enum_services) > run
[*] Enumerating services on METASPLOITABLE3...
```

**Sample services enumerated (partial):**

| Service Name | Account | Start Type | Binary Path |
|---|---|---|---|
| elasticsearch-service-x64 | LocalSystem | Auto | C:\Program Files\elasticsearch-1.1.1\...|
| ftpsvc | LocalSystem | Auto | C:\Windows\system32\svchost.exe -k ftp |
| jenkins | NT AUTHORITY\LocalService | Auto | C:\Program Files\jenkins\jenkins.exe |
| jmx | NT AUTHORITY\LocalService | Auto | C:\Program Files\jmx\jmx.exe |
| msiserver | LocalSystem | Manual | C:\Windows\system32\msiexec.exe /V |
| wampapache | NT AUTHORITY\LOCAL SERVICE | Auto | c:\wamp\bin\apache\apache2.2.21\bin\... |
| wampmysqld | LocalSystem | Auto | c:\wamp\bin\mysql\mysql5.5.20\bin\... |

```
[+] Loot file stored in: /home/kali/.msf4/loot/20250303005656_labB_10.0.5.5_windows.services_617455.txt
```

Results are automatically stored as a loot file tagged to the `labB` workspace and the specific host IP.

**Notable services for further exploitation:**
- **Jenkins** - CI/CD server with known RCE vulnerabilities; often runs as SYSTEM
- **Elasticsearch 1.1.1** - critically outdated; vulnerable to remote code execution (CVE-2014-3120, CVE-2015-1427)
- **WAMP (Apache + MySQL)** - web stack with default credentials risk
- **JMX** - Java Management Extensions; remote monitoring interface exploitable for code execution

**Loot items of interest identified in `/home/kali/.msf4/loot/`:**

**LANManServer** - The Windows Server service enabling SMB file and printer sharing. If misconfigured (e.g., null sessions enabled, weak share permissions), an attacker can intercept or access shared resources without authentication. Reviewing the service configuration file reveals share names and permission settings that could be leveraged for lateral movement.

**DHCP service** - Dynamic Host Configuration Protocol service configuration. A misconfigured or rogue DHCP server can be used to perform DHCP starvation or IP spoofing, redirecting victim traffic through an attacker-controlled gateway - enabling man-in-the-middle attacks across the entire subnet.

---

## Skills Demonstrated

- Metasploit PostgreSQL database initialization and status verification
- Workspace creation, switching, and engagement isolation
- Database-integrated Nmap scanning (`db_nmap -A`) with automatic host/service import
- Host database querying with OS-based filtering (`hosts -S`)
- Automatic RHOSTS population from database query results (`hosts -R`)
- TCP auxiliary port scanner against multiple hosts from RHOSTS
- Service-level database filtering by port number (`services -p`)
- Post-exploitation service enumeration (`post/windows/gather/enum_services`)
- Loot database storage and retrieval (`loot` command)
- Identification of high-value services for further exploitation (Jenkins, Elasticsearch, JMX)
- Multi-host penetration test workflow organization

---

## References

- [Metasploit Unleashed - Using Databases](https://www.offensive-security.com/metasploit-unleashed/using-databases/)
- [Metasploit Framework - Workspaces](https://docs.metasploit.com/docs/using-metasploit/intermediate/metasploit-database-support.html)
- [CVE-2014-3120 - Elasticsearch RCE](https://nvd.nist.gov/vuln/detail/CVE-2014-3120)
- [Jenkins Security Advisories](https://www.jenkins.io/security/advisories/)
- MITRE ATT&CK: [T1046 - Network Service Discovery](https://attack.mitre.org/techniques/T1046/), [T1007 - System Service Discovery](https://attack.mitre.org/techniques/T1007/), [T1213 - Data from Information Repositories](https://attack.mitre.org/techniques/T1213/)
