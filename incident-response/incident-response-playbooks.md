# Incident Response Playbooks – DoD Manufacturing Firm

**Skills Demonstrated:** Incident response planning, playbook development, CMMI Level 2 process documentation, threat-specific investigation/remediation/recovery procedures

---

## Overview

Developed five incident response playbooks for a fictional 300-employee manufacturing firm with Department of Defense contracts, targeting CMMI Level 2 maturity for incident response processes. Each playbook covers **Investigation**, **Remediation**, and **Recovery** phases with specific tools, personnel, and procedures tailored to the organization's environment.

---

## Organization Profile

| Group | Count | Access Level |
|-------|-------|-------------|
| Production Workers | 270 | Worker portal, company email (Outlook.com), locked-down browsers, no outbound web |
| Maintenance/Janitorial | 5 | Same as production + approved supplier portals |
| Engineers | 10 | Engineering workstations (CAD/CAM, EE software), SharePoint portal, full email |
| Sales | 7 | CRM (cloud-hosted), full email, ~3-4x email volume of other groups |
| Executives | 3 | Full internet access, full email |
| HR | 2 | Full internet access, full email |
| IT/IT Security | 3 | Full unrestricted access to all resources |

---

## Playbook 1 – Credential Access via Password Stuffing

**Trigger:** A salesperson's personal Amazon account was compromised; password reuse on company systems is suspected.

### Investigate

- Verify the claim by checking authentication logs for failed logins, unknown IPs, or off-hours access using **Splunk** or **LogRhythm**
- Respectfully ask the employee to confirm whether the compromised password or a variation was reused on any work accounts (*"To protect company systems, could you confirm if this password or a variation of it has been used for accessing work accounts or tools?"*)
- Audit credential reuse across CRM, Outlook.com email, and SharePoint using SIEM cross-correlation
- Investigate the Amazon account itself for evidence of credential stuffing activity targeting other accounts
- Involve IT for log analysis and HR/Legal for compliance guidance on employee questioning

### Remediate

- Force password resets on all potentially affected accounts; prioritize CRM and email
- Deploy adaptive MFA on sensitive systems using **Okta** or **Duo Security**
- Issue targeted guidance to the affected employee on password manager adoption (**LastPass**, **1Password**)
- Conduct company-wide awareness communication on credential hygiene and reuse risks

### Recover

- Run a proactive security audit using **Nessus** or **Qualys**; simulate credential stuffing tests against key systems
- Update internal policy to mandate: password manager use, regular password strength checks, MFA enforcement on critical systems
- Deploy real-time anomaly detection dashboards via **SolarWinds Security Event Manager**
- Report incident resolution and long-term preventive measures to management

---

## Playbook 2 – Command and Control Malware on Engineering Workstation

**Trigger:** Endpoint protection detected malware on an engineering workstation communicating with a known C2 IP range.

### Investigate

- Capture and analyze network traffic from the compromised workstation using **Wireshark**, **Zeek**, or **Splunk** to confirm C2 communication patterns
- Analyze the malware sample in a sandbox environment (**Cuckoo Sandbox**, **ThreatGrid**) and compare IOCs against threat feeds (**VirusTotal**, **AlienVault OTX**)
- Scan adjacent workstations and the engineering subnet for lateral movement using **CrowdStrike** or **Microsoft Defender for Endpoint**
- Collaborate with network security to block the C2 IP at the firewall immediately

### Remediate

- Immediately isolate the infected workstation via **Cisco ISE** Network Access Control
- Remove malware using **Malwarebytes**, **Symantec**, or **Microsoft Defender**
- Reset all credentials used from the infected workstation; enforce MFA via **Okta** or **Duo Security**
- Conduct targeted phishing and malware awareness training for the engineering team

### Recover

- Reimage the workstation from a verified clean backup using **Acronis** or **Veeam**; apply all patches during rebuild
- Conduct post-reimage vulnerability scans with **Nessus** or **Rapid7 InsightVM**
- Reintroduce the workstation to the network only after IT and security sign-off
- Update patch management policy and implement mandatory security updates for all engineering workstations
- Monitor for residual C2 activity using **SolarWinds Security Event Manager** or **ELK Stack**

---

## Playbook 3 – Suspicious Data Transfer Volumes (Potential Exfiltration)

**Trigger:** A sales workstation is generating outbound traffic 3-4x larger than peers — already the highest-volume group in the organization.

### Investigate

- Review outbound traffic logs from the affected workstation using **Wireshark**, **NetFlow Analyzer**, or **Splunk** to identify destination IPs and transfer patterns
- Scan the workstation for malware or data exfiltration tools using **CrowdStrike** or **Microsoft Defender for Endpoint**
- Classify outgoing data using **Symantec DLP** or **Forcepoint DLP** to determine if proprietary CAD files, sales data, or customer PII is involved
- Coordinate with Legal and Compliance to ensure investigation methods meet regulatory requirements given DoD contract obligations

### Remediate

- Apply firewall blocking rules for suspicious destination IPs via **Cisco ASA** or **Palo Alto Networks**
- Isolate the workstation using **Cisco ISE** or **Aruba ClearPass** NAC
- Perform full malware scan with **Malwarebytes** or **Symantec Endpoint Protection**

### Recover

- Restore any altered or corrupted data from clean backups using **Acronis** or **Veeam**
- Implement data transfer volume thresholds and alerts in network monitoring tools
- Update firewall and DLP rules to flag or block large outbound transfers above established baselines
- Provide targeted security awareness training to the sales team on recognizing exfiltration tools and phishing

---

## Playbook 4 – Unknown Malware on Production Worker Workstation

**Trigger:** Endpoint anomaly detection flagged an executable that matches neither known system files nor known malware signatures.

*Note: Production worker workstations have tightly locked-down browsers and no outbound web access — infection vector is likely via the company email system (Outlook.com) or physical media.*

### Investigate

- Review endpoint detection logs to trace the suspicious executable's origin and behavior using **Splunk**, **LogRhythm**, or **CrowdStrike**
- Monitor for lateral movement or network connections using **Zeek**, **Wireshark**, or EDR solutions — given the locked-down nature of production machines, any outbound connection is highly anomalous
- Verify file integrity by comparing the executable against known-good system files using **Tripwire** or submitting the sample to **Cuckoo Sandbox** / **ThreatGrid**

### Remediate

- Quarantine the workstation using **Cisco ISE** or **Aruba ClearPass** NAC immediately
- Remove malicious files using **Malwarebytes**, **Symantec Endpoint Protection**, or **Microsoft Defender**
- Reset workstation credentials and enforce stronger password policies via **LastPass** or **1Password**

### Recover

- Reimage from a clean backup using **Acronis**, **Veeam**, or **Carbonite**
- Conduct post-recovery scans with **Nessus** or **Qualys** before reconnecting to the network
- Given production workers' limited technical exposure, provide targeted training on recognizing suspicious email attachments and reporting anomalous behavior
- Review the email filtering configuration on Outlook.com for the production worker group as the likely infection vector

---

## Playbook 5 – Information Leakage / Dark Web Data Exposure

**Trigger:** Proprietary documents — including sensitive employee data, engineering plans, and customer sales data — were discovered on a dark web site by a contact of an IT employee.

*Note: Given DoD contract obligations, this incident likely triggers mandatory breach notification requirements.*

### Investigate

- Validate the leaked documents using dark web monitoring tools (**Recorded Future**, **DarkOwl**) and assess scope — employee PII, engineering IP, and customer data are all potentially covered by different regulations
- Audit internal access logs for the leaked document types using **Splunk** or **LogRhythm** to identify who accessed the data and when
- Use **Exabeam** or **Splunk UBA** to analyze user behavior patterns and identify insider threat indicators or negligent access
- Involve IT Security, HR, Legal, and executive leadership immediately; DoD contract obligations may require government notification

### Remediate

- Disable any compromised credentials immediately; enforce password resets across affected systems
- If insider involvement is identified, collaborate with HR for appropriate personnel action and preserve evidence for potential legal proceedings
- File breach notifications with required regulatory authorities (CMMC/DFARS requirements for DoD contractors, applicable state breach notification laws for customer PII)

### Recover

- Notify affected employees, customers, and partners with clear communication about scope and mitigation steps; use **Proofpoint** or **Mimecast** for secure notification delivery
- Deploy dark web surveillance tools (**Digital Shadows**, **SpyCloud**) for ongoing monitoring of additional leaks
- Implement least-privilege access controls across the SharePoint engineering portal, CRM, and HR systems
- Apply encryption to sensitive data at rest using **Vormetric** or **Microsoft Azure Information Protection**
- Conduct mandatory security awareness training across all employee groups, emphasizing secure data handling, phishing recognition, and reporting obligations

---

## Skills Demonstrated

- Incident response playbook development (Investigation / Remediation / Recovery)
- CMMI Level 2 process documentation
- Threat-specific tool selection and justification
- Personnel and resource allocation planning for incident response
- Regulatory and compliance awareness (DoD/CMMC, DFARS, GDPR/CCPA)
- Insider threat and exfiltration scenario planning
- Multi-stakeholder coordination (IT, HR, Legal, executive leadership)

---

## References

- [NIST SP 800-61r2 – Computer Security Incident Handling Guide](https://csrc.nist.gov/publications/detail/sp/800-61/rev-2/final)
- [CMMC Framework](https://www.acq.osd.mil/cmmc/)
- [Incident Response Playbooks – GitHub Community Resources](https://github.com/certsocietegenerale/IRM)
