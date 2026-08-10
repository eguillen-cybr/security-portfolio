# Network Scanning & Mapping – Nmap, Hping3, Zenmap, SolarWinds
 
**Tools:** Nmap 7.95, Zenmap, Hping3, Netcat, Advanced IP Scanner, SolarWinds Network Topology Mapper  
**Skills Demonstrated:** Port scanning, OS/service fingerprinting, UDP scanning, vulnerability scanning, banner grabbing, network topology mapping, ping sweep techniques

---

## Overview

Performed comprehensive network reconnaissance against a multi-VM lab environment using industry-standard scanning tools. Mapped hosts, identified open ports and services, fingerprinted operating systems, grabbed service banners, and visualized network topology — simulating the reconnaissance phase of a penetration test.

---

## Lab Environment

| Host | IP Address | OS |
|------|-----------|-----|
| Kali Linux (attacker) | 10.0.5.7 | Kali Rolling |
| Metasploitable Linux | 10.0.5.6 | Ubuntu (Linux 3.x–4.x) |
| Metasploitable Win2K8 | 10.0.5.5 | Windows Server 2008 R2 |
| Windows 10 | 10.0.5.4 | Windows 10 |

Network: VirtualBox NAT Network — `MultimachineNAT` (10.0.5.0/24)

---

## Part 1 – Port Scanning with Nmap

### Default Scan – Metasploitable Linux

```bash
sudo nmap 10.0.5.6
```

**Results (selected open ports):**

| Port | State | Service |
|------|-------|---------|
| 22/tcp | open | ssh |
| 80/tcp | open | http |
| 445/tcp | open | microsoft-ds |
| 631/tcp | open | ipp |
| 3306/tcp | open | mysql |
| 8181/tcp | open | intermapper |

Host latency: 0.00023s — indicating same-subnet local target.

---

### Multi-Host Scan (Intense Scan)

```bash
nmap -T4 -A -v 10.0.5.5 10.0.5.4
```

Scanned both Metasploitable Win2K8 and Windows 10 simultaneously. Win2K8 (10.0.5.5) returned 30+ open ports including RDP (3389), HTTP (80/8080), MySQL (3306), and numerous unknown high ports in the 49000+ range.

---

### OS and Service Version Detection

```bash
sudo nmap -O -sV 10.0.5.6
```

**Metasploitable Linux findings:**

| Port | Service | Version |
|------|---------|---------|
| 22/tcp | ssh | OpenSSH 6.6.1p1 Ubuntu 2ubuntu2.10 |
| 80/tcp | http | Apache httpd 2.4.7 |
| 445/tcp | netbios-ssn | Samba smbd 3.X – 4.X |
| 631/tcp | ipp | CUPS 1.7 |
| 3306/tcp | mysql | MySQL (unauthorized) |
| 8181/tcp | http | WEBrick httpd 1.3.1 (Ruby 2.3.7) |

OS detection: Linux 3.2–4.14 (95% confidence), hostname `METASPLOITABLE3-UB1404`

**Metasploitable Win2K8 findings:**

OS: Microsoft Windows Server 2008 R2 / Windows 7 SP1 (confirmed via CPE)

---

### Targeted Port Scan

```bash
sudo nmap -p 22,80,443,139,445 10.0.5.6
sudo nmap -p 22,80,443,139,445 10.0.5.5
sudo nmap -p 22,80,443,139,445 10.0.5.4
```

**Key findings:** Windows 10 (10.0.5.4) had all five ports filtered — indicating active firewall. Win2K8 had SSH, HTTP, NetBIOS, and SMB all open. Linux target confirmed SSH and HTTP open, 139 and 443 filtered.

---

### Full Port TCP Connect Scan – Win2K8

```bash
sudo nmap -sT -p- 10.0.5.5
```

Discovered 35+ open ports on Win2K8, including non-standard services on high ports that a default scan would miss — demonstrating the value of full-range scans when thorough coverage is required.

---

### UDP Scan

```bash
sudo nmap -sU 10.0.5.6
```

UDP scan ran against Metasploitable Linux. All 1000 scanned UDP ports returned as open|filtered (no-response) — expected behavior since closed UDP ports silently drop packets rather than responding with ICMP unreachable messages, making UDP scans significantly slower than TCP scans.

---

### Custom Scans

**Vulnerability Script Scan:**
```bash
nmap -Pn --script vuln 10.0.5.5
```

Identified multiple vulnerabilities on Win2K8:
- **ssl-dh-params:** VULNERABLE — Diffie-Hellman Key Exchange Insufficient Group Strength (Logjam) on ports 3920 and 4848; Cipher Suite `TLS_DHE_RSA_WITH_AES_128_CBC_SHA`, 1024-bit modulus (RFC2409/Oakley Group 2)
- **ssl-dh-params (anonymous):** VULNERABLE — Anonymous DH Key Exchange MitM Vulnerability on port 8031; Cipher `TLS_DH_anon_WITH_AES_128_CBC_SHA`, 768-bit non-safe prime
- **http-slowloris:** Port 8080 — LIKELY VULNERABLE to Slowloris DoS (CVE-2007-6750)
- **http-litespeed-sourcecode-download:** CVE-2010-2333 detected on port 8080

**SYN Stealth Scan:**
```bash
nmap -sS 10.0.5.6
```

SYN scan sends SYN packets and records SYN-ACK responses without completing the TCP handshake — faster and less likely to appear in application-layer logs than a full connect scan. Confirmed same open ports as the default scan on Metasploitable Linux.

---

## Part 2 – Hping3 Scanning

### Full Port Scan

```bash
sudo hping3 -8 1-65535 -S 10.0.5.6
```

Hping3 scanned all 65,535 ports against Metasploitable Linux. Confirmed open ports: 22 (SSH), 80 (HTTP), 445 (microsoft-ds), 631 (IPP), 3306 (MySQL), 3500, 6697, 8181.

---

## Part 3 – Ping Sweep

### Nmap Ping Sweep (Disable Port Scan)

```bash
sudo nmap -sn 10.0.5.0/24
```

Discovered 7 live hosts on the /24 subnet (10.0.5.1 through 10.0.5.7) in 1.99 seconds. MAC addresses confirmed VirtualBox virtual NICs for lab VMs and QEMU virtual NICs for gateway/infrastructure hosts.

### Nmap TCP SYN Discovery

```bash
sudo nmap -PS22 10.0.5.0/24
```

Same 7 hosts discovered via TCP SYN probe on port 22 rather than ICMP — useful in environments where ping is blocked at the firewall but TCP connections are allowed.

### Hping3 ICMP Sweep

```bash
sudo hping3 -1 --rand-dest 10.0.5.x -I eth0
```

ICMP ping sweep against the lab range confirmed host 10.0.5.1 responding with TTL=255.

---

## Part 4 – Banner Grabbing

### Nmap Banner Script

```bash
sudo nmap --script=banner 10.0.5.5
```

**Results from Win2K8:**

| Port | Banner |
|------|--------|
| 21/tcp | `220 Microsoft FTP Service` |
| 22/tcp | `SSH-2.0-OpenSSH_7.1` |
| 3306/tcp | MySQL blocked connection error (host blocked, flush-hosts required) |
| 7676/tcp | `101 imqbroker 301 portmapper tcp PORTMAPPER 7676` (GlassFish) |

### Manual Banner Grabbing with Netcat

```bash
nc 10.0.5.5 21   # Returns: 220 Microsoft FTP Service
nc 10.0.5.5 22   # Returns: SSH-2.0-OpenSSH_7.1
nc 10.0.5.5 80   # No response (HTTP requires a request to be sent)
```

Port 80 required sending an HTTP GET request to receive a banner — Nmap's version detection handles this automatically but raw netcat connections to HTTP servers need a request issued first.

Nmap and netcat results matched for FTP (port 21) and SSH (port 22). Port 80 confirmed the difference between tools requiring manual interaction vs. automated script-driven probing.

---

## Part 5 – Windows-Side Scanning

### Advanced IP Scanner

Scanned both Metasploitable targets from the Windows 10 VM using Advanced IP Scanner:

| Host | IP | Manufacturer | Services Detected |
|------|-----|-------------|------------------|
| metasploitable3-win2k8 | 10.0.5.5 | PCS Systemtechnik GmbH | HTTP (IIS 7.5), FTP (Microsoft ftpd) |
| 10.0.5.6 | 10.0.5.6 | PCS Systemtechnik GmbH | HTTP (Apache 2.4.7) |

**Advanced IP Scanner vs. Nmap:** Advanced IP Scanner provides a quick, user-friendly host inventory with basic service detection — useful for rapid asset identification. Nmap is significantly more powerful, offering deeper OS fingerprinting, version detection, vulnerability scripting, and protocol coverage that Advanced IP Scanner cannot match.

---

## Part 6 – Network Topology Mapping

### SolarWinds Network Topology Mapper

Scanned the IP range 10.0.5.0–10.0.5.15 using SolarWinds NTM from the Windows 10 VM. Result: **Star topology** with the NAT gateway as the central node, lab VMs visible as leaf nodes (metasploitable3-win2k8, DESKTOP-DKBA1GC, 10.0.5.7).

**Penetration test application:** Star topology identification reveals a single point of failure — targeting the central switch/router could cause a complete network outage. Node details (IP, MAC, polling method) provide a device inventory for subsequent exploitation planning.

### Zenmap Topology View

Scanned the same range in Zenmap and viewed the Fisheye topology:

- **Green nodes** (10.0.5.7, 10.0.5.2): Few or no open ports — lower attack surface
- **Red node** (10.0.5.5 – Win2K8): Multiple open services (FTP, SSH, HTTP, NetBIOS, etc.) — highest vulnerability exposure
- **Yellow nodes**: Moderate port exposure

**SolarWinds vs. Zenmap:** SolarWinds is better suited for enterprise network monitoring, performance visibility, and physical/logical topology documentation. Zenmap provides deeper per-host intelligence (open ports, service versions, vulnerability indicators) that is more directly actionable during a penetration test.

---

## Tool Summary for Network Administrators

**Nmap** is essential for network discovery and asset inventory — identifying every device, open port, service version, and OS on the network. Its scripting engine enables automated vulnerability checks against known CVEs, making it useful for both offensive reconnaissance and defensive hardening audits.

**Hping3** enables packet-level testing: crafting custom TCP/IP packets to evaluate firewall rule behavior, simulate specific attack traffic patterns, and test how devices respond to malformed or unexpected packets — valuable for validating firewall configurations.

**Netcat** functions as a versatile network Swiss Army knife — banner grabbing, port testing, creating listeners for reverse shells in authorized penetration tests, and building simple TCP/UDP tunnels for encrypted communication testing.

---

## Skills Demonstrated

- TCP/UDP port scanning across multiple scan types (SYN stealth, connect, version, OS detection, UDP)
- Service version fingerprinting and OS identification
- Vulnerability scanning with Nmap NSE (`--script vuln`)
- Banner grabbing (automated with Nmap, manual with Netcat)
- Ping sweep and host discovery (ICMP, TCP SYN, Hping3)
- Full-range port scanning (all 65,535 ports)
- Network topology mapping and visualization (SolarWinds, Zenmap)
- Cross-platform scanning (Kali Linux + Windows 10)
- Reconnaissance methodology applicable to penetration testing engagements

---

## References

- [Nmap Documentation](https://nmap.org/docs.html)
- [Hping3 Manual](http://www.hping.org/manpage.html)
- [Advanced IP Scanner](https://www.advanced-ip-scanner.com/)
- [SolarWinds Network Topology Mapper](https://www.solarwinds.com/network-topology-mapper)
