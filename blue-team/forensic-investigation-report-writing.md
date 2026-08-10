# Writing a Forensic Investigation Report: Chain of Custody to Evidentiary Conclusions

## Overview

A forensic analysis is only as useful as the report that documents it. This writeup covers how to produce a court-admissible forensic investigation report, including chain of custody documentation, multi-tool analysis methodology, objective findings, and defensible conclusions.

The scenario: a 2 GB USB drive submitted as the only piece of digital evidence in a case involving a suspected plot to destroy dams, bridges, and port infrastructure in the United States. The analyst's job is to report findings objectively, without stretching evidence toward guilt or ignoring legitimate indicators.

The report structure here follows the model used in NIST SP 800-101r1 and the NIJ Electronic Crime Scene Investigation guides, and reflects what a SOC analyst or DFIR professional would produce before legal handoff.

---

## Environment

| Component | Details |
|---|---|
| Evidence Item | 2 GB Red USB Drive, Win95 FAT32, General UDisk brand |
| Imaging Hardware | Tableau T8 Write Blocker (Firmware 18:03:02), Talon Ultimate Imager |
| Image Hash (SHA-256) | `32296F4B4F46C6C6CCA02991D20CF5BB779BDD11191C92F69798B76425C2DAC3` |
| Analysis Tools | Autopsy 4.19.1, Foremost 1.5.7, Bulk Extractor 1.5.5, BEViewer 1.6.0, PhotoRec 7.2 |
| Platform | CAINE Linux (forensic workstation) |
| Analyst | Emmanuel Guillen |

---

## Part 1 - Chain of Custody and Evidence Documentation

Before any analysis starts, every piece of evidence needs to be documented with enough detail that another examiner could independently verify the item and the process.

**Evidence item documented:**

| Field | Value |
|---|---|
| Item type | Suspect Removable Media |
| Description | 2 GB Red USB drive |
| Format | Win95 FAT32 |
| Brand | General UDisk |
| Suspect computers | N/A |
| Suspect OS | N/A |
| Paper documents | N/A |

**Why this matters in court:**
If evidence documentation is incomplete, defense counsel can challenge whether the item analyzed is the same item that was seized. Every "N/A" entry is meaningful too. It shows that a category was considered and ruled out, not skipped.

---

## Part 2 - Imaging Process Documentation

### Write Blocker Setup

The **Tableau T8** hardware write blocker was placed between the evidence drive and the imaging workstation before the drive was connected. This step is non-negotiable. Connecting evidence media to a live system without a write blocker can modify filesystem timestamps, MFT records, or last-access dates, which can destroy evidence and make the image inadmissible.

### Disk Imaging with Talon Ultimate

The **Talon Ultimate** imager was configured with the following settings:

```
Method:              dd
Unlock HPA:          No
Unlock DCO:          No
Error Granularity:   1 Sector
Hash Method:         SHA-256
Verify:              Yes (Primary)
Destination:         USB_D1
```

**Configuration rationale:**
- `dd` method produces a sector-by-sector forensic image including unallocated space, slack space, and deleted file remnants
- HPA and DCO unlocking left disabled since there was no evidence of hidden areas, and unlocking them would modify the drive state
- 1-sector error granularity maximizes recovery by isolating read errors to the smallest possible unit
- SHA-256 chosen over MD5 because it is stronger and resistant to collision attacks

**Hash verification:**

```
Image SHA-256: 32296F4B4F46C6C6CCA02991D20CF5BB779BDD11191C92F69798B76425C2DAC3
```

The Tableau T8 independently re-hashed the completed image. The hash matched exactly, which confirms the image is a bit-for-bit copy of the original evidence, no changes occurred during imaging, and the image is suitable for forensic analysis.

The image was saved as: `EmmanuelGuillen_DDCapture.001`

---

## Part 3 - Multi-Tool Analysis

A core principle of sound forensic methodology is that no single tool should be the sole basis for findings. Every tool has blind spots. Running multiple tools and comparing output builds stronger confidence in the results and surfaces artifacts that any one tool might miss.

---

### Tool 1: Autopsy 4.19.1

**Purpose:** Full forensic case management, covering filesystem browsing, deleted file recovery, metadata extraction, keyword search, timeline analysis, artifact tagging, and report generation.

**Configuration used:**
- Ingest modules: Plaso (timeline), File Type ID, EXIF Metadata, Extension Mismatch Detector, Keyword Search
- Hash Lookup, VM Extractor, and mobile analyzers: disabled

**Results:**
- **194 files** recovered from the FAT32 volume
- Filesystem type confirmed: Win95 FAT32
- Files of evidentiary interest:
  - `Hydropower notes.docx` - technical notes on hydroelectric infrastructure
  - `hydropower energy proj ppt.pptx` - hydropower project presentation
  - `NorfolkCityCouncil 072211.ppt` - city council presentation referencing port infrastructure
  - `Hurricane IRENE.ppt` - naval response planning document attributed to Jonathan Seys, US Navy
  - `IMG_3062a.jpg` - extension mismatch; actual file type is `.doc` (magic bytes `D0 CF 11 E0`)
  - Images `xlc_161` through `xlc_301` - passport image, drug images, firearm images
  - One file flagged by Autopsy as suspected encryption
- EXIF metadata extracted from image files; camera serial number `0770204950` obtained from Canon DSLR Maker Notes
- Document metadata: owner fields attributed to Jonathan Seys (US Navy) and David White (Virginia Maritime Association)

**Limitations:** Autopsy's keyword search cannot index scanned PDFs that have no text layer. The `HVAC Maintenance Agreement w SeaBoard.pdf` required manual review as a result.

---

### Tool 2: Foremost 1.5.7

**Purpose:** File carving by reconstructing files from raw disk sectors based on header and footer signatures, independent of filesystem structure.

**Commands executed:**

```bash
mkdir ~/foremost_output
foremost -i /home/student/Documents/DDCapture/DDCapture.001 -o ~/foremost_output
```

Foremost carves by scanning for known file signatures (magic bytes) across the entire image, including unallocated space and slack space where deleted files may partially persist.

**Results:**
- **2,690 files** recovered, significantly more than Autopsy's 194
- Additional files included images with embedded GPS coordinates in EXIF metadata, which could serve as geographic pivot points for further investigation
- No recovered files contained textual or visual evidence of a plot targeting bridges, roads, or dams

**Why Foremost recovered more files than Autopsy:**
Autopsy works from filesystem metadata and finds files the filesystem knows about, including deleted entries still in the directory table. Foremost ignores the filesystem entirely and carves directly from raw sectors, recovering fragments that the filesystem has already overwritten or deallocated. The two tools are complementary, not redundant.

---

### Tool 3: Bulk Extractor 1.5.5 and BEViewer 1.6.0

**Purpose:** High-speed extraction of structured data artifacts such as email addresses, URLs, phone numbers, credit card patterns, and GPS coordinates directly from raw disk sectors, with no filesystem parsing required.

**Commands executed:**

```bash
mkdir ~/BE_output
bulk_extractor -o ~/BE_output /home/student/Documents/DDCapture/DDCapture.001
```

BEViewer was used to browse the output feature files in a structured GUI rather than reading raw text output directly.

**Results:**
- Additional email addresses recovered beyond Autopsy's keyword hits
- Additional URLs (browsing history artifacts) recovered from unallocated space
- No recovered artifacts contained concrete evidence of infrastructure attack planning
- Multiple `.mil` email addresses identified, consistent with military document authorship

**Bulk Extractor advantage:**
Because it operates on raw bytes, Bulk Extractor recovers artifacts from deleted files, browser caches, swap space, and file slack that filesystem-aware tools will not surface. It is particularly effective at pulling email addresses and URLs from previously deleted browser artifacts.

---

### Tool 4: PhotoRec 7.2

**Purpose:** File recovery focused on media files, photos, videos, and documents, using signature-based carving optimized for consumer file formats.

**Command executed:**

```bash
sudo photorec /home/student/Documents/DDCapture/DDCapture.001
```

PhotoRec was run interactively, with the target partition and output directory selected during the process.

**Results:**
- Text files, photos, and videos recovered
- No recovered files contained content relevant to infrastructure destruction
- Some recovered files appeared to be graduation photographs based on context
- Some recovered content could not be attributed to a clear source

**Note on PhotoRec vs. Foremost:**
Both tools perform signature-based carving but differ in the formats they support and how they handle carving heuristics. Running both together provides broader signature coverage and cross-validation of results.

---

## Part 4 - Evidentiary Analysis

### Question 1: Is there textual evidence of a plot to destroy roads, bridges, or dams?

**Finding: No.**

No recovered text file, document, or keyword hit contained explicit or implied planning language related to destroying US infrastructure. The documents referencing hydropower, ports, and bridges were presentations advocating for hydropower policy or maritime infrastructure investment, naval emergency response planning documents, and city council meeting materials.

A movie-style script was recovered but contained nothing relevant to the case.

### Question 2: Is there photographic evidence of a plot?

**Finding: Possible context-dependent indicators, assessed as insufficient.**

Images recovered included road maps and infrastructure diagrams associated with the Virginia Maritime Association (consistent with legitimate organizational use), images of firearms for sale, a passport photograph, and images consistent with drug possession.

The infrastructure diagrams appeared in a professional presentation context. The firearms images, while potentially actionable under other legal theories, do not constitute evidence of infrastructure attack planning. No images depicted bridge structures, dam facilities, or explosive devices in a threatening context.

### Question 3: Is there any other evidence tying individuals to such a plot?

**Finding: No direct connection established.**

The firearms images could be argued as circumstantially relevant if additional evidence existed, but none was found. The GPS coordinate data recovered by Foremost could theoretically be used for geographic mapping of activity, but the coordinates were tied to personal or professional photographs rather than reconnaissance imagery.

---

## Part 5 - Conclusion

**Opinion of the analyst:**

The evidence on the submitted USB drive is not sufficient to support a conclusion that the drive's owner was involved in a plot to destroy bridges, roads, or hydroelectric dam installations in the United States.

**Supporting evidence for this conclusion:**

1. **`Hydropower notes.docx` and `hydropower energy proj ppt.pptx`** - content supports advocacy for hydroelectric energy development, not sabotage of existing infrastructure. Metadata owner and creation fields are internally consistent, suggesting the documents were created by the attributed individual for legitimate professional purposes.

2. **`NorfolkCityCouncil 072211.ppt`** - a city council presentation. Content involves port infrastructure in a planning and civic engagement context.

3. **`Hurricane IRENE.ppt`** - a naval emergency response document attributed to Jonathan Seys (US Navy, confirmed via OSINT). Content covers hurricane relief operations, fleet deployment, and damage assessment, consistent with standard naval planning and not with attack planning.

4. **Virginia Maritime Association connection** - multiple documents are linked via metadata to individuals associated with the VMA (David White, Executive Director). The VMA is a legitimate professional organization and the infrastructure-related content reflects its organizational mission.

5. **No explicit planning language** - no recovered file contained the vocabulary, communication patterns, or operational specificity that would be expected in actual attack planning.

**Items flagged for further investigation (not concluded as evidence of the plot):**
- `IMG_3062a.jpg` (actual `.doc` file) - extension mismatch consistent with data hiding; content warrants review but did not contain infrastructure attack material
- Images of firearms and suspected drugs (`xlc_161` through `xlc_301`) - potentially actionable under separate legal theories; outside the scope of this investigation
- File flagged for suspected encryption - contents unknown; requires further analysis with appropriate tools

**Final assessment:** The drive contains files of interest under other potential legal theories (firearms, suspected drugs, data hiding via extension mismatch), but nothing recovered points to a plot against US infrastructure. The flagged items should still be pursued, since the absence of evidence for one crime does not rule out evidence of another.

---

## Skills Demonstrated

- **Chain of custody documentation** - complete evidence intake, imaging process, and hash verification records
- **Multi-tool forensic methodology** - Autopsy, Foremost, Bulk Extractor, and PhotoRec used in parallel with documented rationale for each
- **Tool selection rationale** - filesystem-aware vs. carving-based vs. artifact-extraction approaches explained and compared
- **Objective forensic reporting** - findings separated from conclusions; conclusions tied to specific evidence
- **Evidentiary analysis** - distinguishing context-appropriate content from genuinely suspicious content
- **OSINT attribution corroboration** - document metadata ownership verified through open-source research
- **Professional report structure** - chain of custody through methodology through findings through conclusion, suitable for legal handoff
- **Scope discipline** - recognizing when findings fall outside the case scope while still flagging them appropriately

---

## References

- [NIST SP 800-101r1 - Guidelines on Mobile Device Forensics](https://csrc.nist.gov/publications/detail/sp/800-101/rev-1/final)
- [NIJ Electronic Crime Scene Investigation Guide (OJP 199408)](https://www.ojp.gov/pdffiles1/nij/199408.pdf)
- [Autopsy Digital Forensics Platform](https://www.autopsy.com/)
- [Bulk Extractor Documentation](https://github.com/simsong/bulk_extractor)
- [Foremost File Carver](http://foremost.sourceforge.net/)
- [PhotoRec / TestDisk](https://www.cgsecurity.org/wiki/PhotoRec)
- [The Sleuth Kit Documentation](https://www.sleuthkit.org/)
