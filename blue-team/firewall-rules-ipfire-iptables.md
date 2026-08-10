# Firewall Rule Design — IPFire Perimeter Firewall & Linux Host-Based Firewall (iptables)

## Overview

This lab demonstrates end-to-end firewall policy implementation across two layers of a defense-in-depth architecture: a perimeter firewall (IPFire) and a host-based firewall (iptables via GUFW) on an Ubuntu server. Starting from a fully locked-down baseline — all traffic blocked in both directions — firewall rules were designed, implemented, and validated for a small network serving a web server and SSH server to external clients.

Key activities included configuring destination NAT (port forwarding) rules on IPFire to expose internal services through a WAN interface, building egress filtering rules to restrict outbound traffic to approved protocols only, hardening the Ubuntu host with a default-deny iptables policy with explicit allow exceptions, and validating the entire ruleset using Nmap, live connection testing, and IPFire log analysis. A bonus exercise extended the ruleset to support a new service (vsftpd FTP), demonstrating the repeatable process of opening a new protocol through both firewall layers.

---

## Environment

| Component | Role | IP Address |
|---|---|---|
| IPFire 2.27 (VirtualBox VM) | Perimeter firewall / NAT router / DNS / DHCP | WAN (red0): `10.0.5.9` / LAN (green0): `192.168.5.1` |
| Ubuntu 20.04 LTS (VirtualBox VM) | Internal web + SSH server | `192.168.5.10` |
| Bodhi Linux 6.0 (VirtualBox VM) | External client (simulates Internet) | `10.0.5.5` |
| lighttpd | Web server running on Ubuntu | Port 80 |
| OpenSSH | SSH server running on Ubuntu | Port 22 |
| vsftpd 3.0.5 | FTP server installed in Part 4 | Port 21 |
| GUFW | GUI frontend for iptables on Ubuntu | — |

**Network topology:** Bodhi Linux (outside/WAN) → IPFire (NAT gateway) → Ubuntu (inside/LAN). All client traffic from Bodhi must traverse IPFire's WAN interface (`10.0.5.9`) to reach internal services via port forwarding.

---

## Walkthrough

### Part 1 — IPFire Perimeter Firewall Configuration

#### Baseline: All Traffic Blocked

IPFire shipped in a fully restrictive state — all ingress and egress traffic blocked by default policy. The first verification step confirmed that the Ubuntu web server was unreachable from the outside world before any rules were created.

From Bodhi Linux (external client), browsing to the IPFire WAN IP returned a connection timeout:

```
http://10.0.5.9
→ ERR_CONNECTION_TIMED_OUT
```

This confirmed the firewall's default-deny posture was functioning correctly.

#### Ingress Rules — WAN to LAN (Port Forwarding / Destination NAT)

IPFire was configured through its web GUI (`https://192.168.5.1:444`) to forward inbound traffic on specific ports from the WAN interface to the Ubuntu internal server. Two NAT port-forward rules were created, plus a deny-all non-NAT rule as the final entry:

| Rule # | Protocol | Source | Destination | Description |
|---|---|---|---|---|
| 1 | TCP | Any | Firewall → `192.168.5.10` (port 80) | Port Forward for HTTP |
| 2 | TCP | Any | `192.168.5.10:22` | Port Forward for SSH |
| 3 | All | Any (RED) | Any (GREEN) | Deny all from Red to Green |

The deny-all rule in position 3 ensures that any traffic not explicitly permitted by rules 1 or 2 is blocked, enforcing a default-deny posture at the perimeter.

#### Egress Rules — LAN to WAN (Outbound Filtering)

Five outbound rules were added to allow the Ubuntu server and internal network to reach approved Internet services, with all other outbound traffic blocked by the existing default policy:

| Rule # | Protocol | Source | Destination | Port | Description |
|---|---|---|---|---|---|
| 4 | TCP | GREEN | Any (RED) | 443 | Allow HTTPS traffic |
| 5 | TCP | GREEN | Any (RED) | 80 | Allow HTTP traffic |
| 6 | UDP | GREEN | Any (RED) | 53 | Allow DNS traffic |
| 7 | ICMP (type 8) | GREEN | Any (RED) | — | Allow ICMP echo request |
| 8 | ICMP (type 0) | GREEN | Any (RED) | — | Allow ICMP echo reply |

Note: DNS uses UDP/53 for standard queries. ICMP was restricted to echo request (type 8) and echo reply (type 0) only — all other ICMP types remained blocked, limiting exposure to ICMP-based reconnaissance and attacks.

#### Validation — HTTP Access Through NAT

From Bodhi Linux, browsing to the IPFire WAN IP (`http://10.0.5.9`) successfully forwarded through to the Ubuntu web server and returned the customized page:

```
Browser: http://10.0.5.9
→ "Emmanuel Guillen page" (lighttpd on 192.168.5.10:80)
```

#### Validation — SSH Access Through NAT

From Bodhi Linux terminal, SSH through the WAN IP forwarded to the Ubuntu server via port forwarding rule 2:

```bash
student@Bodhi6:~$ ssh student@10.0.5.9 -p 22
The authenticity of host '10.0.5.9 (10.0.5.9)' can't be established.
ECDSA key fingerprint is SHA256:W4iRmGaXBgfoICFXBuCX93gjyRhxCR2bb/WLarDANNk.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.0.5.9' (ECDSA) to the list of known hosts.
student@10.0.5.9's password:
Welcome to Ubuntu 20.04.3 LTS (GNU/Linux 5.11.0-37-generic x86_64)
...
student@student-vm:~$
```

SSH successfully traversed NAT from the external Bodhi client to the internal Ubuntu server — confirming the port forward rule was operating correctly.

#### Validation — Outbound Internet Connectivity from Ubuntu

From the Ubuntu server, outbound browsing and ICMP were tested against the egress rules:

```bash
student@student-vm:~$ ping google.com
PING google.com (142.250.72.46) 56(84) bytes of data.
64 bytes from den16s08-in-f14.1e100.net (142.250.72.46): icmp_seq=1 ttl=58 time=54.7 ms
64 bytes from den16s08-in-f14.1e100.net (142.250.72.46): icmp_seq=2 ttl=58 time=50.1 ms
64 bytes from den16s08-in-f14.1e100.net (142.250.72.46): icmp_seq=3 ttl=58 time=47.5 ms
64 bytes from den16s08-in-f14.1e100.net (142.250.72.46): icmp_seq=4 ttl=58 time=46.6 ms
64 bytes from den16s08-in-f14.1e100.net (142.250.72.46): icmp_seq=5 ttl=58 time=47.9 ms
--- google.com ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 4004ms
rtt min/avg/max/mdev = 46.589/49.352/54.714/2.914 ms
```

Firefox on Ubuntu also successfully reached `https://www.google.com`, confirming both HTTP/HTTPS egress and DNS resolution were working.

---

### Part 2 — Firewall Validation and Log Analysis

#### Nmap Ruleset Validation

With all IPFire rules in place, an Nmap scan was run from Bodhi Linux against the WAN IP to verify that only the intended ports were reachable. Ping discovery was disabled (`-Pn` equivalent via the scan flags used) to prevent ICMP from interfering with port results:

```bash
student@Bodhi6:~$ nmap -p 1-30000 10.0.5.9
Starting Nmap 7.80 ( https://nmap.org ) at 2024-06-01 01:11 MDT
Nmap scan report for 10.0.5.9
Host is up (0.00051s latency).
Not shown: 29998 closed ports
PORT    STATE  SERVICE
22/tcp  open   ssh
80/tcp  open   http

Nmap done: 1 IP address (1 host up) scanned in 6.68 seconds
```

Exactly two ports open — SSH (22) and HTTP (80) — confirming the IPFire ruleset was enforcing the policy correctly. All 29,998 other ports were closed/blocked.

#### IPFire Firewall Log Analysis

IPFire's web interface (`Logs > Firewall Logs`) was used to review logged traffic. The most recent entries showed a total of 575 firewall hits for the day.

The log revealed `DROP_FORWARD` entries on the `green0` interface — outbound TCP packets being blocked because they did not match an allow rule. These originated from `192.168.5.10` (Ubuntu) destined for external IPs on ports 80 (HTTP) and 443 (HTTPS), with rapidly incrementing source ports (40584, 40582, 40586...).

**What indicates this is a port scan in the logs:**
The logs show sequential source port numbers incrementing by small values across repeated connection attempts to the same destination IP and port within a 1-second window. In a normal browsing session, a handful of connections are made; a port scan generates dozens of DROP entries in rapid succession from one source, targeting the same destination ports. The pattern of `DROP_FORWARD` events at timestamps 00:08:13 through 00:08:28 — 30+ blocked packets in 15 seconds — is a clear scan signature.

**How this data could feed IDPS rules:**
Firewall logs can be ingested by a SIEM or IDPS (such as Snort or Suricata) to build threshold-based detection rules. For example: if a single source IP generates more than 20 `DROP_FORWARD` events targeting the same destination port within a 60-second window, trigger a port scan alert. The source IP, destination port, timestamp, and protocol fields in the IPFire logs provide exactly the fields needed for such correlation rules. Rate-limiting rules could also be added to IPFire itself to automatically block IPs that exceed a connection attempt threshold.

---

### Part 3 — Ubuntu Host-Based Firewall (iptables via GUFW)

#### Default-Deny Policy

Even after IPFire allowed HTTP traffic inbound, a host-based firewall on Ubuntu provides a second layer of defense. GUFW was installed and configured with a default-deny policy on both ingress and egress:

```bash
sudo apt install gufw
```

After setting both Incoming and Outgoing to **Deny** and enabling the firewall, the Ubuntu server was immediately unreachable from Bodhi Linux — even though IPFire's port forward rule was still active:

```
Browser: http://10.0.5.9
→ ERR_CONNECTION_REFUSED
```

This demonstrated the defense-in-depth principle: the perimeter firewall (IPFire) allowed the packet through, but the host firewall (iptables) dropped it at the destination. Both layers must permit traffic for a connection to succeed.

#### Explicit Allow Rules

Five rules were added in GUFW to permit only the required traffic, with IPv6 rules removed for clarity:

| Rule # | Direction | Protocol | Port | Name |
|---|---|---|---|---|
| 1 | IN | TCP | 80 | Allow HTTP IN |
| 2 | IN | TCP | 22 | Allow SSH IN |
| 3 | OUT | TCP | 80 | Allow HTTP Out |
| 4 | OUT | TCP | 443 | Allow HTTPs Out |
| 5 | OUT | TCP | 53 | Allow DNS Out |

#### iptables Chain Verification

GUFW writes rules directly to iptables. The following commands verified the policy was correctly implemented:

```bash
student@student-vm:~$ sudo iptables -L INPUT
Chain INPUT (policy DROP)
target                    prot opt source    destination
ufw-before-logging-input  all  --  anywhere  anywhere
ufw-before-input          all  --  anywhere  anywhere
ufw-after-input           all  --  anywhere  anywhere
ufw-after-logging-input   all  --  anywhere  anywhere
ufw-reject-input          all  --  anywhere  anywhere
ufw-track-input           all  --  anywhere  anywhere
ACCEPT                    tcp  --  anywhere  anywhere  tcp dpt:http
ACCEPT                    tcp  --  anywhere  anywhere  tcp dpt:ssh
```

```bash
student@student-vm:~$ sudo iptables -L OUTPUT
Chain OUTPUT (policy DROP)
target                     prot opt source    destination
ufw-before-logging-output  all  --  anywhere  anywhere
ufw-before-output          all  --  anywhere  anywhere
ufw-after-output           all  --  anywhere  anywhere
ufw-after-logging-output   all  --  anywhere  anywhere
ufw-reject-output          all  --  anywhere  anywhere
ufw-track-output           all  --  anywhere  anywhere
ACCEPT                     tcp  --  anywhere  anywhere  tcp dpt:http
ACCEPT                     tcp  --  anywhere  anywhere  tcp dpt:https
ACCEPT                     udp  --  anywhere  anywhere  udp dpt:domain
```

```bash
student@student-vm:~$ sudo iptables -L ufw-user-input
Chain ufw-user-input (1 references)
target  prot opt source    destination
ACCEPT  tcp  --  anywhere  anywhere  tcp dpt:http
ACCEPT  tcp  --  anywhere  anywhere  tcp dpt:ssh
```

```bash
student@student-vm:~$ sudo iptables -L ufw-user-output
Chain ufw-user-output (1 references)
target  prot opt source    destination
ACCEPT  tcp  --  anywhere  anywhere  tcp dpt:http
ACCEPT  tcp  --  anywhere  anywhere  tcp dpt:https
ACCEPT  tcp  --  anywhere  anywhere  tcp dpt:domain
ACCEPT  udp  --  anywhere  anywhere  udp dpt:domain
```

Both INPUT and OUTPUT chains show `policy DROP` as the default — all traffic is dropped unless matched by an explicit ACCEPT rule. The `ufw-user-input` and `ufw-user-output` chains contain only the four specific exceptions needed, overriding the default drop for those ports only.

With these rules active, Bodhi Linux was again able to browse to the Ubuntu web server through IPFire (`http://10.0.5.9`), and Ubuntu was able to reach the Internet — both operating within the constraints of the configured policy.

---

### Part 4 — Extending the Ruleset: vsftpd FTP Server

To demonstrate the repeatable process of exposing a new service through both firewall layers, vsftpd was installed on the Ubuntu server and the ruleset was extended accordingly.

#### Installation and Configuration

```bash
student@student-vm:~$ sudo apt install vsftpd
Reading package lists... Done
Building dependency tree
Reading state information... Done
The following NEW packages will be installed:
  vsftpd
0 upgraded, 1 newly installed, 0 to remove and 473 not upgraded.
...
Setting up vsftpd (3.0.5-0ubuntu0.20.04.1) ...
Created symlink /etc/systemd/system/multi-user.target.wants/vsftpd.service → /lib/systemd/system/vsftpd.service.

student@student-vm:~$ sudo nano /etc/vsftpd.conf
student@student-vm:~$ sudo systemctl restart vsftpd
```

#### Firewall Rule Updates

**GUFW (host-based firewall):** A new ingress rule was added for FTP:

```
Rule 7: 21/tcp ALLOW IN Anywhere    [Allow FTP]
```

**IPFire (perimeter firewall):** A new port-forward NAT rule was added:

```
Rule 9: TCP | Any → Firewall:21 → 192.168.5.10:21 | Allow FTP
```

#### FTP Connection Validation from Bodhi Linux

```bash
student@Bodhi6:~$ ftp 10.0.5.9
Connected to 10.0.5.9.
220 (vsFTPd 3.0.5)
Name (10.0.5.9:student): student
331 Please specify the password.
Password:
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp>
```

FTP connected successfully through IPFire's WAN IP, traversed the NAT port forward, and authenticated to vsftpd on the Ubuntu server.

#### vsftpd Server Log Verification

```bash
student@student-vm:~$ sudo cat /var/log/vsftpd.log
Sat Jun  1 21:08:56 2024 [pid 42850] CONNECT: Client "::ffff:192.168.5.1"
Sat Jun  1 21:13:22 2024 [pid 42849] [student] OK LOGIN: Client "::ffff:192.168.5.1"
```

The vsftpd log records the connection originating from `192.168.5.1` — the IPFire LAN gateway — because NAT rewrites the source IP as packets traverse the firewall. This is expected behavior: the server sees the gateway's LAN IP rather than the original `10.0.5.5` Bodhi Linux WAN address. This log entry confirms the connection was successfully established and authenticated.

---

## Skills Demonstrated

- **Perimeter firewall design** — Built a production-style IPFire ruleset from scratch: destination NAT for port forwarding, egress filtering by protocol/port, and explicit deny-all as final rule
- **Defense-in-depth** — Implemented independent firewall layers (IPFire + iptables) and demonstrated that both must permit traffic for a connection to succeed
- **iptables / UFW** — Configured default-deny INPUT and OUTPUT chains with explicit ACCEPT rules; verified chain contents with `iptables -L` commands
- **NAT/PAT concepts** — Understood and applied destination NAT (reverse NAT / port forwarding), stateful packet inspection behavior, and source IP rewriting through NAT
- **Firewall validation with Nmap** — Used Nmap port scanning to verify only intended services were exposed, treating the ruleset as a testable security control
- **Firewall log analysis** — Analyzed IPFire DROP_FORWARD log entries to identify port scan signatures (rapid sequential drops from single source) and articulated how log data maps to IDPS threshold rules
- **Service exposure workflow** — Applied a repeatable process (install service → update host firewall → update perimeter firewall → validate connectivity → verify logs) to onboard a new protocol (FTP/vsftpd)
- **Protocol-aware rule writing** — Correctly applied TCP vs. UDP for HTTP, HTTPS, SSH, DNS, and FTP; restricted ICMP to only echo request/reply types

---

## References

- [IPFire Port Forwarding Documentation](https://wiki.ipfire.org/configuration/firewall/rules/port-forwarding)
- [IPFire Firewall Rules Documentation](https://wiki.ipfire.org/configuration/firewall/rules/start)
- [Ubuntu UFW Documentation](https://help.ubuntu.com/community/UFW)
- [vsftpd Documentation](https://security.appspot.com/vsftpd.html)
- [Nmap Reference Guide](https://nmap.org/book/man.html)
- [NIST SP 800-41 Rev. 1 — Guidelines on Firewalls and Firewall Policy](https://csrc.nist.gov/publications/detail/sp/800-41/rev-1/final)
