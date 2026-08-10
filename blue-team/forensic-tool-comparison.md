# Forensic Tool Comparison — Bulk Extractor, Foremost, Photorec vs. Autopsy
### Artifact Extraction and Triage: Understanding What Each Tool Finds and Why


---

## Overview

No single forensic tool finds everything. A competent analyst knows not just how to run tools, but what each tool is optimized for, what it misses, and why those gaps exist. This lab runs three standalone extraction tools — Bulk Extractor, Foremost, and Photorec — against the same 2GB thumb drive image (DDCapture.001) that was previously analyzed with Autopsy, then performs a systematic side-by-side comparison of what each tool found uniquely versus what overlapped.

The goal is to develop triage judgment: given a specific investigative question, which tool or combination of tools should be deployed first?

---

## Environment

| Component | Detail | Role |
|-----------|--------|------|
| Analysis OS | CAINE 11 (Linux) | Forensic analysis VM |
| Disk Image | DDCapture.001 (1.85 GB) | 2GB USB thumb drive image — same image as Autopsy lab |
| Bulk Extractor | v1.x | Feature extraction from raw binary data |
| BEViewer | GUI frontend for Bulk Extractor | Visual histogram and artifact browsing |
| Foremost | v1.5.7 | Signature-based file carving |
| Photorec / QPhotorec | v7.2 | File carving with broader format support |
| Autopsy | v4.x (from prior lab) | Full forensic suite — baseline for comparison |
| Image hash (MD5) | 064ab24069ebac436b815f36468a3a5c | Verified before analysis |

---

## Findings Summary

| Tool | Primary Strength | Unique Output vs. Others |
|------|-----------------|--------------------------|
| Bulk Extractor | Pattern extraction from raw bytes | URL histograms, IP histograms, GPS coordinates, email histograms, AES keys, Ethernet MACs — none of these appear in Autopsy |
| Foremost | File carving by magic bytes | Recovered audio (WAV, WMV), video (MOV), and BMP files not surfaced by Autopsy |
| Photorec | Broadest file type recovery | Recovered the most JPEG files (543); recovered text files (145) and ZIP archives (67) not found by Foremost |
| Autopsy | Metadata + encrypted file detection | File timestamps, EXIF metadata, user activity timeline, encrypted/mismatched extension detection — none available in carving tools |

---

## Walkthrough

### Part 1 — Bulk Extractor

Bulk Extractor scans raw binary data byte-by-byte without any filesystem awareness. It does not carve files — instead it extracts structured patterns (email addresses, URLs, IP addresses, phone numbers, credit card numbers, GPS coordinates, cryptographic keys) and writes them to categorized output files.

**Command:**

```bash
bulk_extractor -o bulkt_output /home/student/Documents/DDCapture.001
```

**Completion output:**

```
All Threads Finished!
MD5 of Disk Image:    064ab24069ebac436b815f36468a3a5c
Phase 3. Creating Histograms
Elapsed time:         95.9493 sec
Total MB processed:   1992
Performance:          20.764 MBytes/sec (10.382 MBytes/sec/thread)
Total email features: 3302
```

**Output files generated:**

```
aes_keys.txt          ccn.txt               ccn_histogram.txt
domain.txt            domain_histogram.txt  email.txt
email_domain_histogram.txt  email_histogram.txt   ether.txt
ether_histogram.txt   ip.txt                ip_histogram.txt
json.txt              rfc822.txt            telephone.txt
telephone_histogram.txt     url.txt         url_histogram.txt
url_searches.txt      url_services.txt      windirs.txt
zip.txt               gps.txt               exif.txt
jpeg_carved.txt
```

**Notable artifacts in BEViewer:**

`email_histogram.txt` — most frequently occurring email addresses extracted from raw memory of documents, browser caches, and email clients resident on the drive. This histogram approach can quickly identify the most active communication accounts without opening individual files.

`ip_histogram.txt` — top IP addresses by frequency:

```
n=5374   192.168.2.7
n=1244   66.135.211.87
n=1097   170.224.8.51
n=419    66.151.149.10
n=398    66.150.96.111
n=362    206.14.107.140
```

The dominant address `192.168.2.7` is a private RFC1918 address — likely the local machine or a frequent LAN peer. External IPs appearing at high frequency could represent C2 servers, frequently visited web services, or other network activity of investigative interest.

`url_histogram.txt` — the top URL was `http://schemas.openxmlformats.org/drawingml/2006/main` (n=4147), indicating heavy use of Microsoft Office Open XML format files on the drive. This is unique to Bulk Extractor — Autopsy does not produce a URL frequency histogram.

`gps.txt` — GPS coordinates embedded in EXIF metadata of image files, extracted directly from raw bytes. This is one of Bulk Extractor's most forensically powerful features: GPS coordinates can place a device at a specific location even if the original photo files have been deleted.

**Comparison with Autopsy:** Both tools found email addresses and phone numbers. Autopsy found the same emails through its ingest modules, but Bulk Extractor additionally produced frequency histograms, URL searches, AES key candidates, and GPS data — none of which Autopsy surfaced in this configuration.

---

### Part 2 — Foremost

Foremost is a file carver that reads raw bytes, looks for known file header and footer signatures, and writes each carved file to a type-sorted output directory. It does not depend on filesystem metadata.

**Command:**

```bash
foremost -i /home/student/Documents/DDCapture.001 -o /home/student/Documents/foremost_output -v
```

**Completion output:**

```
Foremost version 1.5.7
Start: Tue Nov 5 00:14:49 2024
Finish: Tue Nov 5 00:23:48 2024
File: /dev/sdb1    Length: 1 GB (1992278016 bytes)
```

**Files recovered by type:**

```
/foremost_output/
  bmp/    docx/   gif/    htm/    jpg/
  mov/    ole/    pdf/    png/    ppt/
  pptx/   wav/    wmv/    zip/    audit.txt
```

**Comparison with Autopsy:**

Both Foremost and Autopsy recovered JPEG, PNG, PDF, DOCX, and PPTX files. The key differentiator: Foremost recovered **audio (WAV), video (MOV, WMV), and BMP** files that did not appear in Autopsy's output. Autopsy, on the other hand, recovered **file metadata** (creation timestamps, modification dates, EXIF) that Foremost does not produce — Foremost outputs raw file bytes only, with no metadata attached.

> Foremost is optimized for breadth of file type recovery. Autopsy is optimized for metadata richness around the files it finds.

---

### Part 3 — Photorec

Photorec uses a similar signature-based approach to Foremost but supports a broader range of file types and uses a partition-aware scanning method that often recovers more files, particularly JPEGs.

**Setup:** The DDCapture.001 file was renamed to DDCapture.raw to work around a QPhotorec recognition issue with the .001 extension in this CAINE version. QPhotorec was launched from the CAINE forensic tools menu.

**Recovery results (first run — full image):**

```
jpg:    543
txt:    145
zip:     67
apple:   53
png:     47
pdf:     41
doc:     25
gif:     24
tif:     19
asf:      4
others:   3
Total: ~971 files
```

**Second run (partition scan only — 197 files):**

```
txt:    116
jpg:     43
gif:     18
apple:   17
pdf:      2
zip:      1
```

**Comparison with Autopsy:**

Both tools recovered the same core JPEG and PDF files. Photorec recovered significantly more JPEGs (543 vs Autopsy's set) because it scans unallocated space that Autopsy may not fully process in all configurations. However, Photorec strips all metadata — files are renamed to sequential numbers (e.g., `f10908544.jpg`) with no original filename, path, or timestamp information preserved.

A file named `f0032363_Change_PIN_Secret_Word_Form_v1_0.pdf` was recovered uniquely by Photorec — this title suggests a sensitive document (PIN/password change form) that was found only through raw carving, not through the filesystem index. This demonstrates why carving tools are run even after Autopsy completes.

> Photorec recovers more files than Foremost, especially JPEGs. But without metadata, it requires more analyst time to triage the output. Autopsy provides context; Photorec provides volume.

---

### Tool Selection Framework

Based on this comparison, the following triage approach applies:

| Investigative Question | Best Tool |
|------------------------|-----------|
| What email addresses / IPs / URLs are on this drive? | Bulk Extractor |
| Were GPS coordinates embedded in any images? | Bulk Extractor (gps.txt) |
| What files were on this drive, with timestamps? | Autopsy |
| Are any files encrypted or have mismatched extensions? | Autopsy |
| What deleted files can be recovered (broadest recovery)? | Photorec |
| What audio/video files are present? | Foremost |
| Quick triage of communication artifacts | Bulk Extractor + BEViewer |
| Full evidentiary examination with chain of custody | Autopsy |

---

### Bonus — OSINT from GPS Data

Bulk Extractor's `gps.txt` output contained GPS coordinates embedded in image EXIF metadata. Cross-referencing with images tagged in Assessment 1 (Autopsy run), one image contained a visible name: **Cooper Himmerich, Class of 2017**.

OSINT pivot chain:
1. Google search: "Cooper Himmerich class of 2017" → Moorhead High School graduation result, name "Cooper Lee Himmerich"
2. LinkedIn search → profile showing attendance at Minnesota State Community College
3. Instagram → profile picture matched LinkedIn photo, confirming identity
4. Facebook → confirmed same individual, now working as a DJ

This demonstrates how GPS metadata extracted from a disk image can be combined with open-source intelligence to establish the identity of a device owner or associate — a technique used in both law enforcement investigations and corporate DFIR.

---

## Skills Demonstrated

- **Bulk Extractor** — pattern extraction from raw binary, histogram analysis, IP/email/URL/GPS artifact triage
- **Foremost** — signature-based file carving, magic byte recovery, audio/video file recovery
- **Photorec/QPhotorec** — broad-spectrum file carving, unallocated space recovery, partition-aware scanning
- **Tool comparison and triage judgment** — matching tool capability to investigative question
- **OSINT from forensic artifacts** — GPS coordinate → social media identity pivot chain
- **Forensic analysis methodology** — parallel tool execution, output cross-referencing, gap identification
- **Linux command-line forensics** — tool invocation, output directory management, audit log review

---

## Framework Mapping

| Framework | Control | Application |
|-----------|---------|-------------|
| NIST SP 800-86 | Forensic Tool Selection | Multiple tool deployment for comprehensive coverage |
| SWGDE Best Practices | Digital Evidence Examination | Tool validation through parallel execution and output comparison |
| SANS FOR500 | Windows Forensic Analysis | Artifact extraction triage methodology |
| MITRE ATT&CK — T1005 | Data from Local System | Understanding what data carving tools recover and why |

---

## References

- Bulk Extractor GitHub — https://github.com/simsong/bulk_extractor
- Foremost Documentation — https://foremost.sourceforge.net/
- Photorec / TestDisk — https://www.cgsecurity.org/wiki/PhotoRec
- Autopsy Digital Forensics — https://www.autopsy.com/
- NIST SP 800-86: Guide to Integrating Forensic Techniques — https://csrc.nist.gov/publications/detail/sp/800-86/final
- Gary Kessler File Signatures Table — https://www.garykessler.net/library/file_sigs.html
