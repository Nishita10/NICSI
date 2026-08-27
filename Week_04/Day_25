# Day 25 — File Recovery & Firmware Analysis

> **Platform:** Kali Linux
>
> **Focus:** File Recovery and Firmware Analysis
>
> **Tools:** Foremost, Scalpel, Binwalk
>
> **Topics:** File Recovery, Firmware Analysis

## 1. Day 25 Overview

Day 25 continues the digital forensics portion of this learning path, moving from Day 24's file-system, memory, and metadata analysis into extracting useful information from raw or structured binary data when conventional file browsing isn't sufficient. Two areas were covered:

**File Recovery** — recovering files based on their identifiable data structures/signatures rather than relying only on normal directory entries, using **Foremost** and **Scalpel**.

**Firmware Analysis** — examining firmware images from embedded/network/IoT devices to understand their internal structure and identify potentially relevant components, using **Binwalk**.

Both areas share a common theme: working directly with raw data structures rather than trusting the file system's own bookkeeping.

---

## 2. Learning Objectives

By the end of this day, I should be able to:

- Develop a foundational understanding of file recovery concepts.
- Understand file carving as a technique distinct from file-system-based recovery.
- Understand file signatures/magic numbers.
- Understand the difference between file-system recovery and file carving.
- Understand what Foremost is and how it performs signature-based recovery.
- Understand what Scalpel is and how its configuration-driven approach differs from Foremost's.
- Understand how recovered files should be validated rather than trusted outright.
- Understand the limitations of file carving.
- Understand firmware at a high level and how it differs from general-purpose software.
- Understand what a firmware image typically contains.
- Understand what Binwalk is and how it performs signature scanning and extraction.
- Understand embedded file systems at a foundational level.
- Understand why firmware analysis matters for IoT and embedded-device security.
- Understand evidence integrity and documentation practices as they apply to recovered/extracted data.

This document reflects a foundational understanding of file-carving and firmware-analysis techniques, not advanced reverse-engineering expertise.

---

## 3. File Recovery Fundamentals

**File recovery**, in a digital forensics context, is the process of retrieving files that are no longer accessible through normal file-system browsing.

Relevant concepts:

- **Deleted files** — files whose file-system reference has been removed or marked available.
- **File-system references** — the metadata entries (directory entries, inode/MFT records) that normally point to a file's data.
- **Unallocated space** — disk space marked as available for reuse, which may still contain old file data.
- **Raw data** — data considered without reference to file-system structure.
- **File signatures** — identifiable byte patterns marking the start (and sometimes end) of a known file type.
- **File carving** — recovering files based on signatures rather than file-system metadata (see Section 4).
- **Data fragmentation** — a file's data being split into non-contiguous pieces on disk, which complicates carving.
- **Overwritten data** — data that has been replaced by new writes and is generally unrecoverable.

> A deleted file is not necessarily immediately destroyed. When a file is deleted, the file-system reference may be removed or marked as available while portions of the underlying data can remain until overwritten — but recovery is never guaranteed.

---

## 4. File Carving

### What is file carving?

**File carving** is the process of recovering files from raw data based primarily on known file structures/signatures, rather than relying entirely on file-system metadata. This makes it useful in scenarios where file-system metadata is missing, corrupted, or was never present (e.g., analyzing raw unallocated space).

**General workflow:**

```text
Raw Data / Disk Image
        ↓
Identify File Signature
        ↓
Locate File Header
        ↓
Identify File Structure / Footer
        ↓
Extract Data
        ↓
Reconstruct File
        ↓
Validate Recovered File
```

Key terms:

| Term | Meaning |
|---|---|
| **File header** | The bytes at the start of a file that identify its type/structure. |
| **File footer** | The bytes marking the end of a file, when the format defines one. |
| **Magic number** | A specific byte sequence used to identify a file format. |
| **File signature** | The general concept encompassing headers/magic numbers used for identification. |
| **File structure** | The internal organization a format follows between header and footer. |
| **File carving** | The overall technique of extracting files based on these identifiable structures. |

---

## 5. File Signatures / Magic Numbers

File signatures let investigators (and tools) identify a file's actual type independent of its name or extension.

Examples of well-known signatures (illustrative, not exhaustive):

| Format | Typical Header Signature |
|---|---|
| JPEG | Starts with bytes commonly represented as `FF D8 FF` |
| PNG | Starts with a fixed 8-byte PNG signature |
| PDF | Starts with the ASCII characters `%PDF` |
| ZIP | Starts with bytes commonly represented as `PK` (local file header) |

**File extensions are not sufficient to determine actual file type.** For example:

```text
filename.jpg
```

does not guarantee that the underlying bytes represent a valid JPEG file — the extension is just a naming convention and can be wrong, missing, or deliberately misleading.

This is why forensic investigators inspect the actual file signature rather than trusting the extension. It's also worth noting that some formats don't have simple fixed signatures, or have complex internal structures (containers holding multiple embedded formats), which makes carving them more difficult than simpler formats.

---

## 6. File Recovery vs. File Carving

| Feature | File-System Recovery | File Carving |
|---|---|---|
| Depends on file-system metadata | Yes — relies on remaining directory/inode entries | No — relies on file signatures/structure |
| Uses file signatures | Not primarily | Yes — this is the core technique |
| Works with unallocated space | Limited (metadata may already be gone) | Yes — designed for this scenario |
| Can recover fragmented files | Often more reliably, if metadata is intact | Difficult — carving struggles with fragmentation |
| Metadata preservation | Better — original filenames/timestamps often retained | Poor — original filenames/timestamps are usually lost |
| Typical limitation | Fails once metadata itself is gone | Prone to false positives/incomplete files without metadata to guide it |

These are related but distinct techniques — file-system recovery works from remaining structure, while carving works from content patterns when structure is unavailable.

---

## 7. Foremost

### What is Foremost?

**Foremost** is an open-source file-carving tool that scans input data for known file signatures and attempts to recover matching files.

- Purpose: recovering files based on signature matching, independent of file-system metadata.
- Commonly used in forensic investigations where deleted or fragmented file recovery is needed.
- Useful because it automates signature scanning across large raw data sources (e.g., disk images) rather than requiring manual byte-level searching.

---

## 8. Foremost Basic Usage

```bash
foremost -i <FORENSIC-IMAGE> -o <OUTPUT_DIRECTORY>
```

| Option | Meaning |
|---|---|
| `-i` | Input file — the raw data or disk image to scan. |
| `-o` | Output directory — where recovered candidate files are written. |

`<FORENSIC-IMAGE>` is a placeholder — it should always refer to an authorized forensic image, sample evidence, or personally owned data, never someone else's real device without permission. The output directory should itself be treated as evidence-derived material and documented (what tool/version produced it, when, from what input).

---

## 9. Foremost Important Options

```bash
foremost -h
```

Displays available options. Relevant categories for beginner forensic work:

- **Input** (`-i`) — specifying the data source to scan.
- **Output** (`-o`) — specifying where recovered files are written.
- **File-type selection** (e.g., restricting the scan to specific formats like JPEG or PDF) — narrows the scan and can reduce false positives.
- **Verbose mode** — provides more detail about what Foremost is doing during the scan, useful for understanding its process.

This document focuses on these core options rather than an exhaustive listing of every flag.

---

## 10. Foremost Workflow

```text
Forensic Image
      ↓
Foremost
      ↓
Signature Scanning
      ↓
Candidate Files
      ↓
Recovered Files
      ↓
Validation
      ↓
Hash / Documentation
```

| Stage | Explanation |
|---|---|
| Forensic Image | The raw input data being analyzed. |
| Foremost | The tool scans the input for known signatures. |
| Signature Scanning | Foremost searches for header/footer patterns matching known file types. |
| Candidate Files | Byte ranges matching a signature are treated as candidates. |
| Recovered Files | Candidates are written out as extracted files. |
| Validation | Each recovered file must be checked — is it actually valid/complete? |
| Hash / Documentation | Recovered files should be hashed and documented like any other evidence. |

A recovered file is a **candidate piece of evidence**, not automatically confirmed, valid data — it must be validated and documented like any other artifact.

---

## 11. Scalpel

### What is Scalpel?

**Scalpel** is another open-source file-carving tool, distinguished by its **configuration-driven** approach — rather than relying on built-in signature handling for a fixed set of types, Scalpel reads a configuration file that defines which headers/footers to search for.

- Purpose: signature-based file carving, similar in concept to Foremost.
- Key difference: carving behavior is explicitly controlled through a configuration file, giving the investigator more control over which formats are searched for and how.
- Useful when an investigator wants to tune carving to specific expected file types rather than scanning for everything by default.

---

## 12. Scalpel Configuration

Scalpel's configuration file defines, per file type:

- **File type** — a label for the format being searched for.
- **Header** — the byte pattern marking the start of the format.
- **Footer** — the byte pattern marking the end, if the format defines one.
- **Case sensitivity** — whether pattern matching should be case-sensitive, where applicable.
- **Maximum file size** — an upper bound to prevent runaway extraction if a footer is never found.

**Illustrative configuration example** (not derived from any specific real system's configuration):

```text
# type       case  header       footer       max-size
jpg          y     \xff\xd8\xff \xff\xd9     5000000
```

This line illustrates the general shape of a Scalpel configuration entry, not a verified working configuration for any particular Scalpel installation — exact syntax and supported options depend on the installed version.

---

## 13. Scalpel Basic Workflow

```text
Input Evidence
      ↓
Scalpel Configuration
      ↓
Signature Matching
      ↓
Candidate Data
      ↓
File Extraction
      ↓
Validation
      ↓
Documentation
```

```bash
scalpel -h
scalpel <FORENSIC-IMAGE> -o <OUTPUT-DIRECTORY>
```

An investigator might choose Scalpel over Foremost when more controlled carving is desirable — for example, when only specific file types are relevant to the investigation, and reducing noise/false positives from unrelated signatures is a priority.

---

## 14. Foremost vs. Scalpel

| Feature | Foremost | Scalpel |
|---|---|---|
| Primary purpose | Signature-based file carving | Signature-based file carving |
| File carving | Yes | Yes |
| Signature-based recovery | Yes, built-in signature handling | Yes, via configuration file |
| Configuration | Limited built-in options | Explicit, editable configuration file |
| Ease of use | Generally simpler to start with | Requires understanding/editing configuration |
| Customization | Lower | Higher — precise control over targeted formats |
| Typical use | Quick, general-purpose carving | Targeted carving when specific formats/control matter |
| Limitations | Less granular control over signatures | Requires more setup/understanding to use effectively |

Neither tool is universally superior — the appropriate choice depends on the investigation's needs and the evidence at hand.

---

## 15. Limitations of File Carving

File carving is powerful but has significant limitations:

- **Fragmented files** — carving struggles when a file's data isn't contiguous.
- **Overwritten data** — carving cannot recover data that's been overwritten.
- **Corrupted files** — partial or damaged data can produce invalid recovered files.
- **Encrypted files** — encrypted content won't carve into anything meaningful without decryption.
- **Unsupported file types** — carving only works for formats the tool/configuration knows about.
- **False positives** — byte patterns can coincidentally resemble a signature without being a real file of that type.
- **False negatives** — some valid files may be missed if their structure doesn't match expected patterns.
- **File structure complexity** — some formats are too complex for simple header/footer carving to reliably reconstruct.
- **Metadata loss** — carving typically loses the original filename, path, ownership, permissions, and file-system timestamps.

This metadata loss matters forensically — a carved file might contain valid content but without its original context, an investigator often can't determine exactly when, where, or by whom it was created without corroborating evidence.

---

## 16. Validating Recovered Files

Successful recovery of data does not automatically mean successful forensic reconstruction — recovered files must be validated.

```text
Recovered File
      ↓
Validate File Type
      ↓
Check Integrity
      ↓
Calculate Hash
      ↓
Correlate With Evidence
      ↓
Document
```

Validation steps:

- **File integrity** — confirming the file isn't truncated or corrupted.
- **File-type verification** — confirming the recovered data actually matches the expected format (not just a partial signature match).
- **Opening/testing in a controlled environment** — carefully inspecting the file, ideally in an isolated analysis environment given that recovered files are untrusted data.
- **Comparing expected file structure** — checking whether the internal structure is consistent with a valid file of that type.
- **Correlating with other evidence** — cross-referencing the recovered file against other findings.

> Opening unknown recovered files is not automatically safe. Suspicious recovered files should be handled in an isolated analysis environment rather than opened casually on a primary working system.

---

## 17. Firmware Fundamentals

**Firmware** is low-level software embedded in a device, typically responsible for controlling hardware and providing core functionality — distinct from general-purpose application software running on a full operating system, though many devices' firmware does include a full embedded OS.

Common firmware-bearing devices:

- Embedded systems generally
- IoT devices
- Routers
- Network appliances
- Cameras
- Industrial devices
- Consumer electronics

Firmware often contains:

- Boot components
- Operating-system components
- Configuration files
- Executables
- Libraries
- Web interfaces (for device management)
- Scripts
- File systems

---

## 18. Why Firmware Analysis Matters

Security researchers analyze firmware for several authorized purposes:

- **Vulnerability research** — identifying weaknesses in device software.
- **Embedded-device security** — understanding the overall security posture of IoT/embedded systems.
- **Supply-chain security** — verifying components and dependencies used in a device.
- **Malware analysis** — investigating firmware suspected of being tampered with.
- **Configuration analysis** — reviewing how a device is set up by default.
- **Hardcoded secrets** — identifying embedded credentials, keys, or tokens that shouldn't be present.
- **Default credentials** — a common and significant weakness in embedded devices.
- **Outdated software** — identifying old library/component versions with known issues.
- **Exposed services** — identifying network services running unnecessarily.
- **Weak security configurations** — general hardening gaps.

These are things a researcher may look for during **authorized** firmware-analysis work — this document does not provide a procedure for exploiting a real router, camera, or IoT device.

---

## 19. Firmware Image Structure

```text
Firmware Image
      ↓
Bootloader
      ↓
Kernel / OS Components
      ↓
Root File System
      ↓
Applications
      ↓
Libraries
      ↓
Configuration
```

| Component | Role |
|---|---|
| Bootloader | Initializes hardware and loads the next stage (kernel). |
| Kernel / OS Components | Provides the core operating environment. |
| Root File System | Contains the device's files — utilities, configuration, web interface, etc. |
| Applications | Device-specific functionality built on top of the OS. |
| Libraries | Shared code used by applications/services. |
| Configuration | Settings controlling device behavior. |

Actual firmware layouts vary greatly between vendors and devices — this diagram represents a common conceptual structure, not a universal standard.

---

## 20. Binwalk

### What is Binwalk?

**Binwalk** is a tool for analyzing binary files — most commonly firmware images — to identify embedded file types, file systems, compression formats, and other structures.

Capabilities:

- **Signature identification** — scanning a binary for known byte patterns indicating embedded content.
- **Embedded file detection** — identifying files or file systems packed inside a larger binary.
- **Extraction** — pulling identified components out into separate files/directories.
- **Entropy analysis** — measuring randomness across the binary, which can help identify compressed or encrypted regions even without a recognized signature.

Binwalk is useful for embedded-device research because firmware images are often opaque binary blobs — Binwalk provides a starting point for understanding what's actually inside one.

---

## 21. Binwalk Basic Usage

```bash
binwalk <FIRMWARE>
```

Scans the firmware file and reports identified signatures/structures at their byte offsets — a read-only analysis step.

```bash
binwalk -e <FIRMWARE>
```

| Element | Meaning |
|---|---|
| `-e` | Extract identified components into a directory alongside the analysis. |

Why extraction is useful: it turns identified structures (e.g., a file system) into something that can be directly examined with normal file tools, rather than remaining an opaque region within the larger binary.

Why extracted content should be treated as untrusted: it originates from a binary of unknown/unverified provenance and could contain malformed or malicious data — extracted content should be examined in an isolated environment rather than executed or opened carelessly.

`<FIRMWARE>` refers to a legally obtained sample firmware image, never a real vendor device's firmware obtained without authorization.

---

## 22. Binwalk Signature Analysis

Binwalk's signature scan can identify things such as:

- Compression formats (e.g., gzip, LZMA)
- File systems (e.g., SquashFS, JFFS2)
- Executables (e.g., ELF binaries)
- Archives
- Embedded data blobs
- Boot-related structures

> Signature detection does not automatically prove that the identified structure is valid or exploitable — it only indicates that a matching byte pattern was found. Further examination is required to confirm and understand what was actually detected.

---

## 23. Binwalk Extraction

```text
Firmware Image
      ↓
Signature Detection
      ↓
Embedded File-System Detection
      ↓
Extraction
      ↓
Extracted Directory
      ↓
Manual Analysis
```

Extraction can reveal an embedded file system, commonly one of:

- **SquashFS**
- **CramFS**
- **JFFS2**
- **UBI/UBIFS**

These are introduced at a foundational level in Section 24 — this document does not go deeply into kernel/flash internals.

---

## 24. Firmware File Systems

Embedded Linux devices commonly use specialized file systems suited to constrained storage (often raw flash memory rather than a conventional disk):

### SquashFS
A compressed, read-only file system commonly encountered in embedded firmware — well-suited for a static root file system that doesn't need to be writable.

### JFFS2
A flash-oriented file system designed to work directly with raw NAND/NOR flash, including wear-leveling considerations.

### UBIFS
A flash-oriented file system often associated with raw NAND storage, typically used alongside UBI (Unsorted Block Images) for flash management.

### CramFS
Another compressed, read-only file system, generally simpler and older than SquashFS.

Not every firmware image contains these specific file systems — the actual file system depends entirely on the vendor and device.

---

## 25. Firmware Analysis Workflow

```text
Obtain Authorized Firmware
          ↓
Preserve Original
          ↓
Calculate Hash
          ↓
Binwalk Signature Scan
          ↓
Identify Embedded Components
          ↓
Extract Components
          ↓
Identify File Systems
          ↓
Examine Files / Binaries
          ↓
Search for Security-Relevant Artifacts
          ↓
Document Findings
```

| Step | Explanation |
|---|---|
| Obtain Authorized Firmware | Only use firmware you're authorized to analyze (sample/legally obtained). |
| Preserve Original | Keep the original file untouched. |
| Calculate Hash | Establish integrity baseline before analysis. |
| Binwalk Signature Scan | Identify what's embedded in the binary. |
| Identify Embedded Components | Review what Binwalk detected. |
| Extract Components | Pull out identified structures for direct examination. |
| Identify File Systems | Determine what embedded file system(s), if any, were extracted. |
| Examine Files / Binaries | Review individual files within the extracted structure. |
| Search for Security-Relevant Artifacts | Look for the items described in Section 26. |
| Document Findings | Record the process and any findings clearly. |

---

## 26. Firmware Security Artifacts

After extraction, a researcher might examine:

- Configuration files
- Initialization scripts
- Web application files (device management interfaces)
- Service configurations
- Hardcoded credentials
- API keys/secrets
- Certificates
- SSH configuration
- Network configuration
- Version information
- Third-party libraries
- Executables

> Finding a string resembling a password or key does not automatically prove that it is active, valid, or exploitable — it's a lead requiring further verification, not a confirmed finding on its own.

---

## 27. Static Firmware Analysis

**Static analysis** means examining files without executing the firmware or any extracted binaries.

Covers:

- File-system structure
- Strings (readable text embedded in binaries, often revealing configuration paths, version info, or error messages)
- Configuration files
- Executables (examined structurally, not run)
- Libraries
- Version information

Static analysis is often a safer first step because it avoids the risks of actually running unknown/untrusted code. This document does not provide instructions for deploying extracted firmware or executing extracted binaries on real devices.

---

## 28. File Recovery + Firmware Analysis

```text
File Recovery
      ↓
Recover Raw / Embedded Data
      ↓
Identify File Structures
      ↓
Analyze Recovered Files
      ↓
Firmware Analysis
      ↓
Identify Embedded Components
      ↓
Extract File System
      ↓
Investigate Artifacts
```

Both areas covered on Day 25 involve understanding raw data structures rather than relying exclusively on normal file browsing — file carving recovers files from raw disk data using signatures, and firmware analysis identifies and extracts embedded structures from a raw binary image using the same underlying idea of signature-based recognition.

---

## 29. Foremost + Scalpel + Binwalk

| Tool | Main Purpose | Input | Key Technique | Typical Output |
|---|---|---|---|---|
| Foremost | General-purpose file carving | Raw data / disk image | Built-in signature matching | Recovered candidate files |
| Scalpel | Configurable file carving | Raw data / disk image | Configuration-driven signature matching | Recovered candidate files, tuned to configured types |
| Binwalk | Firmware/binary analysis | Firmware image / binary | Signature scanning + extraction | Identified structures / extracted embedded file systems |

### Foremost
Best understood as a general-purpose file-carving tool, useful for quick, broad recovery attempts.

### Scalpel
Best understood as a configurable file-carving tool, useful when precise control over targeted formats matters.

### Binwalk
Best understood as a firmware/binary analysis tool focused on identifying and extracting embedded structures within a binary, distinct in purpose from Foremost/Scalpel's file-carving focus even though all three rely on signature-based identification.

These tools are not interchangeable — Foremost and Scalpel target file carving from raw data, while Binwalk targets structural analysis of binary/firmware images.

---



## 30. Practical Evidence

### Evidence 1 — Foremost
> Add actual terminal screenshot here.

### Evidence 2 — Scalpel
> Add actual terminal screenshot here.

### Evidence 3 — Binwalk
> Add actual Binwalk output screenshot here.

### Evidence 4 — Extracted Firmware Structure
> Add actual screenshot here if available.

---

## 31. Command Reference

### Foremost

```bash
foremost -h
foremost -i <FORENSIC-IMAGE> -o <OUTPUT-DIRECTORY>
```

`-h` shows help/usage; `-i`/`-o` specify input and output respectively.

### Scalpel

```bash
scalpel -h
scalpel <FORENSIC-IMAGE> -o <OUTPUT-DIRECTORY>
```

Scalpel's configuration file determines which file types are carved — exact syntax and available options depend on the installed version.

### Binwalk

```bash
binwalk <FIRMWARE>
binwalk -e <FIRMWARE>
```

The first command performs read-only signature scanning; the second additionally extracts identified components.

---

## 32. Common Beginner Mistakes

### File Recovery

- Assuming every deleted file is recoverable.
- Ignoring fragmentation as a cause of failed/incomplete recovery.
- Assuming a recovered file is automatically valid without checking it.
- Forgetting to hash evidence before and after carving.
- Modifying the original evidence instead of working on a copy.
- Ignoring false positives in carving results.
- Losing sight of original file-system context (filename, path, timestamps) that carving doesn't preserve.

### Firmware Analysis

- Assuming every binary signature Binwalk reports is meaningful.
- Assuming extraction automatically means the device/firmware is compromised or vulnerable.
- Treating strings found in firmware as confirmed, active credentials without verification.
- Running unknown extracted firmware/executables directly instead of examining them statically first.
- Failing to preserve the original firmware image before analysis.
- Ignoring firmware version information when it's available.
- Assuming all firmware uses the same embedded file system.

---

## 33. Limitations

### Foremost
- Depends on known signatures — unknown formats won't be recovered.
- Struggles with fragmentation.
- Loses original metadata.
- Doesn't support every file format.
- Can produce false positives.

### Scalpel
- Depends on the configuration file being set up correctly.
- Still fundamentally signature-dependent.
- Struggles with fragmentation, like Foremost.
- More complex file formats can still be difficult to carve reliably.

### Binwalk
- Signature detection has limits — modified or unusual structures may go undetected.
- Unsupported or heavily customized firmware formats may not extract cleanly.
- Can produce false positives.
- Encrypted or heavily compressed content may not yield meaningful results without further work.
- Extraction can fail or produce incomplete results depending on the firmware's structure.

Across all three tools, output requires interpretation — none of them provide automatic, guaranteed-correct conclusions.

---

## 34. Forensic Evidence Integrity

Day 25 connects directly to Day 24's evidence-integrity principles:

- Preserve original evidence.
- Work on copies wherever possible.
- Hash evidence before and after analysis.
- Document tools and versions used.
- Record the exact commands run.
- Record timestamps of actions taken.
- Maintain chain-of-custody principles throughout.
- Avoid modifying evidence during analysis.

```text
Original Evidence
      ↓
Hash
      ↓
Forensic Copy
      ↓
Analysis
      ↓
Recovered / Extracted Data
      ↓
Hash / Document
      ↓
Report
```

Recovered/extracted artifacts (carved files, extracted firmware components) should themselves be tracked and documented — they become part of the evidence trail, not just intermediate scratch output.

---

## 35. Defensive Security Perspective

### File Recovery
Can assist investigations involving deleted files, data theft, incident response, and general evidence reconstruction — helping determine what a user or attacker did, even after attempted deletion.

### Firmware Analysis
Can help identify outdated components, hardcoded secrets, weak configurations, vulnerable services, unnecessary software, embedded web interfaces, and broader supply-chain/security risks — informing both defenders and manufacturers about a device's actual security posture.

Firmware analysis in particular can help manufacturers and security researchers improve embedded-device security by surfacing issues before (or after) they're exploited in the field.

---

## 36. Incident Response Application

```text
Security Incident
      ↓
Collect Evidence
      ↓
Preserve Evidence
      ↓
Recover Deleted / Hidden Data
      ↓
Analyze Embedded / Binary Data
      ↓
Correlate Artifacts
      ↓
Identify Security Findings
      ↓
Document
      ↓
Remediate
```

File recovery can help reconstruct deleted data relevant to an incident (e.g., files an attacker tried to remove), while firmware analysis becomes relevant when an incident involves embedded/IoT devices, where standard endpoint tooling may not apply.

---

## 37. Ethics and Legal Considerations

These tools must be used only on:

- Personally owned systems
- Authorized forensic evidence
- Intentionally vulnerable labs
- Legally obtained firmware
- CTF/training environments
- Systems covered by explicit permission

Do not analyze firmware obtained through unauthorized access. Do not deploy extracted firmware or discovered credentials against real devices without authorization.

---

## 38. What I Learned

Day 25 gave me a foundational understanding of file carving as a distinct technique from file-system-based recovery — one that works from raw signatures rather than remaining metadata, and that trades off metadata preservation for the ability to recover data even when the file system itself offers no help. Working through Foremost and Scalpel helped me understand the practical difference between a general-purpose carving tool and a configuration-driven one, and why an investigator might choose one over the other depending on the situation.

I also developed a foundational understanding of firmware as embedded software distinct from general-purpose applications, and how Binwalk's signature scanning and extraction capabilities make it possible to look inside an otherwise opaque firmware binary. I became familiar with common embedded file systems like SquashFS and JFFS2 at a conceptual level, and with the kinds of security-relevant artifacts (configuration files, hardcoded secrets, version information) that a researcher might look for after extraction — while also understanding that finding something suspicious is a lead requiring verification, not a confirmed finding.

I did not gain professional forensic expertise, advanced reverse-engineering skill, or complete firmware-exploitation knowledge from this day — my understanding remains foundational, and the actual practical exercises (Section 30) remain to be completed.

---

## 39. Key Takeaways

- Deleted does not always mean permanently destroyed.
- File carving can recover data based on file signatures rather than file-system metadata.
- File carving typically loses original file-system context (filenames, paths, timestamps).
- Foremost performs signature-based file carving with built-in signature handling.
- Scalpel provides configurable, configuration-driven file carving.
- Recovery does not guarantee a valid, complete file — validation is required.
- Fragmentation can make carving difficult or unreliable.
- Evidence should be preserved and hashed before analysis begins.
- Hashing helps verify evidence integrity throughout the process.
- Firmware is composed of multiple possible components (bootloader, kernel, root file system, etc.).
- Binwalk can identify embedded structures within a firmware binary via signature scanning.
- Firmware extraction can reveal an embedded file system for direct examination.
- Extracted firmware/binaries should be treated as untrusted data, not run casually.
- Static analysis (examining without executing) is often an appropriate first step.
- Tool output — from Foremost, Scalpel, or Binwalk — must be interpreted and corroborated, not accepted automatically.

---

## 40. Further Practice

These are **future/suggested practice items**, not completed work.

### Exercise 1 — File Carving
Use a legal/sample forensic image and compare Foremost and Scalpel — their recovered files, false-positive rates, and general limitations.

### Exercise 2 — File Signature Analysis
Inspect sample files and identify their file extension, actual file signature, and MIME/type information, noting any mismatches.

### Exercise 3 — Firmware Analysis
Use a legally obtained sample firmware image and run:
```bash
binwalk <FIRMWARE>
```
Then analyze the detected structures.

### Exercise 4 — Firmware Extraction
Use:
```bash
binwalk -e <FIRMWARE>
```
and inspect the extracted directory structure without executing any unknown binaries.

### Exercise 5 — Evidence Documentation
For every exercise, record: input, tool, version, exact command, output, hash, observation, and conclusion.

---

## 41. Connection to Day 24

```text
DAY 24
Digital Forensics
      ↓
Disk Analysis
Memory Analysis
Metadata Examination
      ↓
DAY 25
      ↓
File Recovery
File Carving
Firmware Analysis
```

Day 24 introduced forensic evidence analysis across disk, memory, and metadata. Day 25 goes deeper into recovering raw file data when normal file-system information is unavailable, and also introduces firmware as a specialized type of binary evidence relevant to embedded/IoT security. Together, these two days build a foundation for digital investigation that spans from structured file systems down to raw signature-based recovery and binary analysis.

---

## 42. Final Day 25 Summary

```text
              DAY 25
                 ↓
       ┌─────────┴─────────┐
       ↓                   ↓
 File Recovery       Firmware Analysis
       ↓                   ↓
   Foremost             Binwalk
   Scalpel                 ↓
       ↓              Extraction
   File Carving            ↓
       ↓              File Systems
   Validation               ↓
       └─────────┬─────────┘
                 ↓
        Forensic Analysis
```

Foremost and Scalpel provide two approaches to file carving — general-purpose and configuration-driven, respectively — while Binwalk provides signature scanning and extraction for firmware/binary analysis. All three tools automate parts of the investigative process, but none of them replace validation, correlation, documentation, and careful interpretation on the investigator's part.

> File recovery and firmware analysis are evidence-driven processes. Tools automate parts of the investigation, but accurate conclusions require validation, correlation, documentation, and careful interpretation.