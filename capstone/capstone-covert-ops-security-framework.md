# Covert Operations Network Security Framework (RFP-CIA-073): Capstone Design Project

## Overview

This project was a graduate-level cybersecurity capstone completed as part of a Southern Utah University Cybersecurity program, structured as a formal RFP response to a fictional CIA solicitation for a covert operations communications security framework. It was completed by a 4-person capstone team over a full semester, culminating in a 30+ page technical proposal covering encryption architecture, hardware-backed authentication, anti-detection systems, emergency data destruction, and DoD-aligned compliance controls.

My primary ownership on the project covered three core technical domains: Secure Authentication Protocols, Advanced Encryption, and the design of Phantom, a custom covert communications application. I also contributed smaller sections covering training materials, audit procedures, and incident response protocols. The remaining sections, including Mission Command Center hardware architecture, budget, scheduling, and SWOT analysis, were led by other team members and are summarized here only for context.

---

## Scope and Objective

The proposal addressed a scenario in which intelligence field agents require covert, tamper-resistant communication with a central command node while operating in hostile or high-surveillance environments. The design had to account for a specific and demanding threat model:

- Nation-state adversaries with deep packet inspection and traffic analysis capabilities
- The "Harvest Now, Decrypt Later" threat, where encrypted traffic is captured today for decryption once quantum computing matures
- Physical device compromise, including seizure, tampering, or forensic extraction
- The need for plausible deniability if a device is inspected by an adversary

Every technical decision in my sections was made against this threat model rather than a generic enterprise one, which is the main difference between this project and standard security architecture work.

---

## My Contribution 1: Advanced Encryption Architecture

### Problem

Standard encryption (AES, RSA, ECC) is secure today but vulnerable to the harvest-now-decrypt-later threat once cryptographically relevant quantum computers exist. A framework designed to protect classified intelligence for years or decades needs to assume the adversary is already capturing ciphertext for future decryption.

### Design

I designed the encryption layer around three complementary mechanisms rather than a single algorithm, since each serves a different role and risk tradeoff:

**FIPS 203 (ML-KEM, derived from CRYSTALS-Kyber)** as the primary key encapsulation mechanism. This is a NIST-standardized post-quantum algorithm used to establish shared secrets over an insecure channel without relying on the discrete-log or factoring problems that quantum computers threaten. I paired this conceptually with FIPS 204/205 for post-quantum digital signatures, so both confidentiality and authenticity are quantum-resistant, not just one.

**One-Time Pad encryption** as a deliberate fallback for the highest-sensitivity communications. OTP is information-theoretically unbreakable when the pad is truly random, used once, and kept secret, properties that make it immune to future cryptanalytic advances by definition rather than by computational hardness assumption. I scoped this as a last-resort option for mission-critical exchanges rather than default traffic, since OTP key distribution is operationally expensive at scale.

**NSA Type 1 algorithm support** as an accommodation layer, allowing the framework to interoperate with existing certified government cryptographic hardware rather than forcing a wholesale replacement of trusted infrastructure.

### Dynamic Key Management

Each of the above mechanisms has different key lifecycle requirements: ML-KEM needs frequent key encapsulation per session, OTP needs secure one-time pad generation and destruction after use, and Type 1 hardware has its own certified key handling procedures. I designed a unified key management layer responsible for automated distribution, storage, renewal, auditing, and revocation, built to accommodate each algorithm's distinct lifecycle rather than forcing a one-size-fits-all key rotation policy across incompatible cryptographic primitives.

### Design Rationale

The core lesson from this section: post-quantum migration isn't a drop-in algorithm swap. It requires layering multiple cryptographic approaches with different assumptions (computational hardness vs. information-theoretic security) and building key management flexible enough to serve all of them without becoming an operational bottleneck.

---

## My Contribution 2: Secure Authentication Protocols

### Problem

Field agents need a device that is simultaneously (a) safe to carry and use in front of an adversary without raising suspicion, and (b) capable of secure, authenticated, high-assurance communication when needed. A single-profile device can't do both: anything secure enough for classified communication looks suspicious if inspected casually, and anything inconspicuous enough to avoid suspicion isn't secure enough for classified use.

### Design: Dual-Profile Architecture

I designed a two-profile system on the device (a Google Pixel 8, selected for its Titan M2 security chip):

- **Daily profile**: PIN-only authentication, deliberately less secure looking, populated with ordinary apps, used openly including in the presence of adversaries. Its entire purpose is to look unremarkable.
- **Secret profile**: Requires multi-factor authentication (PIN + biometric, biometric + CAC, or PIN + CAC), invisible from the daily profile, and the only profile from which Phantom (the covert communications app) can be launched.

The security property this creates is compartmentalization: even a full compromise or casual inspection of the daily profile reveals nothing about the secret profile's existence.

### Hardware Root of Trust: Titan M2

I anchored all authentication material, cryptographic keys, one-time passwords, certificates, and biometric templates in the Titan M2 secure element rather than general OS storage. Titan M2 operates independently of the main processor and handles verified boot, secure key storage, and cryptographic operations. The design goal was that even a fully compromised OS should not be able to exfiltrate authentication secrets, since they never leave the secure element in usable form.

### PKI and Zero Trust

I designed the authentication layer around an internally managed PKI rather than relying on third-party certificate authorities, so every identity verification and encrypted session is anchored in certificates the organization fully controls. Combined with CAC-based private key storage and mandatory MFA for the secret profile, this enforces a zero trust model: no device or session is implicitly trusted, every access is independently authenticated and logged.

### Lockout and Anomaly Detection

To mitigate brute-force PIN attacks, I built in an automatic lockout after three consecutive failed attempts, paired with real-time anomaly detection on authentication patterns so that unusual login behavior (timing, frequency, location inconsistency) triggers alerts rather than waiting for a hard failure threshold.

---

## My Contribution 3: Phantom (Custom Covert Communications Application)

### Problem

Off-the-shelf secure messaging apps (Signal, WhatsApp, etc.) are well-known and their presence on a device is itself a signal that something sensitive is happening. A purpose-built application was needed that could only exist on the secret profile, be functionally invisible otherwise, and degrade gracefully if standard channels are cut off.

### Design

Phantom is the application layer that sits on top of the authentication and encryption architecture described above. Key design decisions I made:

- **Installation scope**: Phantom is installed exclusively on the secret profile and is undetectable from the daily profile, reinforcing the compartmentalization goal of the dual-profile design.
- **End-to-end encryption**: All communications are encrypted end-to-end using the ML-KEM/OTP layered approach described above, so the application layer inherits the post-quantum security properties rather than implementing its own weaker crypto.
- **Code integrity**: Phantom is cryptographically signed, and any tampering with the binary is logged rather than silently allowed to execute. I added runtime integrity checks that validate application code and memory space during execution, to catch in-memory tampering that static signature checks alone would miss.
- **HTTPS fallback server**: I designed an embedded HTTPS server as a contingency channel for scenarios where the primary covert channel is unavailable, such as device loss or network denial. Access requires the same authentication strength as Phantom itself, with an intentionally lengthened PIN requirement to resist brute-forcing given the fallback channel's higher exposure.
- **Traffic obfuscation integration**: Phantom incorporates traffic obfuscation so that even encrypted traffic doesn't have an easily fingerprinted network signature, working in concert with the anti-detection mechanisms designed elsewhere in the framework (domain fronting, protocol mimicry, timing obfuscation, and steganographic channels).

### Design Rationale

The throughline across Phantom's design is that every security property assumes the device may be physically inspected or seized. Detectability, not just decryptability, is treated as a primary threat, which shaped the invisibility requirements, the signed-and-monitored execution model, and the fallback channel design.

---

## Smaller Contributions

I also authored, in less depth than the sections above:

- **Training materials**: A phased training curriculum (classroom/simulation, quick-reference field guides, and live field training scenarios) covering profile switching, MFA procedures, Phantom usage, and emergency purge triggers, with mandatory annual recertification.
- **Audit procedures**: Recurring audit objectives spanning compliance verification (NIST SP 800-53, FIPS 140-3), operational assurance testing, and role-based access reviews, with SCAP-based automated baseline auditing (OpenSCAP, Tenable Nessus) and Titan M2-signed audit logs for integrity.
- **Incident response protocols**: An IRP aligned to NIST SP 800-61 Rev. 2, adapted for the dual-profile/Phantom environment, covering detection and severity triage, containment (secret profile lockout, remote Phantom disable, emergency purge activation), and post-incident review feeding back into training and baseline configuration updates.

---

## Team Context (Summarized, Not My Work)

The broader proposal, developed by a 4-person capstone team, also included a physical Mission Command Center architecture (Cisco firewall/VPN gateway, Supermicro communications server), emergency purge hardware and policy design, automated security baseline enforcement, DoDIN integration, budget and scheduling, and a SWOT analysis. These sections provided the operational and organizational context my authentication, encryption, and Phantom design work needed to plug into, but were led by teammates and are not detailed here.

---

## Skills Demonstrated

- Post-quantum cryptographic design (FIPS 203/ML-KEM) and hybrid classical/PQC architecture
- Information-theoretic security concepts (One-Time Pad) and their operational tradeoffs
- Dynamic, multi-algorithm key lifecycle management design
- Zero Trust architecture and internally managed PKI design
- Hardware root-of-trust integration (Titan M2 secure element)
- Multi-factor authentication design across risk-tiered profiles
- Secure application design: code signing, runtime integrity verification, fallback channel design
- Traffic obfuscation and anti-detection system integration
- NIST SP 800-61-aligned incident response protocol design
- SCAP-based compliance auditing design (OpenSCAP, Tenable Nessus)
- Technical writing and formal RFP-style proposal authorship

---

## References

- NIST FIPS 203: Module-Lattice-Based Key-Encapsulation Mechanism Standard
- NIST SP 800-207: Zero Trust Architecture
- NIST SP 800-61 Rev. 2: Computer Security Incident Handling Guide
- NIST SP 800-53: Security and Privacy Controls for Information Systems
- FIPS 140-3: Security Requirements for Cryptographic Modules
