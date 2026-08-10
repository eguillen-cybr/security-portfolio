# Android Mobile Forensics — Cellebrite UFED Reader
### SQLite Account Databases, OAuth2 Tokens, Call Logs, GPS, SMS Evidence



---

## Overview

Mobile devices are now among the most evidence-rich artifacts in any digital investigation. A smartphone carries call logs, GPS history, SMS communications, application data, account credentials, and authentication tokens — all persisted in SQLite databases that survive across reboots and partial factory resets. This lab uses Cellebrite UFED Reader to analyze a forensic extraction from a Samsung Android device, examining both the parsed "Analyzed Data" view and the raw SQLite database layer.

Cellebrite UFED (Universal Forensic Extraction Device) is the industry-standard mobile extraction platform used by law enforcement agencies worldwide. The `.ufdr` file analyzed here is not a raw system image — it contains only content Cellebrite identified as having potential evidentiary value, with known system files excluded via hash matching.

---

## Environment

| Component | Detail | Role |
|-----------|--------|------|
| Tool | Cellebrite UFED Reader | Mobile forensic analysis platform |
| Evidence File | Samsung CDMA .ufdr | Cellebrite extraction from Android device |
| Device | Samsung Android (CDMA) | Suspect mobile device |
| Carrier | Verizon | Mobile network operator |
| Phone Number | (805) 910-5153 | Device MSISDN |
| Bluetooth ID | D0:FC:CC:4B:05:70 | Hardware identifier |
| Database Type | SQLite | Android's primary application data store |

---

## Findings Summary

| Artifact Category | Key Finding | Evidentiary Value |
|------------------|-------------|-------------------|
| Call Logs | 3 outgoing calls; contacts include "Mom" and two others | Establishes communication timeline and associate network |
| Device Locations | GPS coordinates with timestamps; visited Chipotle, Best Buy | Places device/suspect at specific locations at specific times |
| SMS Messages | Incriminating text: "I will bring them today for you to slip them past your boss" | Direct evidence of coordination; admissible communication record |
| accounts.db | Google, social, and app accounts with hashed passwords | Account inventory; identifies online presence |
| authtokens (OAuth2) | Active session tokens for linked accounts | May enable account access under warrant; session takeover forensics |

---

## Walkthrough

### Part 1 — Extraction Summary

After opening the `.ufdr` in UFED Reader, the **Extraction Summary** pane provided device identifiers:

```
Bluetooth ID:   D0:FC:CC:4B:05:70
Carrier:        Verizon
Phone Number:   (805) 910-5153
```

The **Device Content** panel on the right listed all artifact categories extracted by Cellebrite. Three categories were selected for deep examination based on investigative relevance.

---

### Part 2 — SQLite Database Analysis (accounts.db)

Under `Image13 (ExtX) / Root / system / users / 0 / accounts.db`, the raw SQLite database was opened in UFED's database viewer.

#### Account Records

The `accounts` table contained a list of accounts registered on the device — each entry showing the account name (email/username), account type (e.g., `com.google`, `com.facebook.auth.login`), and an associated password field.

**Key observation:** The passwords were not stored in plaintext. Android stores account passwords in hashed form. The specific hash format visible in the database was not a standard bcrypt or SHA hash — it appeared to be an Android-internal credential token format.

**Could the stored password be used to log into Gmail directly?**

No. The hashed credential stored in accounts.db cannot be submitted directly to Google's authentication endpoint. Google's login flow requires either the plaintext password or a valid session token — the stored hash is an internal Android representation not accepted by web authentication.

#### OAuth2 Authtokens

The `authtokens` table contained active OAuth2 tokens associated with the device's Google and application accounts.

**How OAuth2 works:**

1. The user authenticates to an authorization server (e.g., Google) with their credentials
2. The authorization server issues an **access token** (short-lived) and optionally a **refresh token** (longer-lived)
3. The application presents the access token to resource servers (Gmail, Drive, etc.) to access protected resources
4. When the access token expires, the refresh token is exchanged for a new one — no password re-entry required

**Could the OAuth2 tokens be used to access the account (under warrant)?**

Potentially, if the tokens have not expired and the session has not been revoked. Access tokens are typically short-lived (1 hour), but refresh tokens can persist for weeks or months. In a forensic context with a valid warrant, an active refresh token could be presented to Google's token endpoint to obtain a new access token, granting access to the account's cloud data. This is a live forensic technique used in some investigations, though it requires careful legal authorization and carries risk of evidence contamination.

---

### Part 3 — Call Logs

**Analyzed Data → Call Logs**

Three call records were present in the extraction:

```
Call 1: Outgoing → "Mom"        Duration: [recorded]
Call 2: Outgoing → [contact]    Duration: [recorded]
Call 3: Outgoing → [contact]    Duration: [recorded]
```

All three were outgoing calls. The call log included timestamps, duration, and the associated contact or number.

**Evidentiary value:**

Call logs establish a communication timeline. In a criminal investigation, knowing who the suspect called — and when — can confirm contact with co-conspirators, verify alibis, or contradict a suspect's statements. If a suspect claims they did not contact a particular person around the time of a crime, call logs provide concrete contradiction.

The outgoing-only pattern in this extraction is also notable: it may indicate the device was used primarily to initiate contact rather than receive it, or that incoming calls were not captured by this extraction method.

---

### Part 4 — Device Locations

**Analyzed Data → Device Locations**

The extraction contained GPS location records with exact coordinates and timestamps. Two specific locations were identifiable:

```
Location 1: Chipotle [GPS coordinates + timestamp]
Location 2: Best Buy  [GPS coordinates + timestamp]
```

**Evidentiary value:**

GPS location data can place a device — and by extension its owner — at a specific physical location at a specific time. This is powerful corroborating evidence in cases where physical presence is disputed. Combined with other artifacts (call logs, SMS, purchase records), a location timeline can reconstruct a suspect's movements.

In a murder investigation, for example, GPS data placing the suspect's device at the crime scene at the time of the incident could be decisive corroborating evidence. If the phone was dropped at the scene, location records, device identifiers (Bluetooth ID, phone number), and account information could all link back to the suspect.

---

### Part 5 — SMS Messages

**Analyzed Data → SMS Messages**

The SMS extraction included message content, timestamps, direction (sent/received), and the associated phone number.

**Notable message found:**

```
Direction: Outgoing
Content:   "I will bring them today for you to slip them past your boss"
```

**Evidentiary value:**

This message suggests coordination of a covert activity — specifically, smuggling something past a supervisor. In the context of a corporate theft or insider threat investigation (e.g., stealing proprietary documents while a manager is absent), this message would constitute direct evidence of planning and intent. It demonstrates pre-meditated coordination with a third party.

SMS messages are among the most legally compelling digital artifacts because they are direct, timestamped communications authored by the suspect. Unlike browser history or file access logs, which can be explained away as passive activity, an outgoing SMS reflects deliberate intent.

---

## Key Concepts

**Why SQLite databases matter in mobile forensics:** Android and iOS store virtually all application data — messages, contacts, browser history, location records, health data, app credentials — in SQLite databases. These databases persist across app restarts and often survive partial factory resets. Even deleted records can sometimes be recovered from SQLite free pages using specialized tools.

**Why OAuth2 tokens are forensically significant:** A valid refresh token is functionally equivalent to account access. Unlike passwords, tokens are stored locally on the device in plaintext (in the authtokens table), making them immediately available to a forensic examiner with a lawful extraction. This is why mobile device security relies heavily on remote wipe capabilities — once physical access is obtained, session tokens can be extracted without knowing the account password.

**Why Cellebrite's hash exclusion matters:** The `.ufdr` format excludes known system files by hash. This means the extraction is smaller, faster to analyze, and pre-focused on user-generated and user-modified content — reducing noise and analyst time.

---

## Skills Demonstrated

- **Cellebrite UFED Reader** — extraction file analysis, device content navigation, analyzed data interpretation
- **SQLite database forensics** — Android accounts.db structure, account type identification, credential storage format
- **OAuth2 token analysis** — understanding token lifecycle, forensic implications of active refresh tokens
- **Mobile artifact triage** — call logs, GPS records, SMS evidence assessment for evidentiary value
- **Evidence articulation** — translating raw digital artifacts into investigative conclusions
- **Mobile platform awareness** — Android filesystem structure (ExtX), application data organization

---

## Framework Mapping

| Framework | Control | Application |
|-----------|---------|-------------|
| NIST SP 800-101 Rev 1 | Guidelines on Mobile Device Forensics | Mobile extraction methodology, evidence categories |
| SWGDE Mobile Device Forensics | Best Practices | UFED extraction handling, SQLite database examination |
| MITRE ATT&CK — T1409 | Access Stored Application Data | Android credentials and tokens stored in accessible databases |
| MITRE ATT&CK — T1417 | Input Capture (Mobile) | Understanding what data persists on mobile devices |
| RFC 6749 | OAuth 2.0 Authorization Framework | OAuth2 flow, token types, refresh token lifetime |

---

## References

- Cellebrite UFED Product Page — https://cellebrite.com/en/ufed/
- NIST SP 800-101 Rev 1: Mobile Device Forensics — https://csrc.nist.gov/publications/detail/sp/800-101/rev-1/final
- SQLite Documentation — https://www.sqlite.org/docs.html
- RFC 6749: The OAuth 2.0 Authorization Framework — https://datatracker.ietf.org/doc/html/rfc6749
- MITRE ATT&CK T1409 — https://attack.mitre.org/techniques/T1409/
- Android Security Architecture — https://source.android.com/security
