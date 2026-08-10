# Threat Intelligence Analysis - TRAM, MITRE ATT&CK Mapping & APT Report Analysis

## Overview

This lab demonstrates automated threat intelligence processing using TRAM (Threat Report ATT&CK Mapper), a machine learning tool developed by MITRE's Center for Threat-Informed Defense. TRAM ingests published threat intelligence reports in HTML format, applies NLP-based sentence analysis to extract behavioral indicators, and maps them to MITRE ATT&CK techniques with a confidence score. Three real-world APT and malware campaign reports were analyzed: the Lazarus APT BMP steganography RAT campaign, the BlackEnergy APT spear-phishing campaign, and the GrassCall fake job interview malware targeting Web3 job seekers. Results were reviewed, analyst-accepted mappings were finalized, and findings were exported as structured TRAM reports.


---

## Environment

| Component | Details |
|---|---|
| Analysis Machine | IR-Workstation VM - Linux (Investigator / L34rn1ng!) |
| Tool | TRAM (Threat Report ATT&CK Mapper) - Docker deployment |
| ML Model | TRAM NLP sentence classification |
| Framework | MITRE ATT&CK Navigator |
| Report 1 | Lazarus APT - BMP steganography RAT (ThreatDown/Malwarebytes) |
| Report 2 | BlackEnergy APT - What is BlackEnergy (Kaspersky Lab) |
| Report 3 | GrassCall malware - Crypto wallet theft via fake job interviews |

---

## Walkthrough

### Phase 1 - TRAM Deployment

TRAM runs as a containerized Django web application. Setup required editing the Docker Compose file to configure credentials:

```yaml
version: "3.5"
services:
  tram:
    image: ghcr.io/center-for-threat-informed-defense/tram:latest
    environment:
      - DATA_DIRECTORY=/tram/data
      - ALLOWED_HOSTS=["example host1", "localhost"]
      - DJANGO_SUPERUSER_USERNAME=administrator
      - DJANGO_SUPERUSER_PASSWORD=P4cktIRBook!
      - DJANGO_SUPERUSER_EMAIL=analyst@gosecurity.ninja
```

```bash
$ docker-compose up
```

TRAM's web interface was accessed at `http://localhost:8000`. Reports are uploaded as HTML files and processed by the ML pipeline, which takes 5-10 minutes per document to parse sentences, classify behavioral indicators, and generate ATT&CK technique mappings.

---

### Phase 2 - Lazarus APT Report Analysis

**Source:** ThreatDown (Malwarebytes) - "Lazarus APT conceals malicious code within BMP image to drop its RAT"

**Report summary:** The Lazarus Group, a North Korean state-sponsored APT, embedded a Remote Access Trojan (RAT) inside a Windows BMP image file. When parsed by a malicious document, the image's pixel data is extracted and executed as shellcode - a steganography-based delivery technique designed to evade detection.

**TRAM processed 436 sentences** and identified **36 ATT&CK techniques**. Key accepted mappings:

| Sentence Fragment | ATT&CK Technique | Confidence | Decision |
|---|---|---|---|
| "conceals malicious code within BMP image" | T1027 - Obfuscated Files or Information | 99.8% | Accepted |
| "sign in OneView sign in Partner Portal" | T1518.001 - Security Software Discovery | 48.8% | Reviewing |
| "Block DNS Filtering Mobile Security Services Managed Detection" | T1562.001 - Disable or Modify Tools | 75.0% | Accepted |
| "centralized visibility and management capabilities" | T1219 - Remote Access Software | 32.2% | Reviewing |
| "channel first mentality Managed Service Providers" | T1569.002 - Service Execution | 62.0% | Accepted |
| "Resellers Build growth, profitability, and customer loyalty" | T1543.003 - Windows Service | 26.1% | Reviewing |

**Full TRAM report techniques (36 total, partial list):**

T1003.001 (LSASS Memory), T1016 (System Network Configuration Discovery), T1027 (Obfuscated Files or Information), T1033 (System Owner/User Discovery), T1041 (Exfiltration Over C2 Channel), T1047 (WMI), T1053.005 (Scheduled Task), T1055 (Process Injection), T1056.001 (Keylogging), T1057 (Process Discovery), T1059.003 (Windows Command Shell), **T1059.007 (JavaScript)**, T1068 (Exploitation for Privilege Escalation), T1070.004 (File Deletion), T1071.001 (Web Protocols), T1078 (Valid Accounts), T1082 (System Information Discovery), T1083 (File and Directory Discovery), T1090 (Proxy), T1095 (Non-Application Layer Protocol), T1105 (Ingress Tool Transfer), T1106 (Native API), T1113 (Screen Capture), T1140 (Deobfuscate/Decode Files), T1204.002 (Malicious File), T1219 (Remote Access Software), T1518.001 (Security Software Discovery), T1543.003 (Windows Service), T1547.001 (Registry Run Keys/Startup Folder), T1548.002 (Bypass UAC), T1562.001 (Disable or Modify Tools), T1564.001 (Hidden Files and Directories), T1566.001 (Spearphishing Attachment), T1569.002 (Service Execution), T1573.001 (Symmetric Cryptography)

**ATT&CK Navigator mapping** was used to visually confirm technique placement. The T1059.007 (JavaScript) mapping was manually added per analyst review before exporting the final TRAM report as a `.docx` document.

---

### Phase 3 - BlackEnergy APT Report Analysis

**Source:** Kaspersky Lab - "BlackEnergy APT Attacks: What is BlackEnergy"

**Report summary:** The BlackEnergy APT group, active since at least 2014 and linked to destructive attacks on Ukrainian critical infrastructure, transitioned from a DDoS botnet tool to a sophisticated APT platform. The group distributed malicious Excel documents with macros via spear-phishing emails to gain initial access in targeted networks.

**TRAM processed 153 sentences** (Accepted: 2, Reviewing: 151). Key accepted mappings:

| Sentence Fragment | ATT&CK Technique | Confidence | Decision |
|---|---|---|---|
| "researchers discovered a new malicious document, which infects the system with BlackEnergy Trojan" | T1105 - Ingress Tool Transfer | 100.0% | Accepted |
| "actively using spear-phishing emails carrying malicious Excel documents with macros" | T1566.001 - Spearphishing Attachment | 100.0% | Accepted |

**Analyst rationale:** Both mappings were accepted at 100% confidence and align precisely with the report content. T1105 (Ingress Tool Transfer) describes the delivery of the BlackEnergy payload binary to compromised systems. T1566.001 (Spearphishing Attachment) directly matches the described delivery vector - malicious Excel files with embedded macros sent via targeted email campaigns. No overrides were necessary.

---

### Phase 4 - GrassCall Malware Campaign Analysis

**Source:** ThreatDown - "GrassCall malware campaign drains crypto wallets via fake job interviews"

**Report summary:** GrassCall is a social engineering campaign targeting Web3 job seekers. Victims are lured through fake job interview platforms that install a malicious "GrassCall" meeting application. The app deploys information-stealing malware that targets cryptocurrency wallet credentials and drains funds.

**TRAM processed 324 sentences** (Accepted: 2, Reviewing: 322). Key accepted mappings:

| Sentence Fragment | ATT&CK Technique | Confidence | Decision |
|---|---|---|---|
| "fake job interviews through a malicious GrassCall meeting app that installs" | T1204 - User Execution | 100.0% | Accepted |
| "installs information-stealing malware to steal cryptocurrency wallets" | T1056 - Input Capture | 100.0% | Accepted |

**Analyst rationale:** TRAM initially suggested T1566.001 (Spearphishing Attachment) for the first mapping, but T1204 (User Execution) was a better fit - the victim actively downloads and runs the fake meeting application rather than opening a malicious email attachment. The distinction matters: T1204 represents cases where the user is deceived into executing a seemingly legitimate program. T1056 (Input Capture) was accepted for the credential theft component because the malware intercepts wallet credentials through input monitoring, matching the ATT&CK technique definition precisely.

---

## Skills Demonstrated

- TRAM Docker deployment and configuration (Django + YAML)
- NLP-based threat report ingestion and sentence-level ATT&CK technique classification
- ML confidence score evaluation and analyst-driven mapping review
- MITRE ATT&CK technique identification across 36 techniques from a single report
- ATT&CK Navigator usage for technique visualization and manual mapping
- Analyst override of ML suggestions based on technique definition research
- Multi-report threat intelligence analysis across three distinct APT/malware campaigns
- Structured TRAM report export and documentation
- APT behavioral profiling: Lazarus Group (NK state), BlackEnergy (destructive), GrassCall (financial)

---

## ATT&CK Techniques Summary Across All Reports

| Technique | ID | Campaigns |
|---|---|---|
| Spearphishing Attachment | T1566.001 | BlackEnergy, Lazarus |
| Obfuscated Files or Information | T1027 | Lazarus |
| Ingress Tool Transfer | T1105 | BlackEnergy |
| User Execution | T1204 | GrassCall |
| Input Capture | T1056 | GrassCall |
| Process Injection | T1055 | Lazarus |
| Keylogging | T1056.001 | Lazarus |
| Exfiltration Over C2 | T1041 | Lazarus |
| Disable or Modify Tools | T1562.001 | Lazarus |
| JavaScript Execution | T1059.007 | Lazarus |

---

## References

- [TRAM - Center for Threat-Informed Defense](https://github.com/center-for-threat-informed-defense/tram)
- [MITRE ATT&CK Navigator](https://mitre-attack.github.io/attack-navigator/)
- [ThreatDown - Lazarus APT BMP RAT Report](https://www.threatdown.com/blog/lazarus-apt-conceals-malicious-code-within-bmp-image-to-drop-its-rat/)
- [Kaspersky - BlackEnergy APT](https://securelist.com/blackenergy-apt-attacks-in-ukraine-employ-spearphishing-with-word-documents/73440/)
- [MITRE ATT&CK Framework](https://attack.mitre.org/)
- Calopedas, M. et al. *Incident Response with Threat Intelligence*, Packt Publishing (Chapter 7)
