# RAM Forensics — Volatility & Bulk Extractor
### Memory Analysis: Process Enumeration, DLL Inspection, Command History, Privilege Escalation Detection


---

## Overview

Volatile memory (RAM) contains artifacts that exist nowhere else on a system: running processes, loaded DLLs, decrypted file contents, active network connections, recently typed commands, and injected shellcode. Unlike disk forensics, RAM analysis operates against a snapshot in time — once the system is powered off, this evidence is gone unless it was captured. This makes memory forensics a critical skill in incident response and malware analysis.

This lab analyzes a Windows XP SP3 memory image (`xp-laptop-2005-07-04-1430.img`) using Volatility 2 and Bulk Extractor within the CAINE forensic Linux distribution. Six Volatility modules are executed and interpreted, and Bulk Extractor is run against the raw memory image to extract network and communication artifacts.

---

## Environment

| Component | Detail | Role |
|-----------|--------|------|
| Analysis OS | CAINE 11 (Linux) | Forensic analysis VM |
| Memory Image | xp-laptop-2005-07-04-1430.img | Windows XP SP3 RAM dump (captured July 4, 2005, 14:30) |
| Volatility | v2.6 | Memory forensics framework |
| Profile | WinXSP3x86 | Windows XP SP3 32-bit memory profile |
| Bulk Extractor | v1.x | Pattern extraction from raw memory bytes |
| BEViewer | GUI frontend | Histogram and artifact browsing |
| Photorec | v7.x | File carving from memory dump |

---

## Findings Summary

| Module | Command | Key Finding |
|--------|---------|-------------|
| pslist | Process listing | Normal Windows XP process tree; multiple AV processes running simultaneously |
| dlllist | DLL enumeration | Logitech MouseWare DLLs loaded — unusual peripheral software but not malicious |
| cmdscan | Command history | 20 cmd.exe commands recovered; drive enumeration pattern visible; final command was `dd` memory capture |
| filescan | File handle listing | Standard system files; no anomalous executables or keyloggers identified |
| getsids | Security Identifier mapping | No unauthorized privilege escalation detected; all processes running under expected SIDs |
| malfind | Injected code detection | 6 memory regions flagged; all confirmed as normal Windows processes after investigation |

---

## Walkthrough

### Module 1 — pslist (Process List)

**Command:**

```bash
volatility -f /home/student/Downloads/xp-laptop-2005-07-04-1430.img \
  --profile=WinXSP3x86 pslist
```

**Forensic value:** `pslist` enumerates all running processes at the time the memory image was captured. It reads the Windows kernel's process list (EPROCESS linked list) and outputs each process's name, PID (Process ID), PPID (Parent PID), thread count, handle count, and start time. This is typically the first module run in any memory investigation — it establishes what was running.

**Sample output:**

```
Offset(V)   Name             PID   PPID  Thds  Hnds  Sess  Wow64  Start
0x823c87c0  System           4     0     62    1133  ----  0
0x8214b020  smss.exe         400   4     3     21    ----  0      2005-07-04
0x821c11a8  csrss.exe        456   400   11    551   0     0      2005-07-04
0x814dc020  winlogon.exe     480   400   18    522   0     0      2005-07-04
0x815221c8  services.exe     524   480   17    321   0     0      2005-07-04
0x821d8248  lsass.exe        536   480   20    369   0     0      2005-07-04
0x814f0020  svchost.exe      680   524   19    206   0     0      2005-07-04
```

**Interpretation:** The process tree follows a normal Windows XP boot sequence: System → smss.exe (Session Manager) → csrss.exe (Client/Server Runtime) and winlogon.exe → services.exe and lsass.exe. This parent-child relationship is a key indicator of legitimacy. Malware often injects itself under unexpected parents (e.g., `svchost.exe` spawned by `explorer.exe` rather than `services.exe`).

One notable finding: cross-referencing PIDs revealed multiple antivirus processes running simultaneously, which is unusual and can indicate a previous compromise attempt or simply poor system configuration (running two AV products simultaneously causes conflicts and is a security misconfiguration).

---

### Module 2 — dlllist (DLL List)

**Command:**

```bash
volatility -f /home/student/Downloads/xp-laptop-2005-07-04-1430.img \
  --profile=WinXSP3x86 dlllist
```

**Forensic value:** `dlllist` enumerates all Dynamic Link Libraries (DLLs) loaded by each running process. Malware frequently uses DLL injection to hide within legitimate processes — a malicious DLL loaded inside `explorer.exe` or `svchost.exe` will appear to be part of a trusted process. Unexpected DLL paths (e.g., a DLL loaded from `%TEMP%` or `%APPDATA%` rather than `System32`) are a major red flag.

**Sample output for one process:**

```
Command line: C:\Program Files\Logitech\MouseWare\system\em_exec.exe
Service Pack: 2

Base        Size      LoadCount  LoadTime  Path
0x00400000  0xc000    0xffff               C:\Program Files\Logitech\MouseWare\system\em_exec.exe
0x7c900000  0xb0000   0xffff               C:\WINDOWS\system32\ntdll.dll
0x7c800000  0xf4000   0xffff               C:\WINDOWS\system32\kernel32.dll
0x10000000  0x3e000   0xffff               C:\Program Files\Logitech\MouseWare\system\EVENTEX.dll
0x00320000  0x1d000   0xffff               C:\WINDOWS\system32\COMNCTR.dll
0x6c370000  0xf2000   0xffff               C:\Program Files\Logitech\MouseWare\system\MFC42.DLL
0x77c10000  0x58000   0xffff               C:\WINDOWS\system32\MSVCRT.dll
0x77f10000  0x46000   0xffff               C:\WINDOWS\system32\GDI32.dll
```

**Interpretation:** Logitech MouseWare is a peripheral driver application — its DLLs loading from `C:\Program Files\Logitech\` is expected and legitimate. All other DLLs loaded from `C:\WINDOWS\system32\` — the expected location for Windows system libraries. No DLLs loading from suspicious paths (temp directories, user profile folders, or non-standard locations) were identified. No malicious DLL injection indicators were found in the examined processes.

---

### Module 3 — cmdscan (Command History)

**Command:**

```bash
volatility -f /home/student/Downloads/xp-laptop-2005-07-04-1430.img \
  --profile=WinXSP3x86 cmdscan
```

**Forensic value:** `cmdscan` recovers command history from `cmd.exe` process memory. It reads the CommandHistory structures maintained by `csrss.exe` (the Windows console manager). This can reveal what a user or attacker typed at a command prompt — directory navigation, file operations, network commands, or tool execution — even after the console window was closed.

**Output:**

```
CommandProcess: csrss.exe  Pid: 456
CommandHistory: 0x4e4d88  Application: cmd.exe  Flags: Allocated, Reset
CommandCount: 20  LastAdded: 19  LastDisplayed: 19
FirstCommand: 0  CommandCountMax: 50

Cmd #0:  dd
Cmd #1:  cd\
Cmd #2:  dr
Cmd #3:  ee
Cmd #4:  e:
Cmd #5:  e:
Cmd #6:  dr
Cmd #7:  d:
Cmd #8:  d:
Cmd #9:  dr
Cmd #10: ls
Cmd #11: cd Docu
Cmd #12: cd Documents and
Cmd #13: dr
Cmd #14: d:
Cmd #15: cd dd\
Cmd #16: cd UnicodeRelease
Cmd #17: dr
Cmd #18: dd
Cmd #19: dd if=\\.\PhysicalMemory of=c:\xp-2005-07-04-1430.img conv=noerror
```

**Interpretation:** Command #19 is the memory capture command itself — `dd if=\\.\PhysicalMemory of=c:\xp-2005-07-04-1430.img conv=noerror`. This confirms how the memory image was acquired: the examiner (or the machine's operator) used `dd` to copy physical memory directly to a file. The commands preceding it show drive enumeration (`d:`, `e:`) and directory navigation — the operator was locating the `UnicodeRelease` directory on the D: drive, which contains the dd binary for Windows. This is a clean acquisition chain with no suspicious activity.

---

### Module 4 — filescan (File Handle Scan)

**Command:**

```bash
volatility -f /home/student/Downloads/xp-laptop-2005-07-04-1430.img \
  --profile=WinXSP3x86 filescan
```

**Forensic value:** `filescan` scans memory pool allocations to find FILE objects — structures the Windows kernel creates when a process opens a file. This recovers references to files that were open at capture time, including files that may have been deleted from disk but whose handles were still held in memory. Particularly useful for finding executables, keyloggers, and malicious payloads that were running from temp directories.

**Notable entries in output:**

```
0x000000001f6fa8d8  1  R--rwd  \Device\HarddiskVolume1\Documents and Settings\Sarah
0x000000001f6faa00  3  RWD---  \Device\HarddiskVolume1\$Directory
0x000000001f784690  0  -WD---  \Device\HarddiskVolume1\Documents and Settings\Sarah\My D[ocuments]
0x000000001f784958  1  R--rwd  \Device\HarddiskVolume1\[P]ROGRA~1\COMMON~1\SYMANT~1\VIRUS[Definitions]
0x000000001f784bd0  1  R--rwd  \Device\HarddiskVolume1\[Bt]oexec.bat
0x000000001f800620  1  R--r-d  \Device\HarddiskVolume1\[W]INDOWS\system32\drivers\LSWLNDS.[sys]
```

**Interpretation:** The file handles show a user named "Sarah" had documents and settings directories open. A Symantec antivirus definitions path is visible, corroborating the AV presence from pslist. The files accessed appear to be standard user activity — documents, system drivers, antivirus components. No suspicious executables running from temp directories, no keylogger binaries, no unexpected network drivers were identified.

---

### Module 5 — getsids (Security Identifier Mapping)

**Command:**

```bash
volatility -f /home/student/Downloads/xp-laptop-2005-07-04-1430.img \
  --profile=WinXSP3x86 getsids
```

**Forensic value:** `getsids` maps Windows Security Identifiers (SIDs) to each running process. SIDs identify which user account a process is running under. Malware that has successfully escalated privileges will show `S-1-5-32-544` (the Administrators group) for processes that should be running under a standard user SID. This module detects privilege escalation and identifies which user account was associated with each running process.

**Sample output:**

```
firefox.exe (3276): S-1-2-0 (Local Users with ability to log in locally)
PluckSvr.exe (3352): S-1-5-21-1957994488-484763869-854245398-1006 (Sarah)
PluckSvr.exe (3352): S-1-5-21-1957994488-484763869-854245398-513 (Domain Users)
PluckSvr.exe (3352): S-1-1-0 (Everyone)
PluckSvr.exe (3352): S-1-5-32-544 (Administrators)
PluckSvr.exe (3352): S-1-5-32-545 (Users)
PluckSvr.exe (3352): S-1-5-4 (Interactive)
PluckSvr.exe (3352): S-1-5-11 (Authenticated Users)
dd.exe (3300): S-1-5-21-1957994488-484763869-854245398-1006 (Sarah)
dd.exe (3300): S-1-5-32-544 (Administrators)
```

**Interpretation:** `PluckSvr.exe` (a Pluck RSS reader service) is running under both user "Sarah" (SID -1006) and the Administrators group (S-1-5-32-544). This is expected for a service that was installed with administrative rights. The `dd.exe` process also shows the Administrators SID — necessary because reading `\\.\PhysicalMemory` requires administrative access.

No process showed unexpected privilege elevation. There were no standard user processes that had acquired Administrator SIDs through injection or exploit. The SID distribution was consistent with normal operation.

---

### Module 6 — malfind (Malicious Code Detection)

**Command:**

```bash
volatility -f /home/student/Downloads/xp-laptop-2005-07-04-1430.img \
  --profile=WinXSP3x86 malfind
```

**Forensic value:** `malfind` scans process memory for regions that are: (1) executable, (2) not backed by a file on disk (VAD — Virtual Address Descriptor anomaly), and (3) contain code-like byte patterns. This is the primary indicator of process injection, shellcode, and reflective DLL loading — techniques used by most modern malware to hide within legitimate processes.

**Output:**

```
Process: [flagged process]  Pid: [PID]
Address: 0x02320000  Vad Tag: VadS  Protection: PAGE_EXECUTE_READWRITE

0x02320000  00 00 00 00 59 e9 c6 29 df ff e8 f5 ff ff ff 00
0x02320010  00 00 00 00 00 00 e8 e8 ff ff ff 0a 00 32 02

Disassembly:
0x02320000  0000         ADD [EAX], AL
0x02320002  0000         ADD [EAX], AL
0x02320004  59           POP ECX
0x02320005  e9c629dfff   JMP 0x21129d0
0x0232000a  e8f5ffffff   CALL 0x2320004
```

**6 regions flagged total.**

**Interpretation:** After cross-referencing each flagged region against the pslist output and known Windows XP process behavior, all 6 flagged regions were determined to be normal Windows process memory characteristics for this OS version — not injected code. Windows XP used a looser memory permission model than modern Windows; `PAGE_EXECUTE_READWRITE` regions were common in legitimate processes of this era. The disassembly patterns (ADD [EAX], AL followed by JMP/CALL) are characteristic of standard x86 code alignment padding, not shellcode stubs.

**Conclusion:** No active malware injection detected in this image.

---

### Bulk Extractor on Memory Image

**Command:**

```bash
bulk_extractor -o be_ram_output \
  /home/student/Downloads/xp-laptop-2005-07-04-1430.img
```

**Notable artifacts extracted:**

`email.txt` — email addresses present in memory at capture time:

```
rbs@maths.uq.edu.au
webmaster@espn.go.com
customer.service@espn.go.com
matthew_rothenberg@ziffdavis.com
Kathy_Badertscher@InfoWorld.com
webmaster@InfoWorld.com
PCMOnline@ziffdavis.com
webmaster@theregister.co.uk
pater@slashdot.org
beta@pluck.com
inet@microsoft.com
```

These email addresses were resident in RAM from browser activity, email clients, or cached web content. The presence of `pluck.com` addresses corroborates the PluckSvr.exe RSS reader process identified in pslist and getsids.

`ip_histogram.txt` — most frequent IP addresses in memory:

```
n=5374   192.168.2.7      (LAN address — local peer)
n=1244   66.135.211.87    (eBay infrastructure)
n=1097   170.224.8.51
n=419    66.151.149.10
n=398    66.150.96.111
```

The dominant 192.168.2.7 LAN address confirms the machine was on a local network. The eBay IP frequency suggests active browser activity on eBay at time of capture.

---

## Skills Demonstrated

- **Volatility 2** — profile identification, module execution, output interpretation
- **pslist** — process tree analysis, parent-child relationship validation
- **dlllist** — DLL injection detection, path-based anomaly identification
- **cmdscan** — command history recovery, acquisition chain reconstruction
- **filescan** — file handle enumeration, open file artifact recovery
- **getsids** — SID mapping, privilege escalation detection
- **malfind** — injected code detection, false positive analysis, disassembly interpretation
- **Bulk Extractor on memory** — email/IP artifact extraction from volatile memory
- **Memory forensics reasoning** — understanding what RAM contains that disk does not

---

## Framework Mapping

| Framework | Control | Application |
|-----------|---------|-------------|
| NIST SP 800-86 | Forensic Techniques | Volatile memory collection and analysis |
| NIST SP 800-61 Rev 2 | Incident Response | Memory analysis as part of IR evidence collection |
| MITRE ATT&CK — T1055 | Process Injection | What malfind detects and why |
| MITRE ATT&CK — T1003 | Credential Dumping | lsass.exe process analysis; why it's a target |
| MITRE ATT&CK — T1059.003 | Windows Command Shell | cmdscan recovery of cmd.exe history |
| SANS FOR508 | Advanced Incident Response | Memory forensics methodology |

---

## References

- Volatility Framework — https://github.com/volatilityfoundation/volatility
- Volatility 2 Command Reference — https://github.com/volatilityfoundation/volatility/wiki/Command-Reference
- NIST SP 800-86: Guide to Integrating Forensic Techniques — https://csrc.nist.gov/publications/detail/sp/800-86/final
- The Art of Memory Forensics (Ligh, Case, Levy, Walters) — Wiley, 2014
- SANS FOR508: Advanced Incident Response, Threat Hunting, and Digital Forensics — https://www.sans.org/cyber-security-courses/advanced-incident-response-threat-hunting-training/
- MITRE ATT&CK T1055 Process Injection — https://attack.mitre.org/techniques/T1055/
- Bulk Extractor — https://github.com/simsong/bulk_extractor
