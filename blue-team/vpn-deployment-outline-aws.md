# VPN Deployment – Outline VPN on AWS EC2

**Course:** CYBR 3400 – Network Security  
**Tools:** Outline VPN (Alphabet/Jigsaw), AWS EC2, Outline Manager, Outline Client  
**Skills Demonstrated:** Cloud VPS provisioning, VPN server deployment, client configuration, multi-device VPN management

---

## Overview

Deployed a personal VPN server using **Outline** — a production-grade VPN platform developed by Alphabet's Jigsaw unit, originally designed to protect journalists working in restricted environments. Provisioned an AWS EC2 instance as the VPS host, deployed the Outline server via Docker, and successfully connected from both a desktop and mobile device.

---

## Environment

| Component | Details |
|-----------|---------|
| VPN Platform | Outline (Shadowsocks-based) |
| VPS Provider | AWS EC2 (t2.micro, Free Tier) |
| Server Region | US East (Ohio) — `3.144.134.245` |
| Server Software | Outline Server (Docker container) |
| Management Tool | Outline Manager (Windows) |
| Client Devices | Windows desktop + iOS mobile (bonus) |

---

## Architecture

```
[Client Device]
     |
     | Encrypted Shadowsocks tunnel
     v
[AWS EC2 – 3.144.134.245:38766]
  └── Docker: Outline Server
     |
     v
[Public Internet]
```

Outline uses the **Shadowsocks** protocol — an encrypted proxy designed to be resistant to traffic analysis and censorship, making it more robust than standard VPN protocols in adversarial network environments.

---

## Deployment Steps

### 1. AWS EC2 Instance Provisioning

- Created an AWS Free Tier account
- Launched a `t2.micro` Ubuntu instance in the US East (Ohio) region
- Configured the security group to allow the required TCP/UDP ports for Outline (port range dynamically assigned by the Outline Manager)
- Generated and saved the EC2 SSH key pair for instance access

### 2. Outline Server Deployment

- Downloaded **Outline Manager** on the local Windows machine
- Selected AWS as the hosting provider and followed the guided setup
- Outline Manager automatically:
  - Connected to the EC2 instance via SSH
  - Pulled the official Outline Docker image
  - Started the Outline server container
  - Generated the management API URL and secret
- Setup completed with the server reporting:
  - **857 kB** data transferred in the last 30 days
  - **1 key** (access key) provisioned

### 3. Access Key Management

- Outline Manager generated a single **"Main key"** for initial access
- Clicked the share icon next to the Main key to open the **"Get connected"** dialog
- The dialog provided a `ss://` Shadowsocks URI for client configuration

### 4. Desktop Client Configuration (Windows)

- Downloaded and installed the Outline Client for Windows
- Added the server by pasting the `ss://` access key URI
- Server appeared as **"Outline Server – 3.144.134.245:38766"**
- Connected successfully — client showed **"Connected"** state with the teal connection indicator

### 5. Mobile Client Configuration (iOS) – Bonus Device

- Downloaded Outline from the iOS App Store
- Scanned/imported the same access key
- Connected to the same server (`3.144.134.245:38766`)
- iOS confirmed **"Connected to Outline Server"** with VPN indicator in the status bar
- Verified the server IP by checking external IP — confirmed as `3.144.134.245` (Amazon Technologies, Columbus, Ohio)

---

## Verification

Connected desktop client IP verified via external IP check service:

| Field | Value |
|-------|-------|
| IPv4 Address | `3.144.134.245` |
| IPv6 | Not detected |
| ISP | Amazon Technologies Inc. |
| City | Columbus |
| Region | Ohio |
| Country | United States |

Traffic confirmed routing through the VPN — local ISP IP no longer visible to external services.

---

## Security Considerations

- Access keys (Shadowsocks URIs) are cryptographic credentials — stored securely, not committed to public repositories
- EC2 instance runs minimal software (Docker + Outline only) to reduce attack surface
- Free tier bandwidth limits (~15 GB/month transfer) appropriate for personal use; not suitable for routing all home traffic continuously
- SSH key pair for EC2 instance management stored in private secure storage
- AWS free tier expiration monitored — calendar reminder set to avoid unexpected charges

---

## Skills Demonstrated

- Cloud VPS provisioning and configuration (AWS EC2)
- Docker-based server application deployment
- VPN server setup and access key management (Outline/Shadowsocks)
- Multi-device VPN client configuration (Windows + iOS)
- Network traffic verification and IP routing confirmation
- Security-conscious infrastructure management (minimal attack surface, credential handling)

---

## References

- [Outline VPN – getoutline.org](https://www.getoutline.org/)
- [Outline GitHub Repository](https://github.com/Jigsaw-Code/outline-server)
- [AWS Free Tier](https://aws.amazon.com/free/)
- [Shadowsocks Protocol](https://shadowsocks.org/)
