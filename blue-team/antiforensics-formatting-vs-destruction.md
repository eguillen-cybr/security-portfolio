# Antiforensics — Formatting vs. Data Destruction
### FAT32 Format, dd Zero-Fill, dd Random-Fill: What Survives and What Doesn't


---

## Overview

Suspects and insiders attempting to destroy evidence frequently reach for the most obvious tool available: the format function. What most people do not know is that a standard OS format does not erase data — it only rewrites filesystem metadata, leaving the underlying bytes intact and fully recoverable. This lab demonstrates that reality through three controlled experiments on the same 2GB USB drive, then tests two genuine data destruction methods.

Understanding antiforensics is directly relevant to SOC and DFIR work: an analyst needs to know whether a wiped drive can still yield evidence, and an investigator needs to recognize when a suspect has used a method that actually destroys data versus one that merely gives the appearance of doing so.

**Three methods tested:**
1. Standard FAT32 OS format
2. Overwrite with zeroes (`dd if=/dev/zero`)
3. Overwrite with random data (`dd if=/dev/urandom`)

Each method was followed by file carving (Foremost) and artifact extraction (Bulk Extractor) to measure what survived.

---

## Environment

| Component | Detail | Role |
|-----------|--------|------|
| Analysis OS | CAINE 11 (Linux) | Forensic analysis VM |
| Target Drive | 2GB USB (PHOTOSII, FAT32) | Suspect drive — reimaged between each test |
| Baseline Image | DDCapture.001 | Original drive contents restored between tests via dd |
| File Carving | Foremost v1.5.7 | Recovery attempt after each wipe method |
| Artifact Extraction | Bulk Extractor | Artifact scan after each wipe method |
| Disk Identification | dmesg | Verified device node (/dev/sdb1) before each operation |
| Imaging Tool | ImageUSB (Passmark) | Windows-side reimaging option (DDCapture.001 → DDCapture.img rename) |

---

## Findings Summary

| Method | Foremost Recovery | Bulk Extractor Artifacts | Data Destroyed? |
|--------|-------------------|--------------------------|-----------------|
| FAT32 OS format | ✅ All original files recovered | ✅ Full histograms, emails, IPs | ❌ No |
| dd zero-fill | ❌ 0 files extracted | ❌ All outputs empty | ✅ Yes |
| dd random-fill | ❌ 0 files extracted | ⚠️ 1 domain, 1 email, 1 phone (noise) | ✅ Effectively yes |

---

## Walkthrough

### Part 1 — FAT32 Format (Format Only)

#### Setup

A photo (`Emmanuel_Guillen.jpg`) was added to the suspect drive alongside its existing contents. The Windows Disk Management tool confirmed the drive formatted as FAT32 (PHOTOSII, E:, 1.92 GB).

**Before format:** Drive contained ~346 files including the newly added self-photo, JPEGs, PDFs, Word documents, and other mixed file types.

**Format performed:** Right-click drive → Format → FAT32 → Quick Format → Confirm.

Windows showed the drive as empty after format completion.

#### Recovery via Foremost

The formatted drive was connected to the CAINE VM. dmesg confirmed device node `/dev/sdb1`. Foremost was run directly against the device:

```bash
foremost -i /dev/sdb1 -o /home/student/foremost_output -v
```

**Result: All original files recovered.** The post-format file count matched the pre-format count. The self-photo (`Emmanuel_Guillen.jpg`) was found and opened successfully — filename was reassigned by Foremost (e.g., `02232152.jpg`) but image content was intact.

#### Bulk Extractor on Formatted Drive

```bash
bulk_extractor -o be_output_formatted /dev/sdb1
```

Output: Full histograms recovered — email addresses, IP addresses, photo metadata — essentially identical to the pre-format scan. The format operation had no meaningful effect on recoverable data.

#### Why Format Doesn't Work

A standard OS format rewrites only:
- The File Allocation Table (FAT) entries — marking all clusters as available
- The root directory — removing file name and metadata entries

The actual data bytes in the cluster data area are **not touched**. The sectors that held file content remain bit-for-bit identical to their pre-format state. File carvers like Foremost scan these raw sectors looking for magic byte signatures — they never consult the filesystem at all, so the erased FAT entries are irrelevant.

> **Investigative implication:** A suspect who formats their drive before surrendering it has not destroyed any evidence. Standard forensic carving will recover it.

---

### Part 2 — Zero-Fill with dd

#### Setup

The drive was reimaged to its original state using dd:

```bash
dd bs=1M if=DDCapture.001 of=/dev/sdb conv=noerror
```

Verified via file manager that all original files were present.

#### Zero-Fill

```bash
dd bs=1M if=/dev/zero of=/dev/sdb1
```

This command overwrites every byte on the partition with `0x00`. The operation completed, replacing all data sectors with null bytes.

#### Recovery Attempt via Foremost

```bash
foremost -i /dev/sdb1 -o /home/student/foremost_output -v
```

**Foremost audit.txt result:**

```
Foremost version 1.5.7
File: /dev/sdb1
Length: 1 GB (1992278016 bytes)
Start: Tue Nov 5 00:14:49 2024
Finish: Tue Nov 5 00:23:48 2024
0 FILES EXTRACTED
```

Zero files recovered. Every byte having been overwritten with `0x00`, there are no magic byte signatures for Foremost to match — no `FF D8` headers, no `%PDF-` markers, nothing.

#### Bulk Extractor on Zero-Filled Drive

All output files were empty. No emails, IPs, URLs, or any other artifacts were extractable from a drive filled with null bytes. The zero-fill was completely effective.

#### Why Zero-Fill Works

When every sector is overwritten with the same value (0x00), the original content is gone. Unlike format, which only updates metadata, `dd if=/dev/zero` writes to every addressable sector on the device. File carving tools look for byte patterns — when every byte is zero, no patterns exist to find.

> **Investigative implication:** A zero-filled drive cannot be recovered with standard forensic carving tools. This is a genuine data destruction method.

---

### Part 3 — Random Data Fill with dd

#### Setup

Drive reimaged to original state again, verified present.

#### Random-Fill

```bash
dd bs=1M if=/dev/urandom of=/dev/sdb1
```

This command overwrites every byte with cryptographically random data generated by the Linux kernel's `/dev/urandom` entropy source. This takes longer than zero-fill because the kernel must generate random numbers in real time.

#### Recovery Attempt via Foremost

```bash
foremost -i /dev/sdb1 -o /home/student/foremost_output2 -v
```

**Foremost audit.txt result:**

```
0 FILES EXTRACTED
```

No files recovered. Random data does not contain any recognizable file signatures.

#### Bulk Extractor on Random-Filled Drive

BEViewer showed minimal output: a single domain entry, one email fragment, and one phone-like string. These are almost certainly false positives — random data will occasionally contain byte sequences that match the patterns Bulk Extractor searches for, purely by chance. The data is noise, not recoverable evidence.

**BE Viewer assessment:** The single domain entry, email, and phone number found would not be anything recognizable or verifiable — random bytes occasionally match pattern signatures by coincidence. No actionable intelligence is recoverable.

#### Why Random-Fill Works (and is Arguably More Secure Than Zero-Fill)

Overwriting with random data defeats both file carving and, in theory, advanced physical recovery techniques. With zero-fill, a sophisticated adversary with physical access to the storage medium might theoretically detect faint magnetic remnants of the original data underneath the zeros (though this is increasingly theoretical with modern high-density storage). Random data provides no such pattern baseline.

In practice, both methods are effective against software-based forensic recovery. For the highest assurance (e.g., classified data), multiple passes or physical destruction are required per standards like DoD 5220.22-M or NIST SP 800-88.

---

## Comparative Summary

```
Method              Foremost    Bulk Extractor    Verdict
─────────────────────────────────────────────────────────
FAT32 format        All files   Full artifacts    DATA SURVIVES — NOT secure
dd if=/dev/zero     0 files     0 artifacts       DATA DESTROYED — Secure
dd if=/dev/urandom  0 files     Noise only        DATA DESTROYED — More secure
```

**Best practice for drive disposal:** `dd if=/dev/urandom` (single pass is sufficient for modern flash and HDD media per NIST SP 800-88). For SSDs, manufacturer secure erase commands or physical destruction are preferred due to wear leveling.

---

## Skills Demonstrated

- **Antiforensics awareness** — understanding what format does and does not do at the byte level
- **File carving** — Foremost against raw device nodes, interpreting audit.txt output
- **Data destruction methods** — `dd if=/dev/zero`, `dd if=/dev/urandom`, understanding when each is appropriate
- **Bulk Extractor triage** — distinguishing genuine artifacts from false-positive pattern matches in random data
- **Linux disk operations** — `dmesg`, `dd`, device node identification, forensic reimaging
- **Forensic reasoning** — explaining recovery/non-recovery outcomes in terms of underlying byte-level mechanics
- **Investigative implications** — translating technical findings into actionable conclusions for an investigation

---

## Framework Mapping

| Framework | Control | Application |
|-----------|---------|-------------|
| NIST SP 800-88 Rev 1 | Guidelines for Media Sanitization | Clear (overwrite) vs. Purge vs. Destroy — zero-fill = Clear level |
| NIST SP 800-86 | Forensic Techniques | Understanding data persistence after formatting |
| DoD 5220.22-M | National Industrial Security Program | Multi-pass overwrite standard for classified media |
| MITRE ATT&CK — T1485 | Data Destruction | Adversary technique of intentional data wiping |
| MITRE ATT&CK — T1070.004 | File Deletion | Understanding what deletion methods actually accomplish |

---

## References

- NIST SP 800-88 Rev 1: Guidelines for Media Sanitization — https://csrc.nist.gov/publications/detail/sp/800-88/rev-1/final
- NIST SP 800-86: Guide to Integrating Forensic Techniques — https://csrc.nist.gov/publications/detail/sp/800-86/final
- Foremost Documentation — https://foremost.sourceforge.net/
- Bulk Extractor — https://github.com/simsong/bulk_extractor
- Linux dd Manual — https://man7.org/linux/man-pages/man1/dd.1.html
- MITRE ATT&CK T1485 Data Destruction — https://attack.mitre.org/techniques/T1485/
- MITRE ATT&CK T1070.004 File Deletion — https://attack.mitre.org/techniques/T1070/004/
