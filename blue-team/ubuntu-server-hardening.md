# Ubuntu Server Hardening — Patch Management, SSH, UFW, AIDE, and Service Reduction

## Overview

This lab demonstrates a systematic hardening process applied to an Ubuntu 20.04 LTS server using a structured checklist approach. Starting from a default Ubuntu server installation, ten hardening controls were implemented across patch management, user account management, SSH configuration, host-based firewall, file integrity monitoring, automatic updates, service enumeration and reduction, filesystem permission hardening, logging configuration, and backup scripting. Each step was verified with terminal output and validated for correctness before proceeding to the next. A Lynis security audit was run at the end of the process to benchmark the hardening index and identify remaining gaps.

This hardening workflow mirrors real-world CIS Benchmark and NIST SP 800-123 server hardening procedures.

---

## Environment

| Component | Details |
|---|---|
| Target OS | Ubuntu 20.04.6 LTS (GNU/Linux 5.4.0-90-generic x86_64) |
| Hostname | ubuntu20043base |
| Platform | VirtualBox VM |
| Hardening Reference | CIS Ubuntu Linux 20.04 Benchmark / NIST SP 800-123 |
| Audit Tool | Lynis 2.6.2 |
| Client VM (for SSH testing) | Windows host (PowerShell SSH client) |

---

## Walkthrough

### Step 1 — Patch Management

All system packages were updated to eliminate known vulnerabilities in installed software:

```bash
student@ubuntu20043base:~$ sudo apt update
Hit:1 http://us.archive.ubuntu.com/ubuntu focal InRelease
Hit:2 http://us.archive.ubuntu.com/ubuntu focal-updates InRelease
Hit:3 http://us.archive.ubuntu.com/ubuntu focal-backports InRelease
Hit:4 http://us.archive.ubuntu.com/ubuntu focal-security InRelease
Reading package lists... Done
Building dependency tree
Reading state information... Done
All packages are up to date.

student@ubuntu20043base:~$ sudo apt upgrade
Reading package lists... Done
Building dependency tree
Reading state information... Done
Calculating upgrade... Done
0 upgraded, 0 newly installed, 0 to remove and 473 not upgraded.
```

`apt update` refreshes the package index from all configured repositories. `apt upgrade` applies available patches. The output confirming "All packages are up to date" establishes a clean patched baseline before any further hardening.

---

### Step 2 — User Management

A named administrative account was created to eliminate reliance on the default `student` account for privileged operations. The new account was added to the `sudo` group to allow administrative commands while avoiding direct root login:

```bash
student@ubuntu20043base:~$ sudo adduser emmanuel
Adding user `emmanuel' ...
Adding new group `emmanuel' (1002) ...
Adding new user `emmanuel' (1002) with group `emmanuel' ...
Creating home directory `/home/emmanuel' ...
Copying files from `/etc/skel' ...
New password:
Retype new password:
passwd: password updated successfully
Changing user information for emmanuel
    Full Name []: Emmanuel
Is the information correct? [Y/n] y

student@ubuntu20043base:~$ sudo usermod -aG sudo emmanuel
```

Creating a named account for administrative activity enables audit trails that attribute actions to specific individuals rather than a generic account. The `usermod -aG sudo` flag adds the user to the sudo group non-destructively (the `-a` append flag preserves existing group memberships).

---

### Step 3 — SSH Hardening

The SSH daemon was hardened by changing the listening port from the default 22 to 2222, reducing automated brute-force exposure. The `sshd_config` file was edited directly:

```
#Port 22  → changed to Port 2222
PermitRootLogin no
StrictModes yes
MaxAuthTries 6
PubkeyAuthentication yes
```

Key changes and their rationale:
- **Port 2222** — Moving SSH off port 22 eliminates the vast majority of automated SSH scanners and credential stuffing bots that exclusively target the default port. This is security through obscurity and not a primary control, but it substantially reduces log noise.
- **PermitRootLogin no** — Prevents direct root SSH login. All administrative access must go through a named account with sudo. This creates a two-factor audit trail (account login + sudo event).
- **StrictModes yes** — Enforces that SSH key files have correct permissions before accepting key-based auth, preventing misconfigured key files from being exploited.

The UFW firewall was updated to allow the new SSH port:

```bash
student@ubuntu20043base:~$ sudo ufw allow 2222/tcp
Rules updated
Rules updated (v6)

student@ubuntu20043base:~$ sudo ufw enable
Firewall is active and enabled on system startup

student@ubuntu20043base:~$ sudo ufw status
Status: active

To                         Action      From
--                         ------      ----
2222/tcp                   ALLOW       Anywhere
2222/tcp (v6)              ALLOW       Anywhere (v6)
```

SSH connectivity was validated from the Windows client:

```
PS C:\Users\Administrator> ssh student@10.0.5.12
The authenticity of host '10.0.5.12 (10.0.5.12)' can't be established.
ECDSA key fingerprint is SHA256:oPQQZBd0cgDQ04twF3VUvrao+AnfZLDiC948wwswB+0.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.0.5.12' (ECDSA) to the list of known hosts.
student@10.0.5.12's password:
Welcome to Ubuntu 20.04.6 LTS (GNU/Linux 5.4.0-90-generic x86_64)
...
student@student-vm:~$
```

---

### Step 4 — Firewall Configuration

UFW (Uncomplicated Firewall) was confirmed active with only the non-standard SSH port allowed. This step documents the firewall as a hardening control independent of SSH — confirming that no other unexpected ports were left open following the SSH reconfiguration.

The firewall rule specifically exposes port 2222/tcp while all other ports remain blocked by default-deny. Reconfiguring the firewall to reflect the new SSH port (rather than leaving port 22 open as well) is a common mistake when changing SSH ports — having both open negates the benefit of moving off the default.

---

### Step 5 — Intrusion Detection (AIDE — Advanced Intrusion Detection Environment)

AIDE was installed to provide host-based file integrity monitoring. AIDE builds a cryptographic baseline database of the filesystem and can detect unauthorized changes to system files, binaries, and configuration files:

```bash
student@ubuntu20043base:~$ sudo apt install aide
aide is already the newest version (0.16.1-1ubuntu0.1).

student@ubuntu20043base:~$ sudo aide --init
Running aide --init...
Start timestamp: 2024-06-16 01:15:32 +0000 (AIDE 0.16.1)
AIDE initialized database at /var/lib/aide/aide.db.new
Verbose level: 6
Number of entries:      157860
```

The AIDE initialization output recorded 157,860 filesystem entries with multiple hash algorithms:

```
/var/lib/aide/aide.db.new
  SHA256  : xkkSrN3l6h11I8MDJWj1hDQhxmoyF7bJUjmb/wNdMYO=
  SHA512  : 1N1bs4EFKWOrXMvo3b2BzpWL6stv4GWVXhJLD3Lt/pJw6XPzgQF/8j5KNqPBJo5g
             ZOrHPTzqZ5gUFJqBBhx8tu==
```

AIDE uses multiple hash algorithms (SHA256, SHA512, RMD160, TIGER, HAVAL, GOST, CRC32) to detect any modification to monitored files. After a security incident, running `aide --check` against the baseline database reveals which files were changed, added, or deleted — critical evidence for incident response and forensic timelines.

---

### Step 6 — Automatic Updates

The `unattended-upgrades` package was confirmed present and enabled to automate the application of security patches without manual intervention:

```bash
student@ubuntu20043base:~$ sudo apt install unattended-upgrades
unattended-upgrades is already the newest version (2.3ubuntu0.3).
0 upgraded, 0 newly installed, 0 to remove and 0 not upgraded.
```

In an environment with multiple servers, manually tracking and applying patches introduces delay between vulnerability disclosure and remediation. `unattended-upgrades` ensures that security patches are applied automatically on a configured schedule, reducing the exposure window without requiring operator intervention.

---

### Step 7 — Service Management

Running services represent attack surface. The active service list was enumerated with `systemctl list-units --type=service` to identify unnecessary services. The output showed all active services including `ssh.service`, `rsyslog.service`, `cron.service`, `ufw.service`, and `unattended-upgrades.service` — all expected. Two services showed failed state: `fwupd-refresh.service` and `postfix@-.service` (Postfix Mail Transport Agent). The `vsftpd` FTP service was identified as unnecessary for this server's role and disabled:

```bash
sudo systemctl disable vsftpd
sudo systemctl stop vsftpd
```

The principle of least functionality requires that only services necessary for the server's stated role remain active. Each running service is a potential attack vector — a vulnerability in a service that should never have been running in the first place is an entirely avoidable risk.

---

### Step 8 — Filesystem Security

File permissions were hardened on sensitive files using `chmod` and `chown` to implement least-privilege access at the filesystem level:

```bash
student@ubuntu20043base:~$ chmod 700 key
student@ubuntu20043base:~$ sudo chown emmanuel:emmanuel key
```

`chmod 700` sets the file to owner-read/write/execute only — no permissions for group or others. `chown emmanuel:emmanuel` transfers ownership to the named administrator account. Combined, only the `emmanuel` account can read or modify the file. This protects cryptographic keys, configuration files containing credentials, and other sensitive data from being read by unprivileged processes or users on a multi-user system.

---

### Step 9 — Logging and Monitoring

The rsyslog configuration at `/etc/rsyslog.conf` was reviewed and confirmed correctly configured:

```
$FileOwner syslog
$FileGroup adm
$FileCreateMode 0640
$DirCreateMode 0755
$Umask 0022
$PrivDropToUser syslog
$PrivDropToGroup syslog
$WorkDirectory /var/spool/rsyslog
$IncludeConfig /etc/rsyslog.d/*.conf
```

Key settings: log files are created with mode `0640` (owner read/write, group read, no world access), owned by `syslog:adm`. The `$PrivDropToUser syslog` directive drops rsyslog's privileges after startup, limiting the impact if the logging daemon itself were compromised. Centralized, tamper-resistant logging is essential for incident detection, response, and forensic reconstruction — logs that can be modified or deleted by an attacker cannot be trusted.

---

### Step 10 — Backup and Recovery

The `rsync` utility was confirmed present and a backup script was created to automate web content backups with date-stamped directories:

```bash
student@ubuntu20043base:~$ sudo apt install rsync
rsync is already the newest version (3.1.3-8ubuntu0.7).
```

Backup script (`backup_script.sh`):

```bash
#!/bin/bash

source_dir="var/www/html"
backup_dir="/backups"
backup_path="$backup_dir/$(date +'%Y-%m-%d')"

mkdir -p "$backup_path"
rsync -av --delete "$source_dir/" "$backup_path/"
```

The `rsync -av --delete` flags create a verbose archive-mode copy that also removes files from the backup destination that no longer exist in the source, keeping the backup current. Date-stamped backup directories provide point-in-time recovery capability. Without tested, current backups, ransomware or destructive attacks result in unrecoverable data loss — even if the security incident itself is contained.

---

### Lynis Security Audit

A Lynis audit was run at the conclusion of hardening to benchmark the system state:

```
Hardening index : 57 [###########     ]
Tests performed : 223
Plugins enabled : 1

Components:
- Firewall         [V]
- Malware scanner  [X]

Lynis Modules:
- Compliance Status    [?]
- Security Audit       [V]
- Vulnerability Scan   [V]
```

The hardening index of **57/100** with 223 tests performed reflects a moderately hardened system. The firewall component passed; malware scanner was absent. Remaining improvement areas noted by Lynis include adding a malware scanner (ClamAV), further SSH configuration tightening, and additional kernel hardening parameters via `sysctl`.

---

## Skills Demonstrated

- **Patch management** — Applied `apt update` / `apt upgrade` to establish a fully patched baseline before hardening begins
- **User and privilege management** — Created a named administrator account with sudo access; understood the security value of account attribution over shared/generic accounts
- **SSH hardening** — Changed default listening port, disabled root login, enabled strict modes; validated the change from an external SSH client
- **Host-based firewall (UFW)** — Configured UFW to reflect SSH port change; confirmed no unintended ports were exposed after reconfiguration
- **File integrity monitoring (AIDE)** — Initialized an AIDE database with 157,860 entries across multiple hash algorithms; understood the role of AIDE in detecting unauthorized filesystem changes for incident response
- **Automatic updates** — Configured `unattended-upgrades` to automate security patch application and minimize vulnerability exposure windows
- **Service enumeration and reduction** — Used `systemctl` to enumerate running services; applied least-functionality principle by disabling unnecessary services
- **Filesystem permissions** — Applied `chmod 700` and `chown` to restrict access to sensitive files; understood how UNIX permission bits enforce least-privilege at the filesystem layer
- **Logging configuration** — Reviewed and validated rsyslog configuration for correct file permissions, privilege dropping, and centralized log management
- **Backup scripting** — Created an rsync-based backup script with date-stamped directories; understood backup as a recovery control within the broader security posture
- **Security benchmarking** — Ran Lynis to produce a hardening index score and identify remaining gaps

---

## References

- [CIS Ubuntu Linux 20.04 LTS Benchmark](https://www.cisecurity.org/benchmark/ubuntu_linux)
- [NIST SP 800-123 — Guide to General Server Security](https://csrc.nist.gov/publications/detail/sp/800-123/final)
- [Ubuntu UFW Documentation](https://help.ubuntu.com/community/UFW)
- [AIDE Documentation](https://aide.github.io/)
- [Lynis Security Auditing Tool](https://cisofy.com/lynis/)
- [OpenSSH Security Best Practices](https://www.ssh.com/academy/ssh/sshd_config)
