# Forensic Imaging & Hashing
### Talon Ultimate Imager, FTK Imager, Tableau Write Blocker — Chain of Custody Verification


---

## Overview

Forensic imaging is the foundation of every digital investigation. Before any analysis can begin, the evidence drive must be duplicated in a way that is bit-for-bit accurate, write-protected, and cryptographically verified — so that any court or opposing counsel can confirm the working copy has not been altered from the original. This lab demonstrates the full forensic imaging and chain-of-custody workflow using industry-standard hardware and software tools: the Talon Ultimate standalone imager, a Tableau T8 hardware write blocker, and FTK Imager.

A 2GB suspect USB drive was imaged using two independent methods. SHA-256 and SHA-1/MD5 hashes were computed at each stage to verify integrity. The zip/unzip cycle was also tested to confirm that compressed transport of forensic images is forensically sound.

---

## Environment

| Component | Detail | Role |
|-----------|--------|------|
| Suspect Drive | 2GB USB thumb drive | Source evidence — write-protected throughout |
| Talon Ultimate | Standalone forensic imager | Hardware imaging device (air-gapped, no OS dependency) |
| Tableau T8 | Hardware write blocker | Prevents any writes to suspect drive during local imaging |
| Tableau T8 Firmware | 18:03:02 | Verified before use |
| FTK Imager | AccessData/Exterro | Software imager on host Windows machine |
| MultiHasher / CertUtil | Windows CLI tools | SHA-256, SHA-1, MD5 hash computation |
| GitBash | Windows bash environment | sha256sum / sha1sum / md5sum command line hashing |
| Target Drive | 32GB USB (Talon destination) | Image storage |
| Image File | DDCapture.001 | Raw dd image, ~1.85 GB |

---

## Findings Summary

| Step | Tool | Hash Algorithm | Result |
|------|------|---------------|--------|
| Talon imaging + verify | Talon Ultimate | SHA-256 | Source hash = Verify hash ✅ |
| Independent hash of DDCapture.001 | MultiHasher | SHA-256 | Matches Talon report ✅ |
| Compress image | 7-Zip | — | DDCapture.001.7z created |
| Hash of compressed file | MultiHasher | SHA-256 | `32296F4B4F46C6C6CCA02991D20CF5BB779BDD11191C92F69798B76425C2DAC3` |
| Unzip + re-hash | MultiHasher | SHA-256 | Matches original DDCapture.001 hash ✅ |
| FTK Imager imaging | FTK Imager + Tableau T8 | MD5 + SHA-1 | Computed = Report hash ✅ |
| Cross-tool hash check | CertUtil (MD5) | MD5 | FTK image MD5 = `4574166f5a0bc357ef194c17ed5c6888` ✅ |

---

## Walkthrough

### Part 1 — Talon Ultimate Forensic Imaging

The Talon Ultimate is a standalone hardware imager that operates independently of any host operating system, eliminating the risk of OS-level contamination of the evidence drive. The suspect 2GB USB drive was connected to the write-protected left-hand port; the 32GB destination drive to the right-hand port.

**Task configuration:**

```
Mode:              Drive to File
Source:            USB_S1
Destination:       USB_D1
Method:            dd
Segment Size:      4 GB
Unlock HPA:        No
Unlock DCO:        No
Error Granularity: 1 Sector
Hash Method:       SHA-256
Verify:            Primary
```

The task completed successfully. The Talon generated a PDF/HTML report containing hash information for the image.

**Talon imaging result:**

```
LBA Count:          3,891,200
Sector Size:        512 bytes
Hash Type:          SHA-256
Source Hash:        16ca27edf625808808a6ba75d6c267bff48efa7d607b5e32e34530d273183555
Result:             SUCCESS
Duration:           00:02:04
Time at Completion: 09:37:20
```

---

### Part 2 — Independent Hash Verification

After imaging, the DDCapture.001 file was independently hashed using MultiHasher on the host machine to verify it matched the Talon-generated hash — confirming the image was not corrupted during transfer from the 32GB drive.

**MultiHasher output (DDCapture.001):**

```
File: D:\Emmanuel_Franco-DDCapture\DDCapture.001
SHA-1:   C68E04F0D1E5F90A196326DC166F842ADF775247
SHA-256: 16CA27EDF625808808A6BA75D6C267BFF48EFA7D607B5E32E34530D273183555
```

SHA-256 matches the Talon source hash exactly — image integrity confirmed.

---

### Part 3 — Compressed Transport Verification

A critical question in forensic practice is whether compressing an image for transport or storage alters it. The image was compressed with 7-Zip, then the compressed archive was hashed, transferred to a new folder, unzipped, and re-hashed.

**Steps:**

```bash
# Compress
7z a Emmanuel_Franco-DDCapture.7z DDCapture.001

# Hash the archive
CertUtil -hashfile Emmanuel_Franco-DDCapture.7z SHA256
# Result: 32296F4B4F46C6C6CCA02991D20CF5BB779BDD11191C92F69798B76425C2DAC3

# Copy archive to new folder, unzip, re-hash original
CertUtil -hashfile DDCapture.001 SHA256
# Result: 16CA27EDF625808808A6BA75D6C267BFF48EFA7D607B5E32E34530D273183555
```

**Conclusion:** The SHA-256 hash of the unzipped copy is identical to the original. The zip/unzip process is non-destructive and forensically sound — compressed transport does not alter the evidence.

---

### Part 4 — FTK Imager with Tableau T8 Write Blocker

For the second imaging method, a Tableau T8 hardware write blocker was used. The write blocker sits between the suspect drive and the host PC, ensuring that any OS-level writes (driver installation, volume mounting, thumbnail generation) are blocked at the hardware level.

**Tableau T8 setup:**

```
Power switch: OFF before connecting any drives
Write blocker USB → host PC mini-USB port
Suspect 2GB USB → right-side port of write blocker
Power switch: ON
Firmware version verified: 18:03:02
```

FTK Imager was then used to create a raw (dd) image of the drive.

**FTK Imager configuration:**

```
Image type:           Raw (dd)
Source:               Physical drive (suspect USB)
Image fragment size:  0 (single file, no segmentation)
Destination:          D:\
Output filename:      Image of 12 thumb drive.dd
```

**FTK Imager verification results:**

```
Sector count:   3,891,200

MD5 Hash:
  Computed:       4574166f5a0bc357ef194c17ed5c6888
  Report hash:    4574166f5a0bc357ef194c17ed5c6888
  Verify result:  MATCH ✅

SHA1 Hash:
  Computed:       c68e04f0d1e5f90a196326dc166f842adf775247
  Report hash:    c68e04f0d1e5f90a196326dc166f842adf775247
  Verify result:  MATCH ✅

Bad blocks:     None found
```

**Cross-tool verification using CertUtil:**

```
D:\> CertUtil -hashfile "Image of 12 thumb drive.001" MD5
MD5 hash of D:\Image of 12 thumb drive.001:
4574166f5a0bc357ef194c17ed5c6888
CertUtil: -hashfile command completed successfully.
```

The MD5 hash of the FTK image matches both the FTK Imager report and the SHA-1 from the Talon image (after algorithm normalization), confirming that both tools produced identical, verified copies of the same suspect drive.

---

### Why Two Tools?

Using two independent imaging tools is a best practice in forensic investigations. If both tools produce images with matching hashes, it provides stronger evidentiary assurance than a single image. It also demonstrates that the imaging process itself — regardless of tool — produces consistent, reproducible results. Any discrepancy between the two hashes would indicate a problem with one of the imaging processes and require investigation before proceeding.

---

### Final Folder Contents

At the conclusion of the lab, the working directory contained:

```
DDCapture.001              # Original Talon raw image (1.85 GB)
DDCapture.001.txt          # Talon-generated task report with hash
Emmanuel_Franco-DDCapture.7z    # Compressed archive for transport
DDCapture.001.zip.sha256.txt    # SHA-256 of the compressed archive
Image of 12 thumb drive.dd      # FTK Imager raw image
```

---

## Skills Demonstrated

- **Forensic hardware imaging** — Talon Ultimate standalone imager operation, task configuration, write-protected sourcing
- **Hardware write blocking** — Tableau T8 setup and verification; firmware version documentation
- **Software forensic imaging** — FTK Imager raw (dd) image creation with case metadata
- **Cryptographic hash verification** — SHA-256, SHA-1, and MD5 across multiple tools; cross-tool hash matching
- **Chain of custody documentation** — recording all hash values, tool versions, and settings at each stage
- **Forensic soundness testing** — verifying that compression/decompression does not alter evidence
- **Command-line hashing** — `sha256sum`, `sha1sum`, `md5sum`, `CertUtil -hashfile` in GitBash/Windows CLI

---

## Framework Mapping

| Framework | Control | Application |
|-----------|---------|-------------|
| NIST SP 800-86 | Guide to Integrating Forensic Techniques | Forensic imaging methodology, write blocker use, hash verification |
| SWGDE Best Practices | Digital & Multimedia Evidence | Dual-tool imaging, hash-at-acquisition, chain of custody documentation |
| ISO/IEC 27037 | Digital Evidence Identification & Preservation | Write blocker requirement, bit-for-bit imaging, integrity verification |
| ACPO Principle 1 | UK Digital Evidence Principles | No action should change data held on a digital device — enforced by write blocker |

---

## References

- NIST SP 800-86: Guide to Integrating Forensic Techniques into Incident Response — https://csrc.nist.gov/publications/detail/sp/800-86/final
- Talon Ultimate User Manual — https://www.logicube.com/products/talon/
- Tableau T8 Write Blocker — https://www.opentext.com/products/tableau-forensic-bridges
- AccessData/Exterro FTK Imager — https://www.exterro.com/ftk-imager
- SWGDE Best Practices for Computer Forensics — https://www.swgde.org/documents/published
- ISO/IEC 27037:2012 — Guidelines for Identification, Collection, Acquisition, and Preservation of Digital Evidence
