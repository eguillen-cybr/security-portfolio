# Honeypot Deployment & Analysis — Artillery (Linux) and HoneyBot (Windows)

## Overview

This lab covers the deployment, configuration, and analysis of two honeypot systems: Artillery on AntiX Linux (a low-interaction Linux honeypot with host-based IDPS capabilities) and HoneyBot on Windows (a configurable port simulation honeypot). The lab examines how honeypots work from both the defender's and attacker's perspectives — establishing a baseline of real services with `netstat`, deploying honeypot software to expand the apparent attack surface with simulated services, validating the deception with Nmap and Zenmap scans, and configuring a realistic port profile to make a Windows machine appear to be a Linux server. The bonus section covers Artillery's SSH brute-force blocking capability as an active IPS control, verified through auth log analysis and iptables chain inspection.

---

## Environment

| Component | Role | IP Address |
|---|---|---|
| AntiX Linux (VirtualBox VM) | Honeypot host — Artillery deployment | `10.0.5.13` |
| Windows 10 (VirtualBox VM) | Honeypot host — HoneyBot deployment | `10.0.5.9` |
| Kali Linux / AntiX (scanner) | External attacker simulation | `10.0.5.x` |
| Zenmap 7.92 | Nmap GUI frontend (Windows) | — |
| Artillery (BinaryDefense) | Linux honeypot / host-based IDPS | Installed from GitHub |
| HoneyBot 0.18 | Windows port simulation honeypot | Atomic Software Solutions |

---

## Background: Honeypot Taxonomy

**Research vs. Production honeypots:**
- Both research and production honeypots detect suspicious activity and provide visibility into attacker techniques and methods.
- Research honeypots gather broad intelligence about attacker behavior for the wider security community, without protecting a specific production system. Production honeypots are deployed within a real environment to detect intrusions targeting actual assets, with alerting tuned for immediate response.

**Interaction levels:**
- **High-interaction honeypots** emulate full, functional systems — real operating systems with real services. They provide rich intelligence about attacker TTPs but carry risk if the attacker pivots from the honeypot into the real network.
- **Low-interaction honeypots** simulate only the network surface of services (open ports that respond minimally) without running real service code. They generate less intelligence but are safer and easier to deploy at scale.

**Production network risks:** Honeypots can generate false positives if legitimate scanners or automated tools trigger alerts. If a high-interaction honeypot is compromised, it can become a pivot point for attacks on adjacent real systems — careful network segmentation is essential.

---

## Walkthrough

### Part 1 — Artillery on AntiX Linux

#### Baseline: Real Services Before Artillery

Before deploying Artillery, the real service footprint of the AntiX VM was documented using `netstat`:

```bash
student@antix1:~$ netstat -an|more
Active Internet connections (servers and established)
Proto Recv-Q Send-Q Local Address     Foreign Address   State
tcp        0      0 0.0.0.0:43117     0.0.0.0:*         LISTEN
tcp        0      0 0.0.0.0:111       0.0.0.0:*         LISTEN
tcp        0      0 127.0.0.1:53      0.0.0.0:*         LISTEN
tcp        0      0 0.0.0.0:22        0.0.0.0:*         LISTEN
tcp        0      0 127.0.0.1:631     0.0.0.0:*         LISTEN
```

Real services listening externally: TCP ports 43117 (ephemeral), 111 (rpcbind), 22 (SSH). Ports bound to `127.0.0.1` (localhost) are not externally accessible and were excluded.

#### Artillery Installation and Configuration

Artillery was cloned from GitHub and installed:

```bash
git clone https://github.com/binarydefense/artillery artillery
cd artillery
sudo -s
./setup.py
# Answered: yes (install), yes (keep updated), no (start now)
```

Artillery's configuration was edited to remove SSH from its simulated port list — since SSH is a real service on this machine, Artillery must not also open a fake SSH listener, which would create a conflict:

```bash
nano /var/artillery/config
# Removed "22," from the TCPPORTS line
service artillery start
```

#### Post-Artillery Port Comparison

After Artillery started, `netstat` showed a dramatically expanded set of listening ports:

```
New TCP ports: 16993, 5060, 5061, 5800, 5900, 110, 10000, 8080, 21, 1337, 1433, 25, 44443, 1723
New UDP ports: 5060, 5061, 68, 3478
```

None of these correspond to real services — they are Artillery's simulated honeypot ports. Two notable simulated services:
- **Port 21 — FTP:** Any connection attempt to this port will be logged by Artillery as a potential intrusion indicator
- **Port 25 — SMTP:** Simulates a mail server, attracting spam relay probes and email-based attack reconnaissance

From an attacker's perspective, distinguishing real services from honeypot ports requires attempting to interact with each service and verifying that responses match expected protocol behavior. A real FTP service will respond with a proper `220` banner and proceed through the FTP handshake; a low-interaction honeypot may return generic or unexpected responses, or simply log the connection without responding.

#### Zenmap Scan Results (from Windows VM)

A Zenmap Quick Scan (`nmap -T4 -F`) from the Windows VM confirmed Artillery's expanded port surface:

```
PORT      STATE  SERVICE
21/tcp    open   ftp
22/tcp    open   ssh
25/tcp    open   smtp
110/tcp   open   pop3
111/tcp   open   rpcbind
1433/tcp  open   ms-sql-s
1723/tcp  open   pptp
5060/tcp  open   sip
5800/tcp  open   vnc-http
5900/tcp  open   vnc
8080/tcp  open   http-proxy
10000/tcp open   snet-sensor-mgmt
```

Port 10000 appeared in the Nmap scan but not in the `netstat` output — demonstrating that network-level scanning can detect ports that process-level tools don't enumerate consistently, depending on timing and socket state.

---

### Part 2 — HoneyBot on Windows

#### Baseline: Real Windows Services Before HoneyBot

```
Real TCP ports listening externally: 135, 445, 5040, 5357, 7680, 139
Real UDP ports: 123, 500, 3702, 4500, 5050, 5353, 5355, 137, 138, 1900
```

#### HoneyBot Post-Deployment Nmap Scan (from AntiX Linux)

A full TCP port scan from AntiX Linux revealed HoneyBot's default configuration opens hundreds of simulated ports, including virtually every well-known port:

```bash
$ nmap -Pn -sT -p 1-65535 -T4 10.0.5.9
```

HoneyBot's default configuration is unrealistically expansive — hundreds of sequentially numbered ports appearing open simultaneously is an immediate red flag to any experienced attacker. No real server exposes this many services. This defeats the deception purpose.

Two simulated services in the default configuration:
- **Port 23 — Telnet:** Simulates a legacy remote access service, attracting attackers who probe for unencrypted remote management
- **Port 21 — FTP:** Attracts anonymous FTP probes and credential stuffing attempts

---

### Part 3 — Realistic Port Profiling (Making Windows Look Like Linux)

The default HoneyBot configuration is detectable. The goal of this section was to configure HoneyBot to convincingly impersonate a Linux server running a specific set of services.

#### Target Port Profile (Linux Server Simulation)

The `C:\Honeybot\service` file was edited directly to define a realistic Linux server profile:

```
HoneyBOT
 14
22,0,1,SSH
25,0,1,SMTP
80,0,1,HTTP
111,0,1,NFS
143,0,1,IMAP
443,0,1,HTTPS
631,0,1,CUPS
2049,0,1,NFS
```

This profile simulates: SSH (22), SMTP (25), HTTP (80), NFS portmapper (111), IMAP (143), HTTPS (443), CUPS printing (631), and NFS mountd (2049). All services are TCP only — none of the target services use UDP for their primary function.

Windows Firewall rules were created to block real Windows services from leaking through (135 msrpc, 139 netbios-ssn, 445 smb, 5040, 5357, 7680):

The HoneyBot Services panel was configured to match exactly the 8 ports above.

#### Nmap Validation Scan

```bash
sudo nmap -sS -p 1-65535 10.0.5.9
```

Results showed the intended ports (22, 25, 80, 111, 143, 443, 631, 2049) open, plus residual Windows ports (135 msrpc, 139 netbios-ssn, 5357 wsdapi, 49664-49669 unknown) that Windows firewall rules did not fully suppress.

#### Deeper Nmap Service Fingerprinting

An aggressive scan against the target ports revealed the seams in the deception:

```bash
sudo nmap -sS -p 22,25,80,111,143,631,2049 -A -T4 10.0.5.9
```

Key output:
- Port 80: identified as **Microsoft IIS httpd 5.0** — a Windows web server signature, not Linux Apache
- Port 25 SMTP: responded with `220 PUBLIC08 SMTP Ready` banner — not a standard Linux MTA banner
- Port 143 IMAP: responded with `OK Interim Mail Access Protocol II Service` — a generic IMAP response but with non-standard formatting
- OS detection: identified the system as **iPXE/Linux 2.4/2.6.X or Sony Ericsson embedded**, with `Service Info: OS: Windows`

**Attacker detection indicators:**
1. **OS fingerprint mismatch** — Nmap's OS detection returned Windows CPE despite the Linux-like port profile
2. **Service banner inconsistency** — Port 80 reported Microsoft IIS, which no Linux server would be running
3. **Protocol anomalies** — Ports like NFS (2049) and CUPS (631) returned unrecognized service fingerprints, indicating they don't behave like real implementations of those protocols
4. **Residual Windows ports** — Ports 135, 139, and high-numbered ephemeral Windows ports visible alongside the Linux service profile are immediately suspicious

A skilled attacker who interacts with any of these services beyond a port scan will identify the deception quickly. A more convincing honeypot requires either a real Linux installation or a high-interaction platform that serves genuine protocol responses.

---

### Bonus — Artillery SSH Brute-Force Blocking (IPS Mode)

Artillery was configured as an active IPS to automatically block IP addresses that attempt SSH brute-force attacks. After triggering repeated failed SSH logins from `10.0.5.9`, the auth log confirmed the failed attempts:

```
Jun  7 21:14:08 antix1 sudo: student : TTY=pts/1 ; PWD=/home/student ; USER=root ; COMMAND=/usr/bin/tail -f /var/log/auth.log
Jun  7 21:14:14 antix1 sshd[23273]: Failed password for student from 10.0.5.9 port 50163 ssh2
Jun  7 21:14:15 antix1 sshd[23273]: Failed password for student from 10.0.5.9 port 50163 ssh2
Jun  7 21:14:16 antix1 sshd[23273]: Failed password for student from 10.0.5.9 port 50163 ssh2
Jun  7 21:14:16 antix1 sshd[23273]: Connection reset by authenticating user student 10.0.5.9 port 50163 [preauth]
```

Artillery's iptables chain automatically added a DROP rule for the offending IP:

```bash
$ sudo iptables -L -n
Chain ARTILLERY (0 references)
target  prot opt source      destination
DROP    all  --  10.0.5.13   0.0.0.0/0
```

The `ARTILLERY` chain in iptables shows a `DROP all` rule for the source IP that triggered the brute-force threshold. All subsequent traffic from that address is silently dropped at the kernel level, without the attacker receiving an explicit rejection. This prevents the attacker from knowing whether the block was applied or the host simply became unreachable.

---

## Skills Demonstrated

- **Honeypot taxonomy** — Differentiated research vs. production honeypots and high- vs. low-interaction honeypots; articulated deployment risks in production networks
- **Service baseline documentation** — Used `netstat -an` to establish a pre-deployment service inventory and verified the delta after honeypot installation
- **Artillery deployment and configuration** — Cloned, installed, and configured Artillery on Linux; removed real services from the honeypot port list to avoid conflicts; started and verified the service
- **HoneyBot deployment and configuration** — Installed and configured HoneyBot on Windows; edited the `service` file directly to define a custom port profile
- **Realistic port profiling** — Designed a convincing Linux server port profile (SSH, SMTP, HTTP, NFS, IMAP, HTTPS, CUPS) and applied it to a Windows honeypot; used Windows Firewall rules to suppress real Windows service ports
- **Nmap scan validation** — Used both quick scan and full TCP scan to verify honeypot port surface; used aggressive service fingerprinting (`-A`) to identify deception artifacts
- **Honeypot deception analysis** — Identified specific nmap output indicators (OS fingerprint, service banners, protocol anomalies, residual Windows ports) that expose the honeypot's true identity to an attacker
- **Artillery IPS mode** — Configured Artillery for active SSH brute-force blocking; verified iptables `ARTILLERY` chain DROP rules and correlated with `/var/log/auth.log` failed login entries

---

## References

- [Artillery by BinaryDefense](https://github.com/binarydefense/artillery)
- [The Honeynet Project](https://www.honeynet.org/)
- [NIST SP 800-150 — Guide to Cyber Threat Information Sharing](https://csrc.nist.gov/publications/detail/sp/800-150/final)
- [Nmap Reference Guide](https://nmap.org/book/man.html)
- [MITRE ATT&CK — Honeypot Detection Techniques](https://attack.mitre.org/)
