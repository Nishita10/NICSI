# Day 24 — Digital Forensics Tools

> **Platform:** Kali Linux
>
> **Focus:** Digital Forensics and Evidence Analysis
>
> **Tools:** Autopsy, The Sleuth Kit, Volatility, ExifTool
>
> **Topics:** Disk Analysis, Memory Analysis, Metadata Examination

## 1. Day 24 Overview

Digital forensics is the discipline of identifying, preserving, examining, and interpreting digital evidence in a way that supports accurate, defensible conclusions about what happened on a system. Where the previous days in this course focused on finding and exploiting weaknesses, Day 24 shifts focus toward the investigative side of cybersecurity: understanding what an attacker (or any user) actually did, based on the traces left behind.

Forensic analysis differs from penetration testing in an important way — a penetration test actively probes and sometimes changes a system to test its defenses, while forensic analysis is built around **not** altering the evidence. Evidence preservation and integrity matter because forensic conclusions may be used in incident response decisions, internal investigations, or legal proceedings, and any doubt about whether the evidence was altered undermines the entire analysis.

Day 24 covered three core topics — **Disk Analysis**, **Memory Analysis**, and **Metadata Examination** — using four tools: Autopsy, The Sleuth Kit, Volatility, and ExifTool.

---

## 2. Learning Objectives

By the end of this day, I should be able to:

- Develop a foundational understanding of digital forensics.
- Understand disk analysis and file-system artifacts.
- Understand forensic disk images and why investigators work from copies.
- Understand deleted-file recovery concepts at a conceptual level.
- Understand metadata examination.
- Understand memory forensics and volatile evidence.
- Understand how processes and network connections appear in memory.
- Understand the purpose of Autopsy.
- Understand the purpose of The Sleuth Kit.
- Understand the purpose of Volatility.
- Understand the purpose of ExifTool.
- Understand the relationship between GUI-based and command-line forensic tools.
- Gain a foundational understanding of evidence integrity and chain of custody.

This day is about building foundational familiarity with forensic tools and workflows, not professional-level investigative expertise.

---

## 3. What Is Digital Forensics?

**Digital forensics** is the process of identifying, collecting, preserving, examining, analyzing, and reporting on digital evidence in a way that maintains its integrity and supports sound conclusions.

Key concepts:

- **Digital evidence** — any data that can support or refute a finding about system or user activity.
- **Evidence acquisition** — the process of obtaining evidence (e.g., imaging a disk, capturing memory).
- **Evidence preservation** — ensuring evidence is not altered after acquisition.
- **Evidence analysis** — examining the evidence to identify relevant artifacts.
- **Evidence interpretation** — drawing conclusions supported by the analyzed evidence.
- **Reporting** — documenting the process and findings clearly and reproducibly.

**Forensic lifecycle:**

```text
Identification
      ↓
Collection
      ↓
Preservation
      ↓
Examination
      ↓
Analysis
      ↓
Documentation
      ↓
Reporting
```

| Stage | Explanation |
|---|---|
| Identification | Recognizing what evidence exists and where. |
| Collection | Acquiring the evidence (e.g., disk image, memory dump). |
| Preservation | Protecting the evidence from modification. |
| Examination | Extracting relevant data from the evidence. |
| Analysis | Interpreting the extracted data in context. |
| Documentation | Recording every step taken for reproducibility. |
| Reporting | Communicating findings clearly to stakeholders. |

Throughout this lifecycle, the priority is preserving the integrity of the original evidence — analysis should be performed on copies wherever possible.

---

## 4. Digital Evidence

Digital evidence can take many forms:

- Files
- File systems
- Disk images
- Deleted files
- Logs
- Browser artifacts
- Metadata
- Memory
- Running processes
- Network connections
- User activity
- System configuration
- Timestamps

**Volatile evidence** — evidence that may disappear when a system is powered off or changes state:

- RAM contents
- Running processes
- Active network connections
- Logged-in users

**Non-volatile evidence** — evidence that generally persists after shutdown:

- Disk contents
- File systems
- Stored files
- Logs
- Metadata

This distinction is why memory forensics and disk forensics are treated as separate but complementary disciplines — volatile evidence must be captured *before* a system is powered off, or it's lost.

---

## 5. Evidence Integrity

Evidence integrity means being able to demonstrate that evidence has not changed since it was acquired.

- **Hashing** — computing a fixed-size fingerprint of data (e.g., SHA-256, or MD5 as a legacy/example algorithm) that changes if even a single bit is altered.
- **Evidence verification** — comparing hashes before and after analysis to confirm nothing changed.
- **Original evidence vs. working copies** — investigators keep the original untouched and work on verified copies.
- **Chain of custody** — a documented record of who handled the evidence, when, and what was done to it.

```text
Original Evidence
       ↓
Acquire / Image
       ↓
Calculate Hash
       ↓
Analyze Forensic Copy
       ↓
Verify Integrity
       ↓
Report Findings
```

A hash alone does not prove a complete chain of custody — it only demonstrates that a specific file or image has not changed since the hash was calculated. Chain of custody also requires documented handling procedures.

---

## 6. Disk Analysis

**Disk analysis** examines persistent storage media to reconstruct file activity, system configuration, and user behavior.

Relevant concepts:

- Storage media
- Partitions
- File systems
- Directories and files
- Deleted files
- File-system timestamps
- File metadata
- Hidden files
- Unallocated space
- File carving (recovering file content from raw, unstructured data)
- Forensic disk images

**Common file systems (foundational level):**

| File System | Typical Use |
|---|---|
| FAT/FAT32 | Older/removable media, simple structure |
| NTFS | Modern Windows systems |
| ext4 | Common Linux file system |

This section stays at a conceptual level rather than covering low-level file-system internals in depth.

---

## 7. Forensic Disk Images

A **forensic disk image** is a bit-for-bit copy of a storage device, capturing not just visible files but also deleted data, file-system structures, unallocated space, and other artifacts.

- Investigators analyze **copies**, not the original media, to avoid altering evidence.
- Image formats vary (raw/dd-style images, and container formats that also store metadata), but this document keeps discussion at a conceptual level.
- Hash verification (Section 5) confirms an image matches the original media at the time of acquisition.

Forensic images can reveal far more than a normal file browser would show, which is why disk analysis tools are built to interpret file systems at a structural level rather than relying on the operating system's own view of the files.

---

## 8. Autopsy

### What is Autopsy?

**Autopsy** is a graphical digital-forensics platform built on top of The Sleuth Kit. It provides an accessible interface for many forensic-analysis tasks that would otherwise require multiple separate command-line tools.

### Why is it used?

It's widely used because it consolidates case management, evidence ingestion, file-system analysis, keyword searching, timeline construction, and reporting into a single guided workflow — making it a common entry point for people learning digital forensics.

### Where does it fit in the workflow?

```text
Evidence Acquisition
      ↓
Autopsy (Case Setup & Analysis)
      ↓
Artifact Review
      ↓
Documentation & Reporting
```

### Important Features

- **Case management** — organizing evidence, notes, and findings under a single investigation.
- **Evidence ingestion** — importing disk images or other data sources.
- **Disk-image and file-system analysis** — browsing the structure of the acquired evidence.
- **Keyword searching** — locating relevant terms across the evidence.
- **Timeline analysis** — reconstructing activity based on timestamps.
- **Deleted-file analysis** — surfacing file-system entries for deleted files where recoverable.
- **Hash analysis** — comparing files against known-good/known-bad hash sets.
- **Web/browser artifacts** — extracting browser history and related data.
- **Reporting** — generating structured output of findings.

Autopsy is especially useful for people newer to forensics because it provides a visual, guided workflow instead of requiring memorization of many individual command-line tools.

---

## 9. Autopsy Workflow

```text
Create Case
    ↓
Add Data Source
    ↓
Process / Ingest Evidence
    ↓
Analyze Artifacts
    ↓
Search / Filter
    ↓
Examine Files
    ↓
Review Timeline
    ↓
Identify Evidence
    ↓
Document Findings
    ↓
Generate Report
```

| Stage | Explanation |
|---|---|
| Create Case | Set up a new investigation with case metadata. |
| Add Data Source | Point Autopsy at a disk image or other evidence. |
| Process / Ingest | Run ingest modules to extract artifacts automatically. |
| Analyze Artifacts | Review what the ingest process surfaced. |
| Search / Filter | Narrow down evidence using keywords or filters. |
| Examine Files | Inspect individual files/entries in detail. |
| Review Timeline | Reconstruct activity chronologically. |
| Identify Evidence | Flag items relevant to the investigation. |
| Document Findings | Record observations as they're made. |
| Generate Report | Produce a structured summary of the investigation. |

> **Note:** This describes the intended workflow. See Section 31 for what was actually practiced versus what remains conceptual for Day 24.

---

## 10. Autopsy Important Features (Detail)

- **Case Management** — a case is the organizational container for all evidence, notes, and findings related to a single investigation.
- **Data Sources** — disk images (or other supported evidence types) added to a case for analysis.
- **Ingest Modules** — automated processing steps that extract artifacts (e.g., file types, hashes, keywords) from a data source.
- **File Analysis** — manual inspection of individual files and directory structures.
- **Keyword Search** — searching evidence content for specific terms relevant to the investigation.
- **Timeline** — a chronological view built from file-system and other timestamps, useful for reconstructing a sequence of events.
- **Hash Analysis** — comparing files against known hash sets (e.g., to quickly identify known-good system files or known-bad malicious files).
- **Deleted Files** — Autopsy can surface file-system entries pointing to deleted files, though actual recoverability depends on whether the underlying data has been overwritten.
- **Web Artifacts** — browser history, cookies, and related data at a high level.
- **Reports** — structured output summarizing findings, useful for documentation and review.

---

## 11. The Sleuth Kit (TSK)

**The Sleuth Kit** is the open-source, command-line forensic toolkit/library that Autopsy is built on top of.

- **Autopsy** = primarily a graphical forensic platform, oriented toward guided case-based investigation.
- **The Sleuth Kit** = the underlying command-line toolkit/library providing the low-level file-system and disk-image analysis capabilities that Autopsy uses internally, and that can also be used directly.

They are not competing tools — TSK provides the analytical engine, and Autopsy provides a graphical workflow around it. Understanding TSK directly gives insight into what's happening "under the hood" when using Autopsy.

---

## 12. Important Sleuth Kit Tools

| Tool | Purpose | Evidence Examined | Basic Syntax | Typical Use |
|---|---|---|---|---|
| `mmls` | Displays partition/volume layout | Partition table | `mmls <DISK-IMAGE>` | Identifying where partitions start within an image |
| `fsstat` | Displays file-system details | File-system metadata | `fsstat <DISK-IMAGE>` | Understanding file-system type/parameters before deeper analysis |
| `fls` | Lists files and directory entries, including deleted ones | Directory structure | `fls <DISK-IMAGE>` | Enumerating files, including deleted file-system entries |
| `istat` | Displays metadata for a specific file-system object | Inode/MFT entry metadata | `istat <DISK-IMAGE> <META-ADDRESS>` | Examining detailed metadata (timestamps, size) for one file |
| `icat` | Extracts file content by metadata address | File content | `icat <DISK-IMAGE> <META-ADDRESS>` | Recovering the content of a specific file/entry |
| `blkls` | Lists/extracts data blocks, including unallocated space | Raw block-level data | `blkls <DISK-IMAGE>` | Examining unallocated space for potential file carving |
| `img_stat` | Displays basic information about an image file | Image container metadata | `img_stat <DISK-IMAGE>` | Confirming image type/format before analysis |

`<DISK-IMAGE>` and `<META-ADDRESS>` are placeholders — actual values depend entirely on the specific forensic image being analyzed and are not fabricated here.

---

## 13. Sleuth Kit Example Workflow

```text
Disk Image
    ↓
mmls
    ↓
Identify Partition Layout
    ↓
fsstat
    ↓
Identify File-System Information
    ↓
fls
    ↓
List Files / Deleted Entries
    ↓
istat
    ↓
Examine Metadata
    ↓
icat
    ↓
Extract File
```

| Step | Purpose |
|---|---|
| `mmls` | Determine where each partition begins, since later commands often need an offset. |
| `fsstat` | Confirm the file-system type and general parameters for the target partition. |
| `fls` | Get a listing of files and directories, including entries that indicate deletion. |
| `istat` | Inspect the detailed metadata (timestamps, allocation status) for a specific entry found via `fls`. |
| `icat` | Extract the actual content of a specific file/entry once its metadata address is known. |

Offsets, partition numbers, and metadata addresses are entirely dependent on the specific image being analyzed — none are invented here, since no specific image was analyzed for Day 24 (see Section 31).

---

## 14. Deleted File Analysis

**Deletion**, in most file systems, typically removes the reference to a file (its directory entry / metadata) rather than immediately erasing its actual data. This is why deleted files are sometimes recoverable — the underlying data may still be present in what's now marked as unallocated space, until it's overwritten by new data.

Key points:

- Deletion does not necessarily mean destruction.
- Recoverability depends heavily on whether the freed space has since been reused.
- **File carving** is the technique of searching unallocated space for recognizable file signatures/structures to recover data even without file-system metadata.
- Investigators should never assume a deleted file is automatically recoverable — it depends on time elapsed, disk activity, and file-system behavior.

---

## 15. File Timestamps

Common timestamp categories:

| Timestamp | Typical Meaning |
|---|---|
| Created | When the file was first created on that file system |
| Modified | When the file's content was last changed |
| Access | When the file was last read/opened |
| Changed | When the file's metadata (not necessarily content) was last changed |

Timestamp semantics differ between operating systems and file systems — for example, what "created" means can vary between NTFS and ext4, and some operations (like copying a file) can reset certain timestamps in ways that are easy to misinterpret.

Investigators use timestamps to build **timelines** of activity, but must account for:

- Timezone settings (which can shift apparent event ordering if misread).
- File-system-specific timestamp behavior.
- The general reliability/precision of timestamps on a given system.

---

## 16. Memory Analysis

**Memory forensics** examines the contents of RAM — volatile evidence that exists only while a system is running (or briefly afterward, in specific circumstances).

Relevant concepts:

- RAM contents
- Volatile evidence
- Memory dumps (a captured snapshot of RAM)
- Running processes
- Loaded modules
- Network connections active at capture time
- Command history, where recoverable from memory
- The concept of code injected into a running process
- Malware investigation using memory artifacts

Memory can reveal information that may never touch disk at all — for example, a process running entirely in memory, or network connections that were active only briefly — which is why memory forensics complements disk analysis rather than duplicating it.

---

## 17. Volatility

### What is Volatility?

**Volatility** is an open-source memory-forensics framework used to analyze memory dumps/images.

- Works against a captured memory image, not a live system.
- Organized around **plugins**, each performing a specific type of analysis (e.g., listing processes, listing network connections).
- Supports analysis of processes, loaded modules/DLLs, network connections, user/session information, and can assist in malware investigation.

Exact command names and available plugins depend on which version of Volatility is installed — this document uses modern Volatility 3 conventions.

---

## 18. Volatility 3

General syntax:

```bash
vol -f <MEMORY_IMAGE> <PLUGIN>
```

Where:

- `<MEMORY_IMAGE>` — the path to the captured memory dump being analyzed.
- `<PLUGIN>` — the specific analysis module being run (e.g., process listing, network connections).

No specific installed version or environment configuration is assumed here beyond this general syntax, since a specific setup wasn't confirmed for Day 24.

---

## 19. Important Volatility Concepts / Plugins

| Plugin | What It Attempts to Identify | Why an Investigator Would Use It |
|---|---|---|
| `windows.info` | Basic image/system information | Confirms OS version and image validity before deeper analysis |
| `windows.pslist` | Running processes at capture time | Identifies what was executing on the system |
| `windows.pstree` | Process parent/child relationships | Helps spot unusual process ancestry |
| `windows.pcmdline` | Command lines used to launch processes | Reveals how/why a process was started |
| `windows.netscan` | Network connections/sockets found in memory | Identifies active or recent network activity |
| `windows.dlllist` | Loaded DLLs/modules per process | Helps identify unexpected or suspicious modules |
| `windows.handles` | Open handles held by processes | Reveals what resources a process was accessing |
| `windows.getsids` | Security identifiers associated with processes | Helps understand privilege context of processes |
| `windows.malfind` | Regions of memory that resemble injected/hidden code | Flags candidates for deeper manual investigation |

> `malfind` output is a set of **candidates for investigation**, not automatic proof of malware — legitimate software can sometimes produce similar patterns, and confirmation requires further manual analysis.

No plugin output is fabricated here, since no specific memory image was analyzed for Day 24 (see Section 31).

---

## 20. Memory Forensics Workflow

```text
Memory Image
      ↓
Identify / Validate Image
      ↓
System Information
      ↓
Process Enumeration
      ↓
Process Tree
      ↓
Command-Line Analysis
      ↓
Network Connections
      ↓
Loaded Modules
      ↓
Suspicious Memory Analysis
      ↓
Correlate With Disk Evidence
      ↓
Document Findings
```

| Step | Explanation |
|---|---|
| Identify / Validate Image | Confirm the memory image is well-formed and matches an expected OS/version. |
| System Information | Establish baseline context (OS version, capture time if known). |
| Process Enumeration | List what was running at capture time. |
| Process Tree | Understand parent/child relationships between processes. |
| Command-Line Analysis | Review how processes were launched. |
| Network Connections | Identify active/recent network activity. |
| Loaded Modules | Review DLLs/modules for anomalies. |
| Suspicious Memory Analysis | Investigate candidates flagged by tools like `malfind`. |
| Correlate With Disk Evidence | Cross-reference memory findings with disk artifacts where available. |
| Document Findings | Record the analysis process and conclusions. |

---

## 21. Memory vs Disk Analysis

| Feature | Disk Forensics | Memory Forensics |
|---|---|---|
| Evidence source | Persistent storage (disk images) | Volatile RAM (memory dumps) |
| Volatility | Non-volatile — persists after shutdown | Volatile — lost on power-off unless captured |
| Persistence | High — data can remain long-term | Low — reflects only a point-in-time snapshot |
| Running processes | Not directly visible | Directly visible at capture time |
| Network connections | Only indirect traces (e.g., logs) | Can show active/recent connections directly |
| Deleted files | May be partially recoverable | Not applicable in the same sense |
| Malware artifacts | File-based indicators | In-memory/injected code indicators |
| Typical tools | Autopsy, The Sleuth Kit | Volatility |
| Major limitation | Data may be overwritten or absent if never written to disk | Only reflects the moment of capture; must be captured before shutdown |

Disk and memory forensics complement each other — some evidence (like an in-memory-only malicious process) may only appear in memory, while other evidence (like historical file activity) may only be reliably found on disk.

---

## 22. ExifTool

### What is ExifTool?

**ExifTool** is a widely used command-line tool for reading, writing, and editing metadata embedded in files.

- Supports a very wide range of file types: images, documents, audio, video, and more.
- For forensic purposes, Day 24 focuses on **reading/examining** metadata, not modifying it.
- Metadata can be a useful investigative clue because it often records information about how, when, and with what device or software a file was created or modified.

---

## 23. Metadata Examination

**Metadata** is data about data — information describing a file rather than its primary content.

Relevant categories:

- File metadata (file system-level: size, timestamps)
- Image metadata / EXIF (camera/device-specific)
- Creation/modification information
- Camera information (make, model, settings)
- Software information (what created/edited the file)
- GPS metadata (location, when present)
- File type / MIME type
- Other embedded metadata (e.g., document author fields)

Metadata can provide valuable investigative leads, but it can also be:

- Missing
- Modified
- Stripped
- Incorrect
- Deliberately manipulated

Because of this, metadata should be treated as **evidence requiring corroboration**, not absolute truth. A GPS tag or timestamp in metadata is a lead to investigate further, not a conclusion on its own.

---

## 24. ExifTool Basic Commands

Safe, read-only metadata examination examples:

```bash
exiftool image.jpg
```
Displays all readable metadata fields for the given image.

```bash
exiftool file.pdf
```
Displays metadata for a non-image file type, demonstrating ExifTool's broad format support.

```bash
exiftool -a -u -g1 image.jpg
```
Displays all metadata (`-a`), including unknown tags (`-u`), grouped by category (`-g1`) — useful for a more complete, organized view during an investigation.

> ExifTool also supports metadata **modification/removal**, which is a distinct capability from forensic examination. This document's Day 24 focus is analysis only — modification capabilities are not covered here, since altering metadata is not an appropriate forensic-analysis activity.

---

## 25. EXIF / Image Forensics

Useful forensic fields that may appear in image metadata:

- Make / Model (camera or device information)
- Date/time (creation or modification)
- GPS coordinates (if location services were enabled)
- Software (what program last processed the file)
- Orientation
- Resolution
- Camera settings (exposure, ISO, etc., where applicable)

Investigators can use this metadata to generate **leads** — for example, narrowing down what device likely produced an image, or establishing an approximate timeframe. However:

> Metadata alone should not be treated as conclusive proof. It should be correlated with other evidence.

---

## 26. Metadata Examination Workflow

```text
Acquire File
    ↓
Preserve Original
    ↓
Calculate Hash
    ↓
Read Metadata
    ↓
Identify Relevant Fields
    ↓
Correlate With Other Evidence
    ↓
Document Findings
```

The original file is preserved (and hashed) before analysis so that any later question about whether the file was altered can be answered definitively, and so metadata examination itself doesn't accidentally modify the evidence.

---

## 27. Autopsy vs Sleuth Kit

| Feature | Autopsy | The Sleuth Kit |
|---|---|---|
| Interface | Graphical | Command-line |
| Primary use | Guided, case-based investigation | Low-level, scriptable disk/file-system analysis |
| Disk image analysis | Yes (via TSK underneath) | Yes (native) |
| File-system analysis | Yes | Yes |
| Timeline analysis | Yes, built-in | Possible, but requires more manual tool combination |
| Automation | Ingest modules automate much of the process | Highly scriptable for custom workflows |
| Command-line usage | Limited (primarily GUI) | Core interface |
| Typical user | Beginners to intermediate investigators; case-driven work | Investigators needing fine-grained/scriptable control |

Autopsy and The Sleuth Kit are complementary, not competing — Autopsy is effectively a guided interface built on TSK's underlying engine.

---

## 28. Volatility vs Autopsy

| Feature | Autopsy | Volatility |
|---|---|---|
| Primary evidence | Disk images | Memory images |
| Disk analysis | Yes | No |
| Memory analysis | No (not its focus) | Yes |
| Main purpose | File-system/disk investigation | Volatile memory investigation |
| Example evidence | Files, deleted entries, browser artifacts | Processes, network connections, loaded modules |
| Typical investigation | Reconstructing file/user activity over time | Identifying what was running/active at capture time |

A thorough investigation often uses both — disk analysis for persistent history, memory analysis for what was happening at the moment of capture, especially for identifying malware that may never touch disk.

---

## 29. ExifTool vs Forensic Platforms

ExifTool differs from Autopsy/TSK/Volatility in scope and purpose:

- It's a **specialized** tool focused narrowly on metadata, not full disk or memory investigation.
- Its command-line simplicity makes it fast for quickly checking a single file's metadata.
- It **complements** the larger platforms — Autopsy can surface files of interest, and ExifTool can then be used (or Autopsy's own metadata features) to examine them in more detail.
- Having a dedicated, lightweight tool alongside larger forensic platforms is useful because not every investigation requires spinning up a full case in Autopsy just to check one file's metadata.

---

## 30. Complete Day 24 Forensic Workflow

```text
                Digital Evidence
                       ↓
              Evidence Preservation
                       ↓
              Hash / Integrity Check
                       ↓
             ┌─────────┴─────────┐
             ↓                   ↓
        Disk Evidence       Memory Evidence
             ↓                   ↓
        Autopsy/TSK           Volatility
             ↓                   ↓
       File-System /       Process / Network /
       Artifact Analysis   Memory Analysis
             ↓                   ↓
             └─────────┬─────────┘
                       ↓
               Metadata Analysis
                       ↓
                    ExifTool
                       ↓
              Correlation of Evidence
                       ↓
                 Findings & Report
```

This workflow shows how the four Day 24 tools fit together: evidence is first preserved and hashed, then disk and memory evidence are analyzed in parallel using the appropriate tools, individual files of interest are examined further with ExifTool for metadata, and all findings are ultimately correlated into a single, documented conclusion.

---

## 31. Practical Lab Section

## Lab Environment

- Platform: Kali Linux
- Tools: Autopsy, The Sleuth Kit, Volatility, ExifTool
- Evidence Source: `<SAMPLE-IMAGE>` / `<SAMPLE-MEMORY-DUMP>` *(not yet acquired)*
- Objective: Practice forensic analysis using intentionally created/sample evidence

## Practical Tasks

### Task 1 — Disk Analysis

**Objective:** Practice using Autopsy and The Sleuth Kit against a sample disk image.

**Commands / Procedure:** *(planned — see command reference in Section 32)*

**Observation:** **Concept Studied** — no disk image has been analyzed yet.

**Result:** > **Practical Evidence:** Add actual terminal output, screenshot, or forensic finding here.

### Task 2 — Memory Analysis

**Objective:** Practice using Volatility against a sample memory image.

**Commands / Procedure:** *(planned — see command reference in Section 32)*

**Observation:** **Concept Studied** — no memory dump has been analyzed yet.

**Result:** > **Practical Evidence:** Add actual terminal output, screenshot, or forensic finding here.

### Task 3 — Metadata Examination

**Objective:** Practice using ExifTool against sample images/documents.

**Commands / Procedure:** *(planned — see command reference in Section 32)*

**Observation:** **Concept Studied** — no sample files have been examined yet.

**Result:** > **Practical Evidence:** Add actual terminal output, screenshot, or forensic finding here.

**Status note:** Day 24 focused on building conceptual understanding of digital forensics and the four tools above. No actual disk image, memory dump, or file set was analyzed during this stage — all three tasks remain **Pending / Optional Practice** and are marked accordingly rather than presented as completed.

---

## 32. Command Reference

### Autopsy

Autopsy is primarily used through its **graphical interface** (Section 9's workflow: create case → add data source → ingest → analyze → report), rather than via direct command-line invocation, so no command syntax is listed here.

### The Sleuth Kit

```bash
mmls <DISK-IMAGE>
fsstat <DISK-IMAGE>
fls <DISK-IMAGE>
istat <DISK-IMAGE> <META-ADDRESS>
icat <DISK-IMAGE> <META-ADDRESS>
img_stat <DISK-IMAGE>
```

| Command | Purpose |
|---|---|
| `mmls` | Show partition layout of the image. |
| `fsstat` | Show file-system details for a given partition. |
| `fls` | List files/directories, including deleted entries. |
| `istat` | Show detailed metadata for a specific file-system object. |
| `icat` | Extract the content of a specific file/entry. |
| `img_stat` | Show basic information about the image file itself. |

`<META-ADDRESS>` values are never invented — they depend entirely on the actual image being analyzed.

### Volatility

```bash
vol -f <MEMORY-IMAGE> windows.info
vol -f <MEMORY-IMAGE> windows.pslist
vol -f <MEMORY-IMAGE> windows.pstree
vol -f <MEMORY-IMAGE> windows.pcmdline
vol -f <MEMORY-IMAGE> windows.netscan
```

| Command | Purpose |
|---|---|
| `windows.info` | Confirm image/system information. |
| `windows.pslist` | List running processes at capture time. |
| `windows.pstree` | Show process parent/child relationships. |
| `windows.pcmdline` | Show command lines used to launch processes. |
| `windows.netscan` | Show network connections found in memory. |

Exact plugin names/availability depend on the installed Volatility version.

### ExifTool

```bash
exiftool <FILE>
exiftool -a -u -g1 <FILE>
```

| Command | Purpose |
|---|---|
| `exiftool <FILE>` | Display readable metadata for a file. |
| `exiftool -a -u -g1 <FILE>` | Display all metadata, including unknown tags, grouped by category. |

---

## 33. Important Forensic Principles

### 1. Preserve the original
Never modify the original evidence directly.

### 2. Work on forensic copies
Perform analysis on verified copies wherever possible.

### 3. Verify integrity
Use hashes to confirm evidence hasn't changed.

### 4. Document every action
Keep a record detailed enough for someone else to reproduce the analysis.

### 5. Correlate evidence
Don't draw conclusions from a single artifact in isolation.

### 6. Consider timestamps carefully
Account for timezones and file-system-specific timestamp semantics.

### 7. Do not assume metadata is truthful
Metadata can be missing, incorrect, or deliberately altered.

### 8. Distinguish evidence from interpretation
A finding should be clearly supported by the evidence presented, not just asserted.

---

## 34. Common Beginner Mistakes

- Modifying original evidence instead of working from a copy.
- Forgetting to hash evidence before and after analysis.
- Misinterpreting timestamps (e.g., ignoring timezone differences).
- Treating metadata as absolute truth without corroboration.
- Assuming deleted files can always be recovered.
- Assuming every unusual process found via `pslist`/`pstree` is malicious.
- Treating `malfind` output as automatic proof of malware.
- Using an incorrect disk-image offset or metadata address.
- Using the wrong Volatility plugin for the installed version.
- Confusing disk forensics with memory forensics.
- Failing to correlate findings across multiple evidence sources.
- Failing to document investigative steps as they're performed.
- Ignoring evidence integrity procedures.

---

## 35. Limitations of Forensic Tools

Forensic tools are powerful but not infallible:

- Tool/version differences can affect available features or plugin compatibility.
- Unsupported or unusual file systems may not be fully parsed.
- Corrupted images can produce incomplete or misleading results.
- Metadata may simply be missing.
- Overwritten deleted data cannot generally be recovered.
- Incomplete memory captures limit what can be analyzed.
- False positives and false negatives are possible in automated analysis.
- Anti-forensic techniques (e.g., timestamp manipulation, secure deletion) can hinder investigation.
- Encryption can prevent access to evidence without proper keys/credentials.
- Sometimes the evidence is simply insufficient to reach a firm conclusion.

Because of these limitations, human interpretation and corroboration across multiple evidence sources remain essential — no tool output should be accepted uncritically.

---

## 36. Defensive / Investigative Perspective

### Autopsy / Sleuth Kit
Can help investigate: file activity, deleted files, user artifacts, timelines, and other disk-based evidence — useful for reconstructing what happened on a system over time.

### Volatility
Can help investigate: suspicious processes, network connections, loaded modules, memory-resident malware, and active sessions — useful for understanding what was happening at the moment of compromise or capture.

### ExifTool
Can help examine: file metadata, image provenance clues, GPS information, and software/device information — useful for establishing context around individual files of interest.

These tools directly support incident response and investigation by turning raw disk/memory/file artifacts into structured, reviewable evidence.

---

## 37. Day 24 and Incident Response

```text
Security Incident
       ↓
Identify Evidence
       ↓
Preserve Evidence
       ↓
Acquire
       ↓
Analyze Disk + Memory + Metadata
       ↓
Correlate Findings
       ↓
Determine What Happened
       ↓
Document
       ↓
Remediate
```

Digital forensics is a core component of incident response: once a security incident is suspected, evidence must be identified and preserved quickly, especially volatile memory evidence, and then analyzed using the disk, memory, and metadata techniques covered in this day to determine what actually happened before remediation steps are taken.

---

## 38. Ethics and Legal Considerations

- Only analyze systems/evidence I am explicitly authorized to examine.
- Do not access someone else's files without permission.
- Do not modify or destroy evidence.
- Preserve evidence integrity at every stage.
- Use intentionally created lab evidence for learning purposes.
- Respect privacy when handling any forensic artifacts.
- Do not publish sensitive forensic information (real personal data) to a public GitHub repository.

This matters particularly in forensics because evidence may contain personally identifiable information, and mishandling it — even accidentally, even in a learning context — can have real privacy consequences.

---

## 39. What I Learned

Day 24 gave me a foundational understanding of digital forensics as a discipline distinct from offensive security — one centered on preserving and interpreting evidence rather than actively probing or exploiting systems. I gained a foundational understanding of disk analysis, including how forensic disk images allow investigators to examine file systems, deleted entries, and metadata without touching the original media, and how tools like Autopsy and The Sleuth Kit relate to each other as a graphical platform built on a command-line toolkit.

I also became familiar with memory forensics as a way of capturing and analyzing volatile evidence — processes, network connections, and loaded modules that may exist only in RAM — and how Volatility organizes this analysis around plugins. Working through ExifTool helped me understand how file metadata can provide investigative leads, while also learning why metadata should never be treated as automatically trustworthy.

Overall, I developed a foundational understanding of how disk analysis, memory analysis, and metadata examination fit together as complementary techniques, and why evidence integrity, careful documentation, and correlation across sources matter throughout a forensic investigation.

---

## 40. Key Takeaways

- Digital forensics focuses on collecting, preserving, examining, and interpreting digital evidence.
- Disk analysis focuses on persistent storage evidence.
- Memory analysis focuses on volatile evidence.
- Metadata can provide useful investigative clues, but requires corroboration.
- Autopsy provides a graphical, case-based forensic-analysis workflow.
- The Sleuth Kit provides the underlying command-line forensic capabilities.
- Volatility is designed specifically for memory forensics.
- ExifTool is useful for fast, focused metadata examination.
- Deleted data may sometimes be recoverable, depending on whether it's been overwritten.
- Hashes help verify evidence integrity, but don't by themselves establish full chain of custody.
- Timestamps must be interpreted carefully, accounting for timezone and file-system semantics.
- Metadata should be corroborated with other evidence, not treated as conclusive on its own.
- Suspicious forensic findings (e.g., `malfind` results) require further analysis, not automatic assumptions.
- Documentation and evidence integrity are essential throughout the entire process.

---

## 41. Further Practice

These are **suggested exercises**, not completed Day 24 work.

### Exercise 1 — Autopsy
Using a legal sample forensic image, practice: adding evidence, ingesting artifacts, searching files, reviewing timestamps, and creating a timeline.

### Exercise 2 — Sleuth Kit
On a sample forensic image, practice: `mmls`, `fsstat`, `fls`, `istat`, `icat`.

### Exercise 3 — Volatility
Using a legal sample memory image, practice: `windows.info`, `windows.pslist`, `windows.pstree`, `windows.pcmdline`, `windows.netscan`.

### Exercise 4 — ExifTool
Using sample images and documents, examine: EXIF data, timestamps, software/device information, and GPS metadata (where present).

---

## 42. Final Day 24 Summary

Day 24 introduced three important forensic perspectives:

```text
Disk Analysis
     +
Memory Analysis
     +
Metadata Examination
     ↓
Digital Evidence Investigation
```

The role of each tool:

```text
Autopsy
   ↓
Graphical Forensic Investigation

The Sleuth Kit
   ↓
Command-Line Disk/File-System Analysis

Volatility
   ↓
Memory Forensics

ExifTool
   ↓
Metadata Examination
```

These tools complement one another rather than duplicating each other's functions. Effective forensic investigation requires evidence preservation, careful technical analysis, correlation of findings across multiple sources, and thorough documentation — principles that apply regardless of which specific tool is being used. Day 24 marks a deliberate shift from the offensive-security focus of earlier days toward the investigative discipline of digital forensics, and sets up a foundation for later incident-response-oriented material.