# Emmanuel Guillen - Security Portfolio

**B.S. Cybersecurity, Southern Utah University 2025**  
Location: El Monte, CA | Target Role:  Help Desk / IT Support → Cybersecurity (SOC Analyst track)
Email: emmanuel.guillen.cybr@gmail.com | LinkedIn: https://linkedin.com/in/emmanuel-guillen-cybr

---

## About This Portfolio

30+ independent lab writeups built over four years of hands-on coursework and self-directed study. Every writeup follows the same format: overview, environment, walkthrough with real commands and output, skills demonstrated, and references. No filler. No theory-only labs.

The labs were filtered, anything that didn't demonstrate a real, deployable security skill was cut. What's here reflects what I can actually do.

---

## Skill Coverage at a Glance

| Domain | Tools and Techniques |
|---|---|
| SIEM / Monitoring | TheHive 5.0, Cortex, Suricata, Sysmon (custom XML config), Velociraptor, KAPE |
| Threat Intelligence | MITRE ATT&CK, TRAM, IOC analysis, TTP mapping - Lazarus Group, BlackEnergy, GrassCall |
| Network Forensics | Wireshark, NetworkMiner, PCAP analysis, credential carving, file recovery |
| Endpoint Forensics | Volatility (pslist/malfind/cmdscan), FTK Imager, Autopsy, Talon, MAGNET RAM Capture |
| Disk and File Forensics | FAT16/NTFS/FAT32, MFT analysis, magic byte carving, EXIF extraction, Plaso timelines |
| Mobile Forensics | Cellebrite UFED, SQLite accounts.db, OAuth2 token recovery (Android + macOS) |
| Malware Analysis | FlareVM, msfvenom, AV evasion, static trojan analysis, PuTTY backdoor injection |
| Incident Response | NIST SP 800-61r2 playbooks, CSIRT design, tabletop exercise design |
| Governance | 12-domain enterprise security plan - NIST CSF, ISO 27001, GDPR, HIPAA, FISMA |
| Offensive / Red Team | Metasploit, EternalBlue, Burp Suite, OSINT, post-exploitation, web app hacking |
| Scripting | Bash, PowerShell, Python |

---

## Repository Structure

```
security-portfolio/
├── red-team/               # 10 writeups - offensive ops, OSINT, exploitation
├── blue-team/              # 21 writeups - detection, forensics, hardening, threat intel
├── incident-response/      # 2 writeups - IR playbooks, CSIRT design (Sovereign)
├── malware-analysis/       # 2 writeups  - binary payloads, trojaning, AV evasion
└── capstone/               # CIA-spec covert communications framework (RFP-CIA-073)
```

---

## Highlighted Writeups

These are the labs that best demonstrate day-one SOC analyst skills.

### Blue Team
| Writeup | What It Shows |
|---|---|
| [Sysmon Endpoint Monitoring](blue-team/sysmon-windows-endpoint-monitoring.md) | Deployed and tuned Sysmon with custom XML config; hunted threats via Event IDs 1, 22, 23, 4624/4625 |
| [IDPS - Suricata on IPFire](blue-team/idps-suricata-ipfire.md) | Deployed Suricata with emerging-scan.rules; DNS TLD monitoring, compromised IP alerting, false positive methodology |
| [TheHive Case Management](blue-team/thehive-case-management.md) | TheHive 5.0 + Cortex analyzers (Maltiverse, TeamCymru, CyberCrime-Tracker) for structured incident case management |
| [Threat Intelligence Analysis](blue-team/threat-intelligence-tram-mitre-attack-mapping.md) | Mapped Lazarus Group, BlackEnergy, GrassCall TTPs to MITRE ATT&CK; identified detection opportunities |
| [Volatile Evidence Collection](blue-team/volatile-evidence-collection-magnet-velociraptor-kape.md) | Live memory acquisition with MAGNET; Velociraptor triage scripts; KAPE targeting MFT, Prefetch, SRUM, Event Logs |
| [RAM Forensics - Volatility](blue-team/ram-forensics-volatility.md) | Process analysis, malware detection (malfind), command history, privilege inspection against memory images |
| [Network Forensics](blue-team/network-forensics-networkminer-wireshark.md) | PCAP analysis with Wireshark and NetworkMiner; credential extraction; file carving from captured sessions |

### Red Team
| Writeup | What It Shows |
|---|---|
| [Metasploit and EternalBlue](red-team/metasploit-basics-eternalblue-meterpreter.md) | MS17-010 exploitation; Meterpreter post-exploitation (hashdump, keylogging, remote shell) |
| [OSINT and Intelligence Gathering](red-team/osint-intelligence-gathering-workday-pagerduty.md) | Passive recon with Shodan, SecurityTrails, theHarvester; CVE cross-referencing |
| [Web Application Hacking](red-team/web-hacking-burpsuite-juiceshop.md) | OWASP Juice Shop via Burp Suite; SQLi, broken access control, cryptographic failures |

### Incident Response
| Writeup | What It Shows |
|---|---|
| [IR Playbooks - Sovereign](incident-response/incident-response-playbooks.md) | 5 NIST SP 800-61r2 playbooks: ransomware, insider threat, DDoS, exfiltration, phishing |
| [CSIRT Design - Sovereign](incident-response/incident-response-team-sovereign.md) | Role definitions, escalation matrix, communication protocols, tabletop exercise design |

---

## Certifications

- CompTIA Security+ - In Progress (Expected October 2026)
- B.S. Cybersecurity - Southern Utah University, 2025 - GPA: 3.6

---

## Capstone: Covert Operations Network Security Framework

Envision Solutions / SUU - April 2025 - a simulated RFP response exercise designing a theoretical covert comms system to hypothetical intelligence-community specifications.

Designed Phantom, a custom E2EE covert communications app built to CIA field operation specifications. Key components:

- MFA stack: PIN + biometric + CAC with internal PKI/CA and Titan M2 hardware key storage
- Quantum-resistant encryption: FIPS 203 / ML-KEM / CRYSTALS-Kyber to counter Harvest Now, Decrypt Later attacks
- Anti-detection: domain fronting, protocol mimicry, timing obfuscation, LSB steganography, autonomous cover traffic
- NIST SP 800-61 IR protocol, FIPS 140-3/DoD compliance, and a $4.5M phased deployment budget

---

All writeups use real tools, real output, and real methodology. No simulated screenshots.
