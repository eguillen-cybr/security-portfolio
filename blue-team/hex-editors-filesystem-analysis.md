# Hex Editors & Filesystem Analysis
### FAT16/NTFS Internals, Manual File Carving, MFT Resident Files, Disk Image Mounting


---

## Overview

Understanding how filesystems organize and store data at the raw byte level is a foundational skill in digital forensics. GUI tools like Autopsy and FTK abstract away the underlying structures — but when those tools fail, or when a unique artifact needs to be interpreted without tool support, the ability to read raw hex and navigate filesystem internals manually is what separates a capable analyst from one who can only click buttons.

This lab uses ActiveDisk DiskEditor and WxHexEditor within a Bodhi Linux VM to analyze two 256MB disk images — one formatted FAT16 and one NTFS. Tasks include parsing the BIOS Parameter Block, reading Master Boot Record partition entries, locating files by searching for magic byte signatures, manually extracting a JPEG by hex offset, analyzing NTFS Master File Table (MFT) records, locating resident files embedded directly in the MFT, and mounting a disk image via loopback device.

---

## Environment

| Component | Detail | Role |
|-----------|--------|------|
| Host OS | Bodhi Linux VM (VirtualBox) | Analysis environment |
| DiskEditor | ActiveDisk DiskEditor (LoftSoft) | Filesystem structure analysis and navigation |
| WxHexEditor | Open source hex editor | Manual byte-level carving and editing |
| Disk Image 1 | 256MB_FAT.dd | FAT16 formatted disk image |
| Disk Image 2 | 256MB_NTFS.dd | NTFS formatted disk image |
| Tools | losetup, fdisk, mount | Linux loopback mounting utilities |
| VM Credentials | osboxes.org / osboxes.org | Bodhi Linux VM login |

---

## Findings Summary

| Finding | Detail |
|---------|--------|
| FAT image volume label | NO NAME |
| FAT filesystem type | FAT16 |
| Total clusters (FAT image) | 61,991 clusters |
| Partition bootable? | No — Active partition flag = 0x6F (not 0x80) |
| JPEG magic byte start offset | 0x000270336 |
| JPEG magic byte end offset | 0x000275170 |
| JPEG extracted | Thumbnail successfully recovered via WxHexEditor |
| NTFS resident file content | "Emmanuel Guillen" — embedded directly in $MFT |
| Lincoln's Second Inaugural Address offset | 183,571,460 (decimal) |
| PDF version found | %PDF-1.6 |
| Loopback mount successful | /dev/loop0p1 mounted at /media/tmp |

---

## Walkthrough

### Part 1 — FAT16 Filesystem Analysis

#### BIOS Parameter Block (BPB)

The FAT16 disk image was loaded into ActiveDisk DiskEditor and the FAT Boot Sector was parsed. The BIOS Parameter Block (BPB) contains low-level metadata about the filesystem that is written at format time.

**Key BPB values extracted:**

```
OEM ID:                MSDOS5.0
Bytes per sector:      512
Sectors per cluster:   8
Number of FATs:        2
Root entries:          242
Total sectors (large): 496,027
Media descriptor:      0xF8 (fixed disk)
Sectors per FAT:       101
Sectors per track:     63
Number of heads:       255
File system:           FAT16
Volume label:          NO NAME
Signature (55 AA):     Present — valid boot sector
```

**Cluster count calculation:**

FAT16 does not directly store the total cluster count — it must be derived:

```
Total sectors:         496,027
Reserved sectors:      1
FAT sectors:           2 × 101 = 202
Root directory sectors: (242 × 32) / 512 = 15
Data sectors:          496,027 - 1 - 202 - 15 = 495,809
Sectors per cluster:   8
Total clusters:        495,809 / 8 = 61,976 ≈ 61,991 clusters
```

---

#### Master Boot Record — Partition Bootability

Switching to the Master Boot Record view in DiskEditor, the partition table entries were examined. Each partition entry contains an **Active Partition Flag** at offset 446 (0x1BE). A value of `0x80` indicates a bootable partition; any other value means it is not bootable.

```
Partition 1 Active partition flag: 0x6F  → NOT bootable (0x80 required)
Partition 2 Active partition flag: 0x69  → NOT bootable
Partition 3 Active partition flag: 0x73  → NOT bootable
```

None of the partitions on this image are configured as bootable.

---

#### Long Filename (LFN) Entry

FAT filesystems support long filenames by storing them in special LFN directory entries that precede the corresponding 8.3 format entry. Using the Navigate function in DiskEditor to jump to the Root Directory, the LFN entries for files with MP3 player images were located. The Unicode filename characters were visible in the rightmost columns of the DiskEditor hex view, confirming the LFN structure.

---

#### Gettysburg Address — Text Search

Using DiskEditor's Find function, the string `"Four score"` was searched across the disk image. The Gettysburg Address was located in the data area of the FAT volume. The text was stored in plain ASCII, readable directly in the hex editor's right-hand ASCII column — beginning at offset `0x004288512`.

---

#### Manual JPEG Extraction by Magic Bytes

Every JPEG file begins with the two-byte magic number `FF D8` (Start of Image marker) and ends with `FF D9` (End of Image marker). These are the signatures that file carving tools like Foremost use to locate JPEG files regardless of filesystem metadata.

**Locating the JPEG:**

```
Search for: FF D8  (hex)
Start offset found: 0x000270336

Search for: FF D9  (hex — from current position)
End offset found:   0x000275170
```

**Manual extraction process:**

```
1. In DiskEditor: selected all bytes from 0x000270336 to 0x000275170
2. Copied selected bytes to clipboard (Ctrl+C)
3. Opened WxHexEditor
4. Created new 1MB (1,000,000 byte) file named test.jpg
5. Clicked first position, pasted clipboard contents
6. Saved file
7. Opened Thunar file manager → thumbnail rendered successfully
```

The extracted file was a JPEG thumbnail of a mountain lion (cougar). This confirms that JPEG files can be manually recovered from raw disk images using only magic byte offsets — no filesystem metadata required. This is the underlying technique used by all automated file carving tools.

> **Why this matters:** When a suspect deletes a file, the filesystem entry is removed but the raw bytes remain on disk until overwritten. As long as the start (FF D8) and end (FF D9) markers are intact, the file can be recovered manually or automatically. Understanding this at the byte level explains why file carving works and what its limitations are.

---

### Part 2 — NTFS Filesystem Analysis

#### MFT Resident Files

NTFS uses the Master File Table ($MFT) to store metadata about every file on the volume. For very small files — typically under ~700 bytes — NTFS stores the actual file content directly inside the MFT record rather than allocating separate data clusters. These are called **resident files**.

Using DiskEditor on the 256MB_NTFS.dd image, the $MFT was examined. A resident file was located containing the text `"Emmanuel Guillen"` — a small text file whose content was embedded directly in the MFT entry rather than stored in separate sectors.

**Resident file MFT record fields:**

```
$FILE_NAME offset:   152
File name length:    14
File name namespace: 0
$DATA attribute:
  Non-resident flag: 0 (resident)
  Data embedded at:  offset 392 within MFT record
  Content:           45 6D 6D 61 6E 75 65 6C  → "Emmanuel"
                     20 47 75 69 6C 6C 65 6E  → " Guillen"
```

This confirms that small files do not necessarily occupy their own disk sectors — their content lives inside the MFT itself. This has forensic implications: even if the data sectors of a volume are wiped, resident file content may survive within the MFT area.

---

#### Lincoln's Second Inaugural Address

Using DiskEditor's Find function, the string `"With malice toward none"` was searched on the NTFS image. The text file was located at hex offset **183,571,460** (decimal), stored in a non-resident $DATA attribute pointing to allocated clusters on the volume.

---

#### PDF Magic Bytes

PDFs begin with the signature `%PDF-` followed by the version number. Searching for `%PDF-` in hex (`25 50 44 46 2D`) on the NTFS image:

```
Search string: %PDF-
Match found:   %PDF-1.6
```

The file uses PDF specification version 1.6, which was introduced in Adobe Acrobat 7 (2005). The internal PDF structure — cross-reference tables, object streams, compressed content — was visible in the raw hex view.

---

### Part 3 — Loopback Mounting

Linux allows disk image files to be mounted as virtual block devices using the loopback subsystem. This is how forensic analysts can access the filesystem of a disk image without writing to it or requiring physical hardware.

**Commands executed:**

```bash
sudo -s
losetup -f 256MB_NTFS.dd          # attach image to first available loop device
losetup -a                          # verify: /dev/loop0 -> 256MB_NTFS.dd
fdisk -l /dev/loop0                 # list partitions: /dev/loop0p1
mount /dev/loop0p1 /media/tmp       # mount NTFS partition
ls /media/tmp                       # list filesystem contents
```

**Files visible after mounting:**

```
1920x1080-Cougar2.jpg
1940 Statement.pdf
1-Catalog-Home.doc
450px-Sony_MP3_player.jpg
799px-Medion_MP3_player-0285.jpg
An_mp3-player_(ubt_147).jpeg
bensound-ukulele.mp3
csc-001y-index-to-catalog-and-supplements-twc.docx
FRFforSMEs_SampleFinancialStatements_FSUsers.xls
gettysburg_address.txt
Lincoln's Second Inaugural Address, March 4, 1865.html
Second_Inaugural.txt
small_text.txt
UC24 user manual.pdf
```

A new file was created at `/media/tmp/emmanuel_guillen.txt`, the partition was unmounted, and the loopback device was detached:

```bash
echo "Emmanuel Guillen - test file" > /media/tmp/emmanuel_guillen.txt
umount /media/tmp
losetup -d /dev/loop0
```

Reopening the image in DiskEditor confirmed the newly created file existed as a resident file in the $MFT, with the text content visible directly in the MFT record bytes.

---

## Skills Demonstrated

- **FAT16 filesystem internals** — BPB parsing, cluster count calculation, LFN directory entries, root directory structure
- **NTFS filesystem internals** — MFT record analysis, resident vs. non-resident file identification, $FILE_NAME and $DATA attribute parsing
- **MBR partition table analysis** — active partition flag identification, bootability determination
- **Manual JPEG file carving** — magic byte identification (FF D8 / FF D9), hex offset extraction, manual reconstruction in WxHexEditor
- **Magic byte awareness** — PDF (%PDF-), JPEG (FF D8 FF E0/FF D9) signature identification and search
- **Hex editor proficiency** — ActiveDisk DiskEditor navigation, WxHexEditor file creation and paste operations
- **Linux loopback mounting** — `losetup`, `fdisk`, `mount`, `umount` for forensic image access without modification
- **Filesystem forensic reasoning** — explaining why resident files survive data sector wipes, why deleted files are recoverable until overwritten

---

## Framework Mapping

| Framework | Control | Application |
|-----------|---------|-------------|
| NIST SP 800-86 | Guide to Integrating Forensic Techniques | Filesystem analysis methodology, evidence preservation |
| NIST SP 800-101 Rev 1 | Mobile Device Forensics | File system structure fundamentals applicable across platforms |
| SWGDE Best Practices | Digital Evidence | Manual verification of carving results; understanding tool limitations |
| SANS FOR500 | Windows Forensic Analysis | NTFS MFT analysis, resident file identification, $DATA attributes |

---

## Key Concepts

**Why FAT format doesn't erase data:** When FAT formats a volume, it rewrites the File Allocation Table and root directory entries but does not zero out the data sectors. The raw bytes of every file remain in place until new data overwrites those specific sectors. This is why Foremost and Photorec can recover files from a freshly formatted FAT drive.

**Why NTFS MFT resident files are forensically significant:** Even if an adversary runs a data-wiping tool against the data sectors of an NTFS volume, files small enough to be resident in the MFT may survive intact, because the MFT occupies its own reserved zone on the disk that many wiping tools skip or handle differently.

**Magic bytes as a carving foundation:** File carving tools do not rely on filesystem metadata — they scan raw bytes looking for known signatures. A JPEG can be carved from unallocated space, RAM dumps, swap files, or even network captures as long as its FF D8 header and FF D9 footer are present and intact.

---

## References

- NIST SP 800-86: Guide to Integrating Forensic Techniques — https://csrc.nist.gov/publications/detail/sp/800-86/final
- Microsoft NTFS Documentation — https://docs.microsoft.com/en-us/windows/win32/fileio/master-file-table
- FAT Filesystem Specification — https://www.microsoft.com/en-us/download/details.aspx?id=55302
- File Signatures / Magic Bytes Reference — https://www.garykessler.net/library/file_sigs.html
- ActiveDisk DiskEditor — https://www.disk-editor.org/
- SANS Windows Forensic Analysis (FOR500) — https://www.sans.org/cyber-security-courses/windows-forensic-analysis/
