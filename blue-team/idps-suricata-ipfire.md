# Intrusion Detection & Prevention — Suricata on IPFire

## Overview

This lab demonstrates end-to-end configuration of a network-based IDPS using Suricata integrated into IPFire, the same perimeter firewall used in prior labs. Starting from a default-blocked posture, Suricata was enabled in monitor-only mode on the WAN interface, then progressively expanded to cover LAN-side egress traffic. The lab exercises three distinct detection scenarios: inbound port scan detection using `emerging-scan.rules`, outbound DNS query monitoring for suspicious TLDs, and outbound traffic matching against community threat intelligence lists (`compromised.rules`, `3coresec.rules`). The final section covers IP reputation analysis, false positive evaluation methodology, and how firewall log data maps to IDPS tuning decisions.

---

## Environment

| Component | Role | IP Address |
|---|---|---|
| IPFire 2.27 (VirtualBox VM) | Perimeter firewall / Suricata IDPS engine | WAN (red0): `10.0.5.9` / LAN (green0): `192.168.5.1` |
| Ubuntu 20.04 LTS (VirtualBox VM) | Internal client / web server | `192.168.5.10` / `10.0.5.11` |
| Bodhi Linux 6.0 (VirtualBox VM) | External attacker simulation | `10.0.5.5` |
| Suricata (via IPFire IPS module) | Network-based IDPS engine | Integrated into IPFire |
| Emergingthreats.net Community Rules | Ruleset source | Updated 2024-06-07 21:37:08 |

**Note:** For Part 1 inbound IDS testing, IPFire's default forward policy was temporarily changed from "Blocked" to "Allowed" to permit scan traffic to reach the internal network.

---

## Walkthrough

### Part 1 — Inbound IDS (WAN Interface Monitoring)

#### Enabling Suricata on the Red Interface

Suricata was enabled through IPFire's web GUI under `Firewall > Intrusion Prevention` with the following configuration:

- **Interface:** Enabled on RED (WAN)
- **Mode:** Monitor traffic only (IDS — detection without blocking)
- **Ruleset:** Emergingthreats.net Community Rules (timestamp verified: 2024-06-07 21:37:08)

The full ET ruleset was downloaded and populated, containing 40+ rule categories spanning malware, exploits, scanning, DNS abuse, shellcode, SCADA, and more.

#### Scan Rule Analysis — ET SCAN ICMP IPTools

The `emerging-scan.rules` category was enabled and examined. One rule of note was **ET SCAN ICMP IPTools**, which detects a specific ICMP-based host discovery technique used by the IPTools network scanner. The rule fires when an ICMP echo request matching IPTools' characteristic packet structure arrives at the network — specifically targeting the tool's behavior of sending pings to enumerate live hosts on a subnet and mapping the network by correlating responses. From an attacker's perspective, this is a reconnaissance technique to identify active hosts before selecting targets. The Suricata rule pattern-matches against the IPTools packet signature to identify this activity even when ICMP appears superficially normal.

#### Nmap Scan Detection

With `emerging-scan.rules` enabled, an Nmap scan was run from Bodhi Linux against the IPFire WAN interface. IPFire's `Logs > IPS Logs` was reviewed to examine what Suricata detected.

The IPS logs showed alerts for a broad range of ports that a default Nmap scan probes, including:

| Port | Service |
|---|---|
| 3306 | MySQL |
| 5900/5902 | VNC |
| 5432 | PostgreSQL |
| 5800 | VNC HTTP |
| 1521 | Oracle SQL |
| 1433 | MSSQL |
| 21 | FTP |
| 22 | SSH |
| 23 | Telnet |
| 443 | HTTPS |

These alerts were generated because Suricata's scan rules match the characteristic probing patterns of Nmap — rapid sequential connection attempts to well-known service ports from a single source, with no established session context. The sheer volume and distribution of destination ports within a short time window is itself the signature.

#### OS Detection Probe Rule

The rule **ET SCAN NMAP OS Detection Probe** (SID 2018489) was enabled within `emerging-scan.rules`. An OS detection scan was run from Bodhi Linux:

```bash
nmap -O 10.0.5.9
```

The IPS logs captured multiple **ET SCAN NMAP OS Detection Probe** alerts with type "Attempted Information Leak," showing source `10.0.5.11:54308` probing destination port 42821, then 39987, in rapid succession. The repeated alert entries within the same second confirmed Nmap's TCP/IP fingerprinting behavior — sending crafted packets with unusual flag combinations (NULL, FIN, Xmas, etc.) to elicit OS-specific TCP stack responses.

---

### Part 2 — Outbound IDS (LAN Interface Monitoring)

Suricata was extended to also monitor Green (LAN) interface traffic, covering outbound connections from internal hosts.

#### DNS Query Monitoring — Suspicious TLD Detection

The `emerging-dns.rules` ruleset was enabled. From Ubuntu, Firefox was used to attempt a connection to `onion.to` — a Tor gateway domain associated with anonymous routing infrastructure. While the connection failed (server not found), the DNS lookup itself was captured by Suricata.

IPS log entry:

```
Date:        06/21 16:36:37
Name:        ET DNS Query for .to TLD
Type:        Potentially Bad Traffic
IP info:     10.0.5.5:44432 → 216.170.153.146:53
SID:         2027757
```

This rule fires on DNS queries for `.to` TLD domains because they are frequently associated with Tor exit node proxies, phishing infrastructure, and evasion techniques. Detecting the DNS lookup — even when the connection itself fails — is valuable for a SOC analyst because it identifies the intent of the internal host regardless of whether the outbound connection succeeded.

#### Compromised IP Detection — Threat Intelligence Matching

The `3coresec.rules` and `compromised.rules` rulesets were enabled. Firefox was used to visit `216.170.153.146` directly, which generated a Suricata alert:

```
Date:        06/21 16:36:37
Name:        ET DNS Query for .to TLD
Type:        Potentially Bad Traffic
IP info:     10.0.5.5:44432 → 216.170.153.146:53
SID:         2027757
```

The alert type "Potentially Bad Traffic" reflects that `216.170.153.146` appears in the `compromised.rules` intelligence list — a community-maintained list of IPs historically associated with malicious activity, command-and-control infrastructure, or compromised hosts used in attacks.

#### Reverse DNS / IP Reputation Analysis

A reverse DNS lookup on `216.170.153.146` identified the network block as assigned to **TDS Telecom**. Historical abuse reports from approximately 2021 associated this IP range with port scanning and exploitation activity.

#### False Positive Evaluation Methodology

The `compromised.rules` file located at `/var/lib/suricata/compromised.rules` on the IPFire console contains thousands of IP addresses with creation dates — not update dates. This creates a significant false positive risk: IPs that were malicious years ago may have changed ownership, been cleaned up, or been reassigned entirely.

A rigorous false positive evaluation workflow for a production SOC should include:

1. **Check the rule creation date** — if the entry is 2+ years old, treat the alert as a lower-confidence indicator requiring corroboration
2. **Run a current IP reputation lookup** via services such as VirusTotal, AbuseIPDB, or Shodan to check recent reporting
3. **Perform a reverse DNS lookup** to identify current ownership and compare against the historical abuse record
4. **Check WHOIS** for current ASN registration and network assignment changes
5. **Correlate with other telemetry** — a single `compromised.rules` alert with no supporting behavioral indicators (no unusual DNS, no lateral movement, no large data transfers) is low confidence on its own
6. **Examine the destination context** — was this a user browsing to a URL that resolved to this IP, or was the connection initiated programmatically? The latter is more suspicious

No single data point is sufficient. The combination of rule age, current reputation data, and behavioral context determines whether an alert warrants escalation.

---

## Skills Demonstrated

- **Network IDPS deployment** — Configured Suricata within IPFire on both WAN (inbound) and LAN (outbound) interfaces; understood the architectural difference between monitoring Red vs. Green traffic
- **Ruleset management** — Enabled and managed Emergingthreats.net community rulesets; understood rule category taxonomy (scan, DNS, compromised, shellcode, exploit, etc.)
- **Scan detection analysis** — Read Suricata IPS logs to identify Nmap default scan and OS detection probe signatures; correlated alert volume and port distribution patterns to scan behavior
- **DNS-based threat detection** — Understood how Suricata intercepts DNS queries for suspicious TLDs as an early indicator of potential data exfiltration or Tor usage, independent of connection success
- **Threat intelligence list analysis** — Applied `compromised.rules` and `3coresec.rules` community blocklists to detect outbound connections to historically malicious IPs
- **Reverse DNS and IP reputation investigation** — Used reverse DNS lookup to attribute `216.170.153.146` to TDS Telecom and correlate with historical abuse reports
- **False positive methodology** — Articulated a multi-step evaluation process for determining whether an aged threat intelligence alert represents a current threat or a stale false positive
- **IDPS-to-SIEM pipeline reasoning** — Explained how Suricata alert data (source IP, destination port, frequency, timestamp) maps to IDPS threshold-based detection rules

---

## References

- [IPFire Intrusion Prevention Documentation](https://wiki.ipfire.org/configuration/firewall/ips)
- [Emergingthreats.net Community Rules](https://rules.emergingthreats.net/)
- [Suricata Documentation](https://suricata.readthedocs.io/)
- [NIST SP 800-94 Rev. 1 — Guide to Intrusion Detection and Prevention Systems](https://csrc.nist.gov/publications/detail/sp/800-94/rev-1/draft)
- [AbuseIPDB — IP Reputation Lookup](https://www.abuseipdb.com/)
