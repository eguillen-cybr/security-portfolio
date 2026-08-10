# Incident Response Team Design — Sovereign (AI/ML Enterprise)

## Overview

This writeup documents the design and justification of a Computer Security Incident Response Team (CSIRT) for **Sovereign**, a fictional enterprise-scale AI and machine learning company. The CSIRT was developed following the **NIST SP 800-61r2** incident response framework and covers team structure, staffing, and response plans for three distinct incident types. This exercise demonstrates practical knowledge of IR team composition, escalation paths, and how organizational context shapes incident response strategy.

---

## Environment

| Component | Details |
|---|---|
| **Company** | Sovereign (fictional) |
| **Industry** | Artificial Intelligence & Machine Learning |
| **Size** | 1,001+ employees |
| **Headquarters** | New York City, NY — 600 employees, primary IT infrastructure |
| **Regional Offices** | London, UK (200 employees); Singapore (150 employees) |
| **Remote Workforce** | ~100 employees globally via VPN |
| **OS Environment** | Hybrid Windows / Linux |
| **Infrastructure** | On-premises data centers + AWS and Azure cloud |
| **Database** | PostgreSQL |
| **Security Stack** | Firewalls, IDPS, WAF, load balancers, SIEM |
| **Framework Used** | NIST SP 800-61r2 |

---

## IT Organizational Structure

```
Chief Information Officer (CIO)
└── IT Security Manager
    ├── Network Engineers (5)       — On-prem and cloud infrastructure
    ├── System Administrators (8)   — System reliability and performance
    ├── Developers (10)             — Internal tools and customer-facing apps
    └── Helpdesk Support (10)       — Global employee technical support
```

The IT Security Manager has direct ownership of the CSIRT and reports incidents upward to the CIO. Regional offices in London and Singapore operate satellite IT teams that interface with the central CSIRT through the Incident Response Coordinator.

---

## CSIRT Team Composition

The CSIRT is staffed to handle incidents across Sovereign's hybrid, multi-region environment. Roles were selected based on NIST guidance for organizations with complex infrastructure and regulatory exposure.

| Role | Headcount | Responsibilities |
|---|---|---|
| **Incident Response Coordinator** | 1 | Oversees all response activities; primary liaison to external entities (legal, law enforcement, regulators, MSSP) |
| **Forensic Analyst** | 2 | In-depth analysis of compromised systems; evidence collection and preservation; chain of custody |
| **Security Analyst** | 2 | Threat monitoring and detection via SIEM; triage and initial containment; IOC identification |
| **Network Specialist** | 1 | Network-layer incident containment; firewall/IDPS rule modification; traffic analysis |
| **Communication Specialist** | 1 | External communications to clients, stakeholders, and regulatory bodies |

**Operating Model:** The CSIRT operates during business hours with an on-call rotation for after-hours incidents. Advanced forensic capabilities beyond internal team scope are augmented through a contracted **Managed Security Service Provider (MSSP)**.

---

## Incident Response Plans

All three plans follow the NIST four-phase lifecycle: **Preparation → Detection & Analysis → Containment/Eradication/Recovery → Post-Incident Activity.**

---

### IR Plan 1 — Phishing Attack

**Trigger:** Suspicious email reported by employee; alert from email security gateway; credential use from anomalous location.

**Response Steps:**

**Detection & Analysis**
```
1. Security Analyst reviews email headers, sender reputation, embedded URLs/attachments
2. Cross-reference IOCs (sender domain, IP, hash) against threat intel feeds
3. Query SIEM for other users who received the same email
4. Determine if any credentials were submitted or malware executed
```

**Containment**
```
5. Isolate affected user accounts — disable or force password reset
6. Block sender domain and malicious URLs at email gateway and WAF
7. Preserve email artifacts for forensic analysis
```

**Eradication & Recovery**
```
8. Remove phishing emails from all mailboxes organization-wide
9. Restore account access after credential reset and MFA verification
10. Review IDPS/firewall logs for any C2 beacon activity post-click
```

**Post-Incident**
```
11. Document timeline and attack vector in incident report
12. Deploy targeted phishing awareness training for affected department
13. Update email filtering rules based on TTPs observed
```

---

### IR Plan 2 — Ransomware Attack

**Trigger:** Endpoint detection alert; sudden mass file encryption; ransom note discovered on file shares.

**Response Steps:**

**Detection & Analysis**
```
1. Security Analyst confirms ransomware activity via SIEM and EDR alerts
2. Identify patient zero — determine initial infection vector (phishing, RDP, exploit)
3. Map lateral movement — identify all affected hosts and shared drives
4. Assess encryption scope: local files only vs. network shares vs. cloud-synced data
```

**Containment**
```
5. Immediately isolate infected systems from the network (disable NIC or quarantine VLAN)
6. Network Specialist blocks lateral movement paths — isolate affected network segments
7. Disable compromised user accounts to prevent further propagation
8. Preserve memory dumps and disk images of infected hosts before remediation
```

**Eradication & Recovery**
```
9. Wipe and reimage infected systems from known-good baselines
10. Restore data from clean AWS/Azure backups — verify backup integrity before restore
11. Validate restored systems are clean before reconnecting to the network
12. Patch the exploited vulnerability or close the abused attack vector
```

**Post-Incident**
```
13. Conduct full forensic analysis to determine full attack chain
14. File regulatory notifications if client data was encrypted/exfiltrated
15. Review and harden backup strategy — ensure backups are air-gapped from production
16. Update IDPS signatures and firewall rules based on observed TTPs
```

---

### IR Plan 3 — Data Breach

**Trigger:** Anomalous data exfiltration detected by DLP or SIEM; unauthorized API access; third-party notification; client-reported credential exposure.

**Response Steps:**

**Detection & Analysis**
```
1. Security Analyst identifies the compromised data set — scope, classification, affected clients
2. Forensic Analyst determines attack vector: credential theft, SQL injection, insider threat, API abuse
3. Identify all systems touched by the attacker — review access logs, API logs, DB query logs
4. Determine exfiltration method and volume — DNS tunneling, HTTP POST, cloud storage upload, etc.
```

**Containment**
```
5. Revoke compromised credentials and API keys immediately
6. Block exfiltration channels identified (IP blocks, domain blocks, egress firewall rules)
7. Preserve all relevant logs, DB transaction records, and network captures for forensic review
8. Engage MSSP for advanced forensic support if breach scope exceeds internal capacity
```

**Eradication & Recovery**
```
9. Patch or remediate the exploited vulnerability
10. Rotate all credentials and secrets associated with compromised systems
11. Audit all third-party integrations and API access permissions
12. Restore affected systems from clean backups if integrity was compromised
```

**Post-Incident**
```
13. Communication Specialist notifies affected clients per contractual and regulatory requirements
14. File required regulatory disclosures (GDPR for EU clients, applicable US state laws)
15. Conduct root cause analysis — publish internal lessons-learned report
16. Implement additional DLP controls and anomaly detection rules based on exfiltration method
```

---

## Skills Demonstrated

- NIST SP 800-61r2 incident response lifecycle application
- CSIRT team design and role justification for enterprise-scale organizations
- Incident response planning for phishing, ransomware, and data breach scenarios
- Understanding of hybrid cloud/on-prem IR considerations (AWS, Azure, on-prem)
- Escalation path design including MSSP integration and external stakeholder communication
- Regulatory awareness — GDPR, client notification obligations, evidence preservation

---

## References

- NIST SP 800-61r2 — *Computer Security Incident Handling Guide* — https://csrc.nist.gov/publications/detail/sp/800-61/rev-2/final
- ENISA — *Good Practice Guide for Incident Management* — https://www.enisa.europa.eu/publications/good-practice-guide-for-incident-management
- MITRE ATT&CK — Phishing (T1566), Data Encrypted for Impact (T1486), Exfiltration techniques — https://attack.mitre.org
