# Network Forensics – NetworkMiner & Wireshark PCAP Analysis

**Tools:** NetworkMiner 2.x, Wireshark  
**Skills Demonstrated:** Network packet capture analysis, credential extraction, file carving, forensic artifact identification

---

## Overview

Performed forensic analysis on two real-world network packet capture (PCAP) files using NetworkMiner and Wireshark. Identified hosts, extracted transmitted files, recovered credentials, mapped DNS history, and carved HTTP objects — demonstrating the kind of post-incident network forensics used in SOC investigations.

---

## Environment

| Component | Details |
|-----------|---------|
| Primary Tool | NetworkMiner 2.x (Windows) |
| Secondary Tool | Wireshark |
| PCAP Sources | Public forensic repositories (UMass traces, chrissanders/packets) |
| Analysis Type | Passive/offline forensic analysis |

---

## PCAP File 1 – Web Browsing Trace

### Summary

Loaded the first capture into NetworkMiner. The trace represented typical end-user web browsing activity from a single Ubuntu workstation (`192.168.0.51`).

**Forensic findings:**

| Category | Details |
|----------|---------|
| **Hosts** | 167 unique hosts identified, including Google, Mozilla, DoubleClick ad networks, and NTP servers |
| **Files** | 674 files extracted — RPM packages (libvirt, Firefox, Mesa), HTML pages, CSS, JavaScript |
| **Images** | 36 images extracted including UI assets from openoffice.org and browser chrome elements |
| **Credentials** | 6 credential entries — HTTP cookies including session tokens (AMSESSIONID, PREF cookies) for Google and local services |
| **Sessions** | 588 sessions tracked — mix of HTTP (port 80) and SSL (port 443) connections |
| **DNS** | 1,422 DNS queries/responses providing a full domain browsing history |

### Forensic Significance

- **Hosts tab** maps the entire network topology and can identify unauthorized devices
- **Files tab** can surface executables, documents, or malware dropped over HTTP
- **Credentials tab** captures session tokens that could be replayed to impersonate users — critical in insider threat or account takeover investigations
- **DNS history** provides a timeline of all domains contacted, useful for C2 detection
- **Images** can be examined for steganography or contraband content

---

## PCAP File 2 – Mixed Protocol Trace

### Summary

Second capture contained a smaller host footprint with a Windows machine (`10.0.52.164`) communicating with various external hosts including Expedia, Google, ZoneLabs, and OpenOffice mirrors.

**Forensic findings:**

| Category | Details |
|----------|---------|
| **Hosts** | 24 hosts — internal IPs (10.0.52.x, 10.10.10.x, 10.11.11.x) and external (Expedia, Google, ZoneLabs, OpenOffice, Yahoo) |
| **OS Fingerprinting** | Windows host at 10.0.52.164 identified; Linux hosts at several external IPs |
| **Files** | 64 files — HTML pages, CSS, GIFs, octet-stream objects from ZoneLabs |
| **Images** | 36 images — OpenOffice UI assets, logos, and icons |
| **Credentials** | 6 entries — HTTP cookies from Expedia and Google; no plaintext passwords recovered, but usernames linked to timestamps |
| **Sessions** | 17 sessions, all HTTP port 80 |

### Forensic Significance

- **OS fingerprinting** allows investigators to narrow down which physical machine generated traffic
- **ZoneLabs octet-stream objects** flagged as potentially suspicious — unknown binary content transferred from a security vendor IP
- **Credential timestamps** enable investigators to reconstruct a timeline of user activity even without plaintext passwords

---

## File Extraction with Wireshark

Used Wireshark's built-in **Export Objects → HTTP** function on the second PCAP to carve files directly from the packet stream.

### Process

1. Opened PCAP in Wireshark
2. Navigated to **File → Export Objects → HTTP**
3. Reviewed the full HTTP object list — included HTML, CSS, JavaScript, and GIF files from Expedia, Google, OpenOffice, and ZoneLabs
4. Selected and saved `index.html` (Expedia) and `DomainHome.css` (OpenOffice) to disk

### Notable Extracted Objects

| Packet | Host | Type | Filename | Size |
|--------|------|------|----------|------|
| 9 | www.expedia.com | text/html | \ | 121 B |
| 61 | www.expedia.com | text/html | Default.asp | 32 KB |
| 405 | www.openoffice.org | text/css | print.css | 226 B |
| 519 | www.openoffice.org | image/gif | eggsopen_v6_with_gull.gif | 28 KB |
| 313 | pa2.zonelabs.com | application/octet-stream | (binary) | 553 B |

**Key finding:** The ZoneLabs octet-stream binary transfers are the most forensically interesting — in a real investigation these would be submitted to a sandbox for analysis.

---

## NetworkMiner vs. Wireshark – Comparison

| Feature | NetworkMiner | Wireshark |
|---------|-------------|-----------|
| **Ease of use** | High — automatic parsing into tabs | Moderate — requires filter knowledge |
| **Host overview** | Excellent — automatic host grouping | Manual filtering required |
| **Credential extraction** | Automatic | Manual — requires following streams |
| **File carving** | Automatic | Manual via Export Objects |
| **Packet-level detail** | Limited | Excellent |
| **Protocol analysis** | Summary only | Full dissection |
| **Best use case** | Quick triage and artifact overview | Deep-dive investigation |

**Summary:** NetworkMiner excels at rapid triage — giving analysts an instant overview of hosts, files, and credentials without manual filtering. Wireshark provides the deeper packet-level visibility needed once a suspicious artifact or connection is identified. In practice, both tools complement each other well in a forensic workflow.

---

## Skills Demonstrated

- Passive network forensic analysis (offline PCAP)
- Host enumeration and OS fingerprinting from network traffic
- HTTP credential and session token extraction
- DNS query history reconstruction
- File carving from packet captures (NetworkMiner automatic + Wireshark HTTP export)
- Forensic significance assessment of network artifacts

---

## References

- [NetworkMiner – Netresec](https://www.netresec.com/?page=NetworkMiner)
- [Wireshark Documentation](https://www.wireshark.org/docs/)
- [Public PCAP Sources – chrissanders/packets](https://github.com/chrissanders/packets)
