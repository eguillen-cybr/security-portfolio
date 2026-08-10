# Volatile Evidence Collection – MAGNET RAM Capture, Velociraptor & KAPE

## Overview

This lab demonstrates first-response procedures for collecting volatile and forensic evidence from a compromised Windows endpoint. Based on a simulated data breach scenario — a CEO's laptop suspected of compromise during international travel — three industry-standard tools are used in sequence: MAGNET RAM Capture for live memory acquisition, Velociraptor for memory dumps and targeted artifact collection via a scripted batch triage, and KAPE (Kroll Artifact Parser and Extractor) for comprehensive artifact collection with automated preprocessing. The lab reflects real incident response workflows where evidence volatility and chain-of-custody integrity are critical.


---

## Environment

| Component | Details |
|---|---|
| Suspect Machine | CorpLaptop VM — Windows (User: Michael Scott / L34rn1ng!, Admin: P4$$w0rd!) |
| IR Workstation | IRWorkstation VM — Linux (Investigator / L34rn1ng!) |
| Evidence Drive | E:\ (secondary virtual disk added to CorpLaptop VM) |
| Tool 1 | MAGNET RAM Capture v1.2.0 (MRCv120.exe) |
| Tool 2 | Velociraptor (Windows AMD64 binary) |
| Tool 3 | KAPE (gkape.exe GUI) |
| Framework | Incident Response with Threat Intelligence (Chapter 4 lab) |

---

## Scenario

Michael Scott, CEO of a global energy company, returned from an international conference in Asia. Shortly after, confidential strategic information — accessible only to a small group including the CEO — began circulating publicly. His laptop is the primary suspect. As lead investigator, the goal is to collect volatile memory and forensic artifacts from the live machine before any shutdown that would destroy RAM contents, active network connections, and process state.

**Key investigative questions guiding collection:**
- Were any new programs installed during the trip?
- Was the user redirected to malicious sites (browser history artifacts)?
- Is there evidence of data exfiltration in network connection logs?
- What does the process execution timeline show?

---

## Walkthrough

### Phase 1 — Directory Structure Setup

Before collecting evidence, a structured directory tree was created on the E:\ drive to maintain organization and chain of custody:

```
E:\
├── FirstResponseTools\
│   ├── Installers\         ← Tool executables
│   └── velociraptor\       ← Velociraptor binary and triage.bat
└── Evidence\
    ├── MemDump\            ← RAM images
    ├── Artifacts\          ← Velociraptor artifact ZIPs
    └── KAPE\
        ├── Target\         ← KAPE disk artifact collection (VHDX)
        └── Module\         ← KAPE preprocessed output (CSV/TXT)
```

Separating tools from evidence is standard forensic practice — it prevents accidental contamination and makes evidence packaging straightforward.

---

### Phase 2 — Memory Acquisition with MAGNET RAM Capture

MAGNET RAM Capture is a lightweight, standalone executable requiring no installation. It captures a full raw image of physical memory with minimal footprint.

**Steps:**
1. Launched `MRCv120.exe` as Administrator on CorpLaptop
2. Set segment size: **Don't split** (single image file)
3. Set output path: `E:\Evidence\MemDump\Corp-LT-Mem-01`
4. Clicked **Start**

**Result:** A raw memory image file was created at `E:\Evidence\MemDump\Corp-LT-Mem-01` capturing the full physical RAM state of the suspect machine at that point in time.

```
E:\Evidence\MemDump\
└── Corp-LT-Mem-01      ← Raw RAM image
```

**Why this matters:** RAM contains running processes, decrypted credentials, clipboard contents, open network sockets, and registry hives that are not written to disk. Once the machine is powered off, this evidence is permanently lost.

---

### Phase 3 — Memory and Artifact Collection with Velociraptor

Velociraptor extends beyond raw memory capture by collecting targeted forensic artifacts defined via a scripted triage batch file. This approach enables non-technical first responders to run a pre-configured collection with a single double-click.

#### 3a. Artifact Discovery

```cmd
E:\FirstResponseTools\velociraptor> velociraptor.exe artifacts list > artifacts.txt
```

This exports all supported artifact collectors to a text file for review. Three artifacts were selected for the triage:

| Artifact | Purpose |
|---|---|
| `Windows.Memory.Acquisition` | Full physical memory dump via WinPmem driver |
| `Windows.Forensics.Timeline` | Timeline of file system activity and program execution |
| `Windows.Applications.Edge.History` | Microsoft Edge browsing history (URLs, timestamps, users) |

#### 3b. Creating the Triage Batch File

```bat
@echo off
velociraptor.exe artifacts collect -v Windows.Memory.Acquisition --output ..\..\Evidence\MemDump\VeloMemDump.zip
velociraptor.exe artifacts collect -v Windows.Forensics.Timeline --output ..\..\Evidence\Artifacts\Timeline.zip
velociraptor.exe artifacts collect -v Windows.Applications.Edge.History --output ..\..\Evidence\Artifacts\EdgeHistory.zip
```

#### 3c. Execution and Results

```cmd
E:\FirstResponseTools\velociraptor> triage.bat
```

**Memory dump result:** `PhysicalMemory.raw` and `Windows.Memory.Acquisition.json` were created in the MemDump directory.

**Timeline result:** `Windows.Forensics.Timeline.json` was produced (zero bytes in this lab environment — the artifact collector ran but the target's activity timeline was empty).

**Edge history result:** `Windows.Applications.Edge.History.json` contained populated JSON records:

```json
{"User":"michael.scott","FullPath":"C:\\Users\\michael.scott\\AppData\\Local\\Microsoft\\Ec...
{"User":"michael.scott","FullPath":"...","title":"how to make delicious cookies - Google Search",...
{"User":"michael.scott","FullPath":"...","title":"how to make butter cookies - Google Search",...
{"User":"Administrator","FullPath":"C:\\Users\\administrator\\AppData\\Local\\Microsoft\\Ec...
```

The Edge history artifact captured browsing activity for both `michael.scott` and `Administrator` accounts, including search terms — directly relevant to establishing a timeline of the CEO's web activity before and after the suspected compromise.

#### 3d. Velociraptor Server — Offline Collector Generation

On the IR-Workstation (Linux), a Velociraptor server was configured to generate a standalone executable collector — useful when deploying first-response collection to sites without technical staff:

```bash
$ sudo ./velociraptor config generate -i        # Generate server/client config
$ sudo ./velociraptor --config server.config.yaml user add investigator --role administrator
$ sudo ./velociraptor --config server.config.yaml frontend -v
```

The web GUI at `https://127.0.0.1:8889` was used to build an offline collector with three artifacts:
- `Triage.collection.upload`
- `Windows.Applications.Edge.History`
- `Windows.Attack.Prefetch`

The resulting `Collector_velociraptor-*.exe` was transferred to the CorpLaptop and executed as Administrator, automatically collecting all configured artifacts without manual steps.

---

### Phase 4 — Comprehensive Artifact Collection with KAPE

KAPE (Kroll Artifact Parser and Extractor) provides both raw artifact collection (Target mode) and automated preprocessing of those artifacts (Module mode) — significantly reducing analysis time.

#### 4a. Configuration

**Target collection (left panel — what to collect):**

| Setting | Value |
|---|---|
| Target Source | C:\ |
| Target Destination | E:\Evidence\KAPE\Target |
| Targets | `!SANSTriage`, `EdgeChromium`, `WindowsTimeline` |
| Container Format | VHDX (virtual hard disk — forensically sound container) |

**Module processing (right panel — how to preprocess):**

| Setting | Value |
|---|---|
| Module Destination | E:\Evidence\KAPE\Module |
| Modules | `!EZParser`, `LiveResponse_NetworkDetails`, `Windows_Systeminfo`, `Velocidex_WinPmem` |
| Zip Password | L34rn1 |

#### 4b. Results — Module Output Directories

KAPE's module processing produced structured, analyst-ready CSV and text output organized by category:

**FileSystem/**
- `MFTECmd_$MFT_Output.csv` — 129,905 KB — complete Master File Table parse (every file on C:\)
- `MFTECmd_$J_Output.csv` — 50,906 KB — USN Journal (file change log)
- `MFTECmd_$SDS_Output.csv` — 1,529 KB — security descriptor data

**ProgramExecution/**
- `Amcache_ProgramEntries.csv` — 55 KB — recently executed programs
- `Amcache_AssociatedFileEntries.csv` — 50 KB
- `PECmd_Output.csv` — 2,063 KB — Prefetch file analysis (execution timestamps, run count)
- `PECmd_Output_Timeline.csv` — 76 KB

**Registry/**
- 30+ CSV files including: `TypedURLs`, `RecentDocs`, `UserAssist`, `KnownNetworks`, `Services`, `TaskCache`, `NetworkAdapters`, `MountedDevices`, `BamDam` (Background Activity Monitor)

**FileFolderAccess/**
- `michael.scott_ActivityPackage.csv`, `administrator_Activity.csv` — user file access activity
- `LECmd_Output.csv` — LNK file analysis (recently opened files via shortcuts)
- `AutomaticDestinations.csv`, `CustomDestinations.csv` — Jump List analysis

**SQLDatabases/**
- 15 ChromiumBrowser CSV files — Chrome/Edge history, downloads, cookies, favicons

**SystemActivity/**
- `SrumECmd_AppResource.csv` — 56 KB — application resource usage (CPU, network, disk per app)
- `SrumECmd_NetworkConnections.csv` — network connection history
- `SrumECmd_NetworkUsage.csv` — 16 KB — bytes sent/received per application

**EventLogs/**
- `EvtxECmd_Output.csv` — 78,750 KB — all Windows Event Log entries parsed to CSV

**LiveResponse/**
- `arp_cache.txt`, `dns_cache.txt`, `ipconfig.txt`, `netbios_cache.txt`, `netbios_sessions.txt`, `network_connections.txt`, `routing_table.txt`, `SystemInfo.csv`

**Target Output:**
```
E:\Evidence\KAPE\Target\
├── 2025-02-16T014751_KAPEVirtualdisk\
├── 2025-02-16T014751_ConsoleLog.txt        (86 KB)
└── 2025-02-16T014751_KAPEVirtualdisk.zip  (86,516 KB)
```

The VHDX container holds a forensic copy of all collected artifacts, mountable as a read-only volume for analysis.

---

## Skills Demonstrated

- Volatile memory acquisition using MAGNET RAM Capture
- Targeted artifact collection via Velociraptor batch triage scripting
- Velociraptor server deployment and offline collector generation
- KAPE GUI configuration for Target and Module collection
- Master File Table (MFT) and USN Journal acquisition
- Prefetch, Amcache, and UserAssist analysis for program execution history
- Registry artifact extraction (TypedURLs, RecentDocs, KnownNetworks, BamDam)
- Browser history forensics (Edge/Chromium via KAPE SQLDatabases module)
- SRUM database parsing for network and application usage history
- Windows Event Log bulk parsing to CSV
- Live response network state capture (ARP, DNS, netstat, routing table)
- Evidence directory structuring and chain-of-custody practices

---

## References

- [MAGNET RAM Capture – Magnet Forensics](https://www.magnetforensics.com/resources/magnet-ram-capture/)
- [Velociraptor Documentation](https://docs.velociraptor.app/)
- [KAPE Documentation – Kroll](https://www.kroll.com/en/services/cyber-risk/incident-response-litigation-support/kape)
- [EZTools by Eric Zimmerman](https://ericzimmerman.github.io/)
- [SANS Windows Forensic Analysis Poster](https://www.sans.org/posters/windows-forensic-analysis/)
- MITRE ATT&CK: [T1005 – Data from Local System](https://attack.mitre.org/techniques/T1005/), [T1074 – Data Staged](https://attack.mitre.org/techniques/T1074/)
- Calopedas, M. et al. *Incident Response with Threat Intelligence*, Packt Publishing
