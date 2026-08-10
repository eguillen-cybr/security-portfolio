# TheHive 5.0 – Cybersecurity Case Management & Cortex Analysis

**Tools:** TheHive 5.0, Cortex, Maltiverse, TeamCymru, CyberCrime-Tracker  
**Skills Demonstrated:** CSIRT case management, artifact analysis, threat intelligence integration, multi-org administration

---

## Overview

Deployed and configured TheHive 5.0 as a full cybersecurity case management platform. Simulated a real CSIRT workflow by creating incident cases, assigning tasks to analysts, running automated artifact analysis through Cortex, and integrating third-party threat intelligence analyzers.

---

## Environment

| Component | Details |
|-----------|---------|
| Platform | TheHive 5.0 (VM) |
| Analysis Engine | Cortex (integrated) |
| Analyzers Used | Maltiverse, TeamCymru MHR, CyberCrime-Tracker |
| Artifact Types | File hashes, URLs |

---

## Lab Walkthrough

### 1. Case Creation & Task Assignment

- Logged in as the built-in `thehive` demo user
- Created a new **Ransomware Incident** case using an Empty Case template with Severity: HIGH, TLP: RED, PAP: RED
- Added investigation tasks modeled after the book's ransomware playbook (e.g., *Triage and artifacts collection*, *Containment*, *Forensic analysis*)
- Created a new analyst user (`emmanuel.guillen@thehive.local`) and assigned all tasks to that user
- Tasks were visible in the Tasks tab with Assignee and Activity populated

**Key takeaway:** TheHive's case management allows CSIRT leads to delegate work across analysts and track task status and activity logs per case — critical for multi-person incident response.

---

### 2. Playbook Case Template & Observable Analysis

- Imported a ransomware playbook case template from the GitHub source linked in the course text
- Assigned all playbook tasks to the analyst user
- Added a malware file hash as an **Observable** (type: hash)
- Ran all available **Cortex analyzers** against the hash observable

**Maltiverse Analysis Result:**
- Hash identified as **Generic Malware / Hybrid-Analysis** classification
- 89 antivirus detections across engines
- PE32 executable (GUI), Intel 80386, for MS Windows
- File size: 912,264 bytes
- First seen: Feb 2024 | Last seen: Sep 2024
- Malicious score: 10/10

---

### 3. Cortex – TeamCymru MHR Integration

- Logged in as `orgadmin` to view Cortex job history
- Located the **TeamCymru MHR** analyzer job for the submitted hash
- Viewed the JSON job report:

```json
{
  "summary": {
    "taxonomies": [
      {
        "level": "info",
        "namespace": "TeamCymruMHR",
        "predicate": "last_seen",
        "value": "2024-12-29 23:18:00"
      },
      {
        "level": "info",
        "namespace": "TeamCymruMHR",
        "predicate": "detection_pct",
        "value": "44"
      }
    ]
  },
  "full": {
    "last_seen": "2024-12-29 23:18:00",
    "detection_pct": "44"
  },
  "success": true
}
```

**Findings:** Hash was last seen Dec 29, 2024, with a 44% detection rate across TeamCymru's sensor network — consistent with a moderately prevalent malware sample.

---

### 4. Multi-Organization Administration

- Logged in as `admin` (superadmin)
- Created a new organization named **Emmanuel**
- Created user `daniel.sanchez@thehive.local` with roles: `read`, `analyze`, `orgadmin`
- Assigned user to the Emmanuel organization

---

### 5. Custom Analyzer – CyberCrime-Tracker

- Logged into the new org as `daniel.sanchez`
- Configured the **CyberCrime-Tracker** analyzer via the Cortex First Start guide (created API key on the external service)
- Submitted a known malicious URL artifact: `rangemaster777u[.]ru/adm/frmcp0/`
- Ran the analyzer and reviewed the job report:

```json
{
  "summary": {
    "taxonomies": [
      {
        "level": "malicious",
        "namespace": "CCT",
        "predicate": "C2 Search",
        "value": "1 hit"
      }
    ]
  },
  "full": {
    "results": [
      {
        "date": "01-01-1970",
        "url": "rangemaster777u.ru/adm/frmcp0/",
        "ip": "",
        "type": "SpyEye",
        "vt_latest_scan": "https://www.virustotal.com/latest-scan/..."
      }
    ]
  },
  "success": true
}
```

**Findings:** URL confirmed as a **SpyEye C2 server** with 1 hit in CyberCrime-Tracker's database. TLP: AMBER, PAP: AMBER.

---

## Skills Demonstrated

- Cybersecurity case management (TheHive 5.0)
- CSIRT task delegation and workflow tracking
- Artifact/observable ingestion and automated analysis (Cortex)
- Threat intelligence enrichment (Maltiverse, TeamCymru, CyberCrime-Tracker)
- Multi-organization and multi-role user administration
- Interpretation of malware analysis reports (detection rates, C2 identification, file classification)

---

## References

- [TheHive Project](https://thehive-project.org/)
- [Cortex Analyzers](https://github.com/TheHive-Project/Cortex-Analyzers)
- [Incident Response with Threat Intelligence (O'Reilly)](https://learning.oreilly.com/library/view/incident-response-with/9781801072953/)
