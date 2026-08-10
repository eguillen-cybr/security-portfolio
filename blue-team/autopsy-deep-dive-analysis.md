# Autopsy Deep-Dive: Filesystem Forensics, EXIF Analysis and Artifact Recovery

## Overview

This lab covers advanced use of Autopsy 4.19.1 to perform a complete forensic examination of a FAT32 disk image. The analysis covers filesystem enumeration, embedded metadata extraction, extension mismatch detection, magic byte verification, keyword searching, and timeline reconstruction. These are core skills for digital forensics and SOC work.

The disk image came from a 2 GB USB drive previously imaged with a Tableau T8 write blocker and Talon Ultimate imager, SHA-256 verified. The scenario: a thumb drive submitted as potential evidence in a case involving suspected infrastructure threats.

---

## Environment

| Component | Details |
|---|---|
| Analysis Tool | Autopsy 4.19.1 |
| Disk Image Format | dd (.001 split image) |
| Filesystem | Win95 FAT32 |
| Image Hash (SHA-256) | `32296F4B4F46C6C6CCA02991D20CF5BB779BDD11191C92F69798B76425C2DAC3` |
| Ingest Modules Enabled | Plaso (log2timeline), File Type Identification, EXIF Parser, Extension Mismatch Detector, Keyword Search |
| Ingest Modules Disabled | Hash Lookup, Virtual Machine Extractor, Android Analyzer, DJI Drone Analyzer, iOS Analyzer |
| Platform | CAINE Linux (forensic workstation) |

---

## Walkthrough

### 1. Case Creation and Image Ingestion

A new single-user Autopsy case was created with the examiner name and email recorded for chain of custody purposes. The `.001` image file was added as a **Disk Image or VM File** data source.

Key ingest decisions:
- **Hash Lookup disabled** - the image hash was already verified at imaging time, so re-hashing every individual file would add time without much investigative value at this stage
- **Plaso enabled** - log2timeline builds a unified forensic timeline from filesystem timestamps, which is useful for event reconstruction
- **Extension Mismatch enabled** - automatically flags files where the file header (magic bytes) does not match the declared extension

Autopsy was allowed to finish all ingest tasks before analysis started. The blue progress bar in the status bar needs to fully disappear, not just reach 100%, before the results are reliable.

---

### 2. Filesystem Enumeration

Expanding **Data Sources > Vol 1 / Vol 2** in the left-hand pane revealed the raw directory structure of the FAT32 volume.

**Findings:**
- Filesystem type: **Win95 FAT32**
- Files present included documents, PDFs, PowerPoints, and images
- Example file identified: `02 - Direct-Dep Info Flyer.pdf`

FAT32 does not support access control lists (ACLs) or per-file ownership at the OS level. That means file attribution has to come from **embedded metadata** rather than filesystem permissions, which is an important caveat when trying to assign ownership in a legal context.

---

### 3. PDF Internal Structure Analysis

Two types of PDFs were present on the drive:

**Standard text-layer PDFs** (e.g., `Business Account Analysis` series):
- Internal structure contained embedded `.jpg` and `.png` image assets alongside searchable text
- Autopsy's keyword search returned hits within these files

**Scanned document PDFs** (e.g., `HVAC Maintenance Agreement w SeaBoard.pdf`):
- Internal structure consisted of `.tif` image files, one per page
- No searchable text layer present
- Keyword search returned zero hits for phrases visible in the document

**Why this matters:**
The HVAC file was created by scanning a physical document. Without Optical Character Recognition (OCR), the content is invisible to keyword searches. This has a few practical implications:
1. Keyword searches cannot be used to rule out scanned documents as relevant
2. Any scanned document needs manual review or OCR preprocessing before automated analysis
3. This is a common blind spot in automated forensic pipelines that gets overlooked

The HVAC file was tagged **Follow Up** for manual review.

---

### 4. EXIF Metadata Extraction

Navigating to **Analysis Results > EXIF Metadata** exposed camera metadata embedded in image files across the drive.

**Camera models identified:**
- Canon EOS Digital Rebel XS
- Kodak Z712 IS Zoom Digital Camera
- Kodak C875 Zoom Digital Camera

**Camera serial number extraction (Canon DSLR):**

One file in the `CRX87` series, taken by the Canon DSLR, was extracted for external analysis:

```
Right-click file > Extract Files > Save to working directory
```

After uploading to an online EXIF metadata tool, the **Maker Notes** field (a Canon-specific EXIF extension) contained the camera serial number:

```
Camera Serial Number: 0770204950
```

**Evidentiary value of serial numbers:**
- If the physical camera is seized, a serial number match ties that specific device to a specific image
- Useful for establishing possession in cases involving illicit imagery
- **Lens and focal length data** can also be used to reconstruct scene geometry, estimating camera-to-subject distance and corroborating or refuting a suspect's account of where a photo was taken

---

### 5. Document Metadata and Owner Attribution

Navigating to **Data Artifacts** (document metadata) revealed embedded author and ownership fields in Office documents.

**Hurricane Irene.ppt metadata:**

| Field | Value |
|---|---|
| File Created (metadata) | 2011-08-24 16:30:37 MDT |
| Filesystem Creation Date | September (current year, when disk was imaged) |
| Document Owner | Jonathan Seys |
| User ID | Different from owner |

**Why the dates do not match:**
The filesystem creation date reflects when the file was written to this specific disk, not when the document was originally authored. Document metadata timestamps travel with the file itself and in this case predate the disk imaging by years. This is expected and does not indicate tampering.

**Why the User ID does not match the owner:**
The "Owner" field is set by the authoring application at creation time. The "User ID" reflects the Windows account active when the file was last saved or modified. These are often different people if a document has passed through multiple hands.

**Implication for attribution:**
Neither field alone is enough to place a specific person at the keyboard. A document attributed to "Jonathan Seys" could have been authored by Seys on his own machine, saved by someone else on a shared machine, or modified and re-saved by a third party after the original was created.

OSINT corroboration: A web search on "Jonathan Seys" returned a LinkedIn profile confirming US Navy affiliation and a NARA archive record (nara.getarchive.net) documenting his naval assignment. Cross-referencing with the PowerPoint content, which covers port operations, fleet deployment, and hurricane response, the authorship attribution lines up well with a naval officer.

**Second document - `Awnings Old vs New.doc`:**
- Owner field: **David White**
- Organization metadata linked to **Virginia Maritime Association**
- OSINT search on "David White Virginia Maritime Association" confirmed David White as Executive Director of the VMA, consistent with the document content around commercial maritime infrastructure

---

### 6. Extension Mismatch Detection

Autopsy's **Extension Mismatch Detected** module flagged `IMG_3062a.jpg` as suspicious.

**Analysis:**

The file was named as a JPEG image. Switching to the **Hex tab** in the content pane revealed the actual file header:

```
D0 CF 11 E0 A1 B1 1A E1
```

Cross-referencing against known magic bytes:
- **JPEG magic bytes:** `FF D8 FF`
- **Microsoft Compound Document (DOC) magic bytes:** `D0 CF 11 E0 A1 B1 1A E1` (match)

The file is actually a **.doc** Microsoft Word document with a `.jpg` extension.

**Data hiding assessment:**
The filename `IMG_3062a` mimics the naming convention of camera-generated images like `IMG_3062.jpg`. A `.doc` file containing text would not naturally end up with a camera-style image filename through any normal workflow. This points to intentional data hiding by renaming a document to blend into a photo library and avoid casual inspection.

The file was tagged **Follow Up**.

---

### 7. Keyword Hits and Military Email Addresses

The **Keyword Hits > Email Addresses** section surfaced multiple `.mil` email addresses associated with a single `.doc` file.

**Findings:**
- The `.doc` file contained content consistent with military authorship and distribution
- Multiple `.mil` addresses embedded in the file suggest broad internal military distribution
- Wide distribution makes single-person attribution harder; pinning down the original author would require access logs, server-side email headers, or document version history

---

### 8. Images/Videos Tool - Flagged Content

The **Images/Videos** tool in the top toolbar lets you visually browse all recovered image files. Toward the end of the collection (files `xlc_161` through `xlc_301`), a cluster of images was found containing:

- A passport photograph
- Images of drugs
- Images of firearms

These items are independently suspicious and potentially legally actionable regardless of the infrastructure-plot scenario. In a real investigation they would be escalated immediately and documented as separate evidentiary items.

---

### 9. Timeline Analysis

The **Timeline** tool reconstructed a chronological event sequence from filesystem timestamps and EXIF metadata.

**Latest photograph timestamp (from EXIF, not filesystem):**
```
2015:08:07 15:07:36
```

The distinction matters here. The filesystem timestamp reflects when the disk image was created this semester, while EXIF timestamps reflect when the camera actually fired. Relying on filesystem dates alone would collapse the entire timeline to the imaging date, which is useless forensically. Embedded metadata timestamps should always be prioritized when reconstructing the original sequence of events.

---

### 10. Report Generation

A full HTML report was generated via **Generate Report**:

- Format: HTML
- Header: "Assessment 1 Report"
- Footer: "CSIS 3700"
- Scope: All Results

The report was written to the `Reports` subfolder within the Autopsy case directory and serves as the permanent record of findings, suitable for disclosure in legal proceedings.

---

## Skills Demonstrated

- **Autopsy case management** - case creation, examiner documentation, data source ingestion, ingest module configuration
- **FAT32 filesystem analysis** - directory enumeration, file identification, volume structure
- **PDF internal structure differentiation** - text-layer vs. scanned (image-only) PDFs and the forensic implications of each
- **EXIF metadata extraction** - camera model, serial number, lens and focal length analysis
- **Magic byte / file header analysis** - hex-level verification of true file type vs. declared extension
- **Extension mismatch detection** - identifying potential data hiding via misnamed files
- **Document metadata attribution** - Owner vs. User ID fields, timestamp discrepancies, OSINT corroboration
- **Keyword search scoping** - understanding why scanned documents evade text-based search
- **Timeline reconstruction** - distinguishing filesystem timestamps from embedded EXIF timestamps
- **Evidence tagging and report generation** - chain of custody documentation within Autopsy

---

## References

- [Autopsy Digital Forensics Platform](https://www.autopsy.com/)
- [NIST SP 800-101r1 - Guidelines on Mobile Device Forensics](https://csrc.nist.gov/publications/detail/sp/800-101/rev-1/final)
- [FAT32 File System Specification](https://www.cs.fsu.edu/~cop4610t/assignments/project3/spec/fatspec.pdf)
- [JPEG Magic Bytes Reference - Gary Kessler's File Signatures Table](https://www.garykessler.net/library/file_sigs.html)
- [EXIF Standard - CIPA DC-008](https://www.cipa.jp/std/documents/e/DC-008-2012_E.pdf)
- [log2timeline / Plaso Documentation](https://plaso.readthedocs.io/)
