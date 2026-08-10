# macOS Forensics — Autopsy Investigation
### Criminal Case Analysis: Weapons Trafficking, Controlled Substances, Stolen Credit Cards


---

## Overview

This investigation involved a forensic examination of a macOS High Sierra disk image (`img_High_Sierra.vmdk`) seized from a suspect facing multiple criminal charges. The image was previously ingested into Autopsy over approximately 36 hours due to hardware errors on the original drive. The drive itself became inoperable after initial imaging; the prosecution provided the saved Autopsy case rather than the raw image. Hash verification confirmed the image had not been altered since the initial acquisition.

The examining attorney retained forensic analysis services to determine whether sufficient evidence existed to support each of five alleged charges before proceeding to a grand jury.

**Charges under investigation:**
1. Trafficking in illegal weapons
2. Dealing and/or manufacture of controlled substances
3. Money laundering
4. Use of stolen credit cards to purchase drugs and weapons
5. Extortion/blackmail of a utah.gov government official

---

## Environment

| Component | Detail | Role |
|-----------|--------|------|
| Analysis Tool | Autopsy v4.x | Digital forensics suite |
| Evidence File | MacOS_quemu.aut | Saved Autopsy case (macOS High Sierra VMDK) |
| Image Status | Partially corrupted — hardware failure during imaging | Some sectors unreadable; hash verified unchanged |
| OS | macOS High Sierra | Suspect machine OS |
| User Account | student | Primary user directory examined |
| File System | HFS+ (via ExtX volume) | macOS native filesystem |

---

## Evidence Log

| # | Artifact | Location | Relevance |
|---|----------|----------|-----------|
| 1 | Documents directory listing | `/Users/student/Documents/` | Multiple incriminating files present |
| 2 | Cannabis growing guides (PDF) | `/Users/student/Documents/` | Controlled substance manufacture |
| 3 | Methamphetamine recipe (PDF) | `/Users/student/Documents/` | Controlled substance manufacture |
| 4 | Colt M16A2 firearm image | `/Users/student/Documents/` | Weapons trafficking |
| 5 | Tor Browser installed | `/Users/student/Library/Application Support/TorBrowserData/` | Dark web access tool |
| 6 | DuckDuckGo search history — weapons | Extracted text strings | Weapons research: M16A2, machine guns, ammo |
| 7 | DuckDuckGo search history — drugs | Extracted text strings | Meth recipes, cannabis growing, drug purchases |
| 8 | DuckDuckGo search history — stolen credit cards | Extracted text strings | Stolen CC for sale searches |
| 9 | DuckDuckGo search history — dark web | Extracted text strings | Tor browser searches, darknet fraud sites |
| 10 | Malware sample repositories | Extracted text strings | theZoo, MalwareBazaar, VirusSign searches |

---

## Walkthrough

### Part 1 — Filesystem Examination

#### Student Home Directory

Navigating to `Data Sources → img_High_Sierra.vmdk → vol_vol5 → Users → student`, three subdirectories were identified:

```
.bash_sessions     (shell session history)
.Trash             (deleted items — recoverable)
Downloads          (downloaded files)
```

#### Documents Directory — Incriminating Files

The `/Users/student/Documents/` directory contained a mixture of documents and images that directly relate to three of the five charges:

```
Best Weed Strains Lists | Leafly.pdf             ← controlled substances
CC Brian's Club 2.png                            ← possible stolen CC data
CC Brian's Club.png                              ← possible stolen CC data
colt-m16a2-17__90700.1641570791.jpg              ← illegal weapons (M16A2 is NFA-regulated)
Complete_Beginners_Cannabis_Growing_Guide.pdf    ← controlled substance manufacture
Growing Weed Indoors: A Beginner's Guide [2022]  ← controlled substance manufacture
How to make Methamphetamine at home | RESEAF     ← controlled substance manufacture (meth)
methamphetamine recipe at DuckDuckGo.pdf         ← controlled substance manufacture
Private Info Brian's Club.png                    ← possible stolen CC / fraud data
```

The filename `CC Brian's Club` is significant: **Brian's Club** is a well-known dark web stolen credit card marketplace. Files named with this reference strongly suggest the suspect was engaged with stolen payment card data.

The Colt M16A2 is a select-fire military rifle classified as a machine gun under the National Firearms Act (NFA). Civilian possession without a pre-1986 registered transfer is a federal felony under 26 U.S.C. § 5861.

#### Thumbnail Gallery — Controlled Substance Content

The `Complete_Beginners_Cannabis_Growing_Guide.pdf` contained 51 embedded images including: cannabis plants at various growth stages, grow room equipment (carbon filters, HID lighting systems), measurement devices, and packaging materials. This is not general reference material — it is operational growing instruction.

The methamphetamine recipe documents similarly contained step-by-step synthesis content.

#### Installed Software — Tor Browser

Under `/Users/student/Library/Application Support/TorBrowserData/`, the following directory structure confirmed Tor Browser installation (not merely a download):

```
TorBrowserData/
  Browser/
  Tor/
  UpdateInfo/
```

The presence of `TorBrowserData` in the Application Support directory — rather than in Downloads — confirms the application was actively installed and used, not merely downloaded. Tor Browser provides anonymized access to both the clearnet and `.onion` dark web services.

---

### Part 2 — Analyzed Data & Search History

#### Web Search History — Weapons (Evidence #6)

Extracted text strings from the Autopsy keyword search and browser artifact modules showed DuckDuckGo search history containing:

```
Dumps Search
Colt M16A2 *New in Box* Transferable Machine Gun 5.56mm
Guns, Ammo & Accessories - Online Gun Dealers | Impact Guns
Guns - Class 3 - Machine Guns - Impact Guns
DuckDuckGo — Privacy, simplified [multiple weapon-related queries]
```

These searches confirm active intent to locate and potentially purchase restricted firearms through online channels.

#### Web Search History — Controlled Substances (Evidence #7)

```
growing cannabis indoors at DuckDuckGo [6+ repeated queries]
How to Grow Marijuana Indoors | Leafly
Best Weed Strains Lists | Leafly
methamphetamine recipe at DuckDuckGo [12+ repeated queries]
methamphetamine receptors at DuckDuckGo
How to Make Crystal Meth Step by Step the Easy Way at Home
Frank's Mama's Meth Recipe - True Homemade Taste
How to make Methamphetamine at home | RESEARCH CHEMICALS OFFICIAL
White Trash Magazine: Dope Recipes
```

The volume and repetition of meth recipe searches (12+ distinct queries) indicates deliberate research, not casual curiosity.

#### Web Search History — Stolen Credit Cards (Evidence #8)

```
stolen credi cards - Google Search [multiple queries]
stolen credi cards for sale - Google Search
THE BEST 5 DARKNET FRAUD SITES OF JANUARY 2020
Millions of credit card details for sale on dark web for as little as 75p
unicc equivalent - Google Search
```

The search for "unicc equivalent" is particularly notable: **Unicc** was a major dark web stolen credit card marketplace (taken down in 2022). Searching for equivalents indicates active intent to source stolen payment card data.

#### Web Search History — Dark Web Access (Evidence #9)

```
What is the dark web (darknet)?
tor browswer - Google Search [multiple queries]
Tor Project | Success [confirmed successful access]
THE BEST 5 DARKNET FRAUD SITES OF JANUARY 2020
```

The Tor Project success page appearing in history confirms the suspect successfully connected to the Tor network, corroborating the installed Tor Browser finding.

#### Web Search History — Malware Repositories (Evidence #10)

```
List of Mac viruses, malware and security flaws - Macworld UK
Malware: Where can I find virus samples (real) for analysis?
Malware Sample Sources — New & Maintained | by Buket Gençaydın
MalwareSamples (Mr. Malware) · GitHub
VirusShare.com
VirusSign | Malware Research & Data Center
MalwareList - VirusSign
MalwareBazaar | Browse malware samples
GitHub - ytisf/theZoo: A repository of LIVE malwares
GitHub - jstrosch/malware-samples: Malware samples, analysis exercises
GitHub - MalwareSamples/Macos-Malware-Samples
```

The suspect was searching for and potentially downloading live malware samples including macOS-specific samples. This could relate to extortion/blackmail tooling, though no direct connection to the utah.gov charge was established.

---

### Part 3 — Charge-by-Charge Assessment

#### Charge 1: Trafficking in Illegal Weapons
**Evidence: Supports prosecution**

The Colt M16A2 image, combined with repeated DuckDuckGo searches for machine guns and NFA-regulated firearms on online dealers, establishes both possession of firearm imagery and active intent to locate such weapons. The M16A2 is a select-fire, NFA-regulated machine gun — civilian transfers have been prohibited since 1986.

**Grand jury recommendation:** Sufficient circumstantial evidence to present. Physical weapon not recovered from digital evidence alone, but the combination of imagery, search history, and dark web access tools establishes a credible pattern.

---

#### Charge 2: Dealing and/or Manufacture of Controlled Substances
**Evidence: Strongest charge — strongly supports prosecution**

Multiple PDF documents with operational drug synthesis and cultivation content were found in the Documents folder — not in Downloads or Trash, indicating intentional retention. Twelve or more repeated meth recipe searches, six or more cannabis growing searches, and downloaded guides spanning cannabis cultivation and meth synthesis collectively demonstrate sustained, purposeful research into drug manufacture.

**Grand jury recommendation:** Sufficient evidence. Document possession combined with search history volume is compelling.

---

#### Charge 3: Money Laundering
**Evidence: Insufficient**

No financial records, transaction logs, cryptocurrency wallets, or communications referencing laundering activity were found in the digital evidence. Money laundering typically leaves traces in banking apps, cryptocurrency wallets, or communication about fund transfers — none of these were present.

**Grand jury recommendation:** Insufficient digital evidence to support this charge in isolation.

---

#### Charge 4: Stolen Credit Cards
**Evidence: Supports investigation — warrants follow-up**

Files named `CC Brian's Club.png` and `Private Info Brian's Club.png` reference a known stolen credit card marketplace. Search history confirmed active searching for stolen card markets and darknet fraud sites. The `unicc equivalent` search confirms familiarity with the stolen card ecosystem.

**Grand jury recommendation:** Sufficient to establish probable cause for further investigation. The Brian's Club file content would need to be reviewed (image contents were not extracted in this partial analysis) to establish definitive possession of stolen card data.

---

#### Charge 5: Extortion/Blackmail of utah.gov Official
**Evidence: Insufficient**

No email communications, messages, documents, or search history referencing utah.gov, any government officials, or extortion activity were found. The keyword search for `utah.gov` yielded only SUU (Southern Utah University) faculty and staff listings — consistent with a student doing academic research, not targeting a government official.

**Grand jury recommendation:** No digital evidence to support this charge.

---

## Skills Demonstrated

- **Autopsy case examination** — filesystem navigation, analyzed data review, keyword search, metadata analysis
- **macOS forensics** — HFS+ filesystem structure, Application Support directory, macOS-specific artifact locations
- **Evidentiary reasoning** — charge-by-charge analysis, distinguishing supporting evidence from insufficient evidence
- **Browser artifact analysis** — DuckDuckGo search history extraction, URL pattern analysis
- **Installed application forensics** — distinguishing installed vs. downloaded software via directory structure
- **Dark web indicator identification** — Tor Browser installation, darknet marketplace references
- **Neutral forensic reporting** — presenting evidence objectively without advocating for a specific outcome

---

## Framework Mapping

| Framework | Control | Application |
|-----------|---------|-------------|
| NIST SP 800-86 | Forensic Techniques | Systematic evidence collection and analysis methodology |
| Federal Rules of Evidence Rule 901 | Authentication | Hash verification confirming image integrity before analysis |
| SWGDE Best Practices | Digital Evidence Examination | Structured examination of filesystem, analyzed data, keyword hits |
| MITRE ATT&CK — T1426 | System Information Discovery (Mobile) | Tor Browser as anonymization/dark web access tool |
| MITRE ATT&CK — T1539 | Steal Web Session Cookie | OAuth/session token forensics relevance |

---

## References

- Autopsy Digital Forensics Platform — https://www.autopsy.com/
- NIST SP 800-86: Guide to Integrating Forensic Techniques — https://csrc.nist.gov/publications/detail/sp/800-86/final
- Tor Project — https://www.torproject.org/
- Brian's Club — Known stolen CC marketplace (DOJ reference) — https://www.justice.gov/
- NFA Machine Gun Restrictions — 26 U.S.C. § 5861, Hughes Amendment (FOPA 1986)
- theZoo Malware Repository — https://github.com/ytisf/theZoo
- SWGDE Best Practices for Computer Forensics — https://www.swgde.org/
