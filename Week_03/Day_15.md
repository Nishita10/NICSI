# Day 15 — Password Auditing (Offline Recovery)


**Tools**
● John the Ripper ● Hashcat

**Topics**
● Hash Identification ● Offline Password Recovery ● Password Policy Review

A complete reference covering hash identification, offline password recovery, and password policy review — using John the Ripper and Hashcat. (Topics are explained first, followed by the tools — each explained by how it works and what it's used for, then its most important commands.)

> ⚠️ **Ethical Note:** Password cracking must only be performed on hashes you own, that belong to systems you're explicitly authorized to test, or that are provided for lab/CTF practice. Cracking real credentials without authorization is illegal in most jurisdictions regardless of how the hashes were obtained.

---

## Table of Contents
1. [Hash Identification](#1-hash-identification)
2. [Offline Password Recovery](#2-offline-password-recovery)
3. [Password Policy Review](#3-password-policy-review)
4. [Tool: John the Ripper](#4-tool-john-the-ripper)
5. [Tool: Hashcat](#5-tool-hashcat)
6. [Quick Cheat Sheet](#6-quick-cheat-sheet)

---

## 1. Hash Identification

**Hashing** is a one-way mathematical function that turns a password into a fixed-length string of characters (a **hash**) — the same input always produces the same output, but you can't reverse the hash back into the original password directly. Systems store password *hashes*, not plaintext passwords, so that even a full database breach doesn't hand over usable passwords directly.

**Hash identification** is the first step of any password-cracking workflow: figuring out **which hashing algorithm** produced a given hash, since every cracking tool needs to know this before it can even attempt to recover the original password.

### 1.1 Why This Step Can't Be Skipped
A hash is just a string of characters — there's no universal label attached to it saying "this is MD5" or "this is bcrypt." Different algorithms are recognized by clues like:
- **Length** — MD5 hashes are always 32 hex characters; SHA-1 is 40; SHA-256 is 64.
- **Format/prefix** — bcrypt hashes typically start with `$2a$`, `$2b$`, or `$2y$`; a Linux `/etc/shadow` SHA-512 hash starts with `$6$`.
- **Character set** — hex-only characters (0-9, a-f) suggest a raw hex digest; a mix including `$`, `.`, `/` often indicates a salted format with an embedded algorithm identifier.

### 1.2 Common Hash Types You'll Encounter

| Hash Type | Example Prefix/Length | Common Use |
|---|---|---|
| MD5 | 32 hex chars | Legacy, still seen in old systems (weak, fast to crack) |
| SHA-1 | 40 hex chars | Legacy (weak, deprecated for passwords) |
| SHA-256/512 | 64/128 hex chars | General-purpose hashing (fast — not ideal alone for passwords) |
| bcrypt | `$2a$`, `$2b$`, `$2y$` prefix | Purpose-built for passwords (slow/salted by design) |
| NTLM | 32 hex chars | Windows password hashes |
| Linux shadow (SHA-512 crypt) | `$6$` prefix | Modern `/etc/shadow` entries |

### 1.3 Why It Matters
- Feeding the wrong hash type into a cracking tool wastes time — it will either fail outright or, worse, silently produce garbage results.
- The hash type also tells you how **hard** cracking will realistically be — a fast, unsalted MD5 hash can be brute-forced far quicker than a slow, purpose-built bcrypt hash, which is designed specifically to resist this.

---

## 2. Offline Password Recovery

**Offline password recovery** (commonly just called "password cracking") is the process of recovering the original plaintext password from a hash **without needing to interact with the live system** at all — you already have the hash (e.g., from a database dump, a memory dump, or an authorized extraction), and the cracking happens entirely on your own machine.

### 2.1 "Offline" vs. "Online" — Why the Distinction Matters
This is different from *online* password attacks (like Hydra/Medusa, covered on Day 16), which repeatedly try logging into a live service over the network:

| | Offline Recovery | Online Attack |
|---|---|---|
| What you need | The password **hash** already extracted | Network access to a **live login** |
| Speed | Extremely fast — limited only by your hardware (millions/billions of guesses per second) | Slow — limited by network latency and target's rate limiting |
| Detectability | Invisible to the target — no requests ever reach them | Visible — generates login attempts/failed-login logs |
| Typical source | Database dumps, `/etc/shadow`, Windows SAM/NTDS, memory dumps | A live application's login form/API |

### 2.2 Core Attack Methods

| Method | Description |
|---|---|
| **Dictionary attack** | Try every word in a wordlist (plus common variations) as the candidate password |
| **Brute-force attack** | Try every possible combination of characters within a defined length/charset — guaranteed to succeed eventually, but can take an impractically long time for long/complex passwords |
| **Mask attack** | A smarter brute-force — you define a known *pattern* (e.g., "capital letter, 5 lowercase letters, 2 digits") instead of blindly trying every possible character combination |
| **Rule-based attack** | Take a dictionary and apply transformation rules to each word (e.g., append numbers, capitalize the first letter, swap `a`→`@`) — mimics how real people actually modify a base word into a "unique" password |
| **Hybrid attack** | Combines a dictionary with brute-force/mask elements (e.g., dictionary word + 4 brute-forced digits) |

### 2.3 Why It Matters
- Demonstrates, concretely, whether an organization's password hashes could realistically be cracked if a database were ever breached — the single most direct proof of weak password practices.
- Forms the technical foundation for Day 16's broader password auditing/wordlist topics.

---

## 3. Password Policy Review

**Password policy review** is the process of using the *results* of a cracking exercise to evaluate — and recommend improvements to — an organization's actual password requirements and practices, rather than just treating cracking as a one-off technical exercise.

### 3.1 What a Successful Crack Run Actually Tells You
Every password you successfully recover is a data point about the *real* policy in effect, regardless of what's written in a company handbook:
- **Cracked in seconds with a small dictionary** → passwords are likely default, common, or based on the company/product name.
- **Cracked only with rule-based mangling (e.g., `Password1!`)** → users are meeting *minimum complexity requirements* technically, but in predictable, low-entropy ways.
- **Not cracked at all within a reasonable time budget** → suggests genuinely strong, random passwords are in use (or a slow/well-salted hashing algorithm is doing its job).
- **Many accounts sharing the same cracked password** → suggests a weak default password policy, shared credentials, or a recent breach's password being reused.

### 3.2 What a Good Review Covers
- **Minimum length & complexity requirements** — are they actually enforced, and do cracked passwords suggest the policy is too weak (e.g., 8 characters is now considered short)?
- **Password reuse/rotation practices** — do cracked hashes reveal the same password reused across multiple accounts?
- **Hashing algorithm in use** — is the organization still using a fast, weak algorithm (MD5/SHA-1) instead of a purpose-built slow one (bcrypt/Argon2/scrypt)?
- **Presence of default/vendor credentials** — were any hashes simply the unchanged default from a vendor or template account?

### 3.3 Why It Matters
- Turns a technical exercise into an **actionable business recommendation** — the real deliverable of a password audit isn't "we cracked 40% of passwords," it's "here's exactly what policy change would fix it."
- Often the single most cost-effective security improvement an organization can make — better password policy enforcement is usually far cheaper than most other security controls.

---

## 4. Tool: John the Ripper

**John the Ripper** ("John") is one of the oldest and most widely used offline password cracking tools — known for its broad hash-format support, built-in hash auto-detection, and its efficient default cracking modes.

### 4.1 How It Works
- John takes a file of hashes (in a supported format) and attempts to recover the plaintext using one of several **modes**: single-crack mode, wordlist mode, incremental (brute-force) mode, or a combination.
- It includes a large library of hash-format parsers, and in many cases can **auto-detect the hash type** directly from the file — directly applying Section 1's concept.
- Its default **wordlist mode** applies built-in **mangling rules** automatically — trying variations of each dictionary word (capitalization, appended digits, leetspeak substitutions) without you needing to configure this manually, which is one of John's most practical, time-saving features.
- Results (cracked passwords) are stored in John's own **"pot file"** (`john.pot`), and can be displayed at any time with a separate `--show` command.

### 4.2 What It's Used For
- The default **first tool to reach for** when you have a hash file and aren't sure of its exact format — its auto-detection handles the Section 1 identification step for you in many cases.
- CPU-based cracking — John is traditionally CPU-focused (though it has GPU support via extensions), making it convenient for quick jobs without needing specialized hardware.
- Auditing Linux `/etc/shadow` password hashes and other common system-level credential stores.

### 4.3 Important Commands

```bash
john --format=raw-md5 hashes.txt
```
Runs John against a hash file, explicitly specifying the **format** — useful when you already know the hash type from Section 1's identification step.

```bash
john hashes.txt
```
Runs John **without specifying a format**, letting its auto-detection attempt to identify the hash type itself — a good first try when the hash type is unknown.

```bash
john --wordlist=rockyou.txt hashes.txt
```
Runs a **dictionary attack** using a specified wordlist (`rockyou.txt` is the most commonly used practice wordlist) against the hash file.

```bash
john --wordlist=rockyou.txt --rules hashes.txt
```
`--rules` applies John's built-in **mangling rules** on top of the wordlist — automatically trying capitalized, appended-digit, and leetspeak variants of each word.

```bash
john --incremental hashes.txt
```
Runs a full **brute-force (incremental) attack** — tries all possible character combinations; slow, but exhaustive, best reserved for shorter/simpler hashes.

```bash
john --show hashes.txt
```
Displays all **already-cracked passwords** for the given hash file from John's pot file — the command you'll run after a cracking session to see the actual results.

```bash
john --status
```
Shows the **live progress** of a currently running John session — useful for long-running jobs to check how far along the attack is.

```bash
unshadow /etc/passwd /etc/shadow > combined.txt
```
A companion utility (not John itself) that **merges** `/etc/passwd` and `/etc/shadow` into one file John can process — a required prep step before cracking real Linux system hashes.

```bash
john --wordlist=rockyou.txt --format=nt hashes.txt
```
Targets **Windows NTLM hashes** (`--format=nt`) specifically — common when auditing hashes extracted from a Windows SAM or NTDS dump.

---

## 5. Tool: Hashcat

**Hashcat** is the fastest widely-used password cracking tool available, built to leverage **GPU acceleration** — where John is traditionally CPU-first, Hashcat is built GPU-first, making it dramatically faster for large-scale cracking jobs, especially against fast hash algorithms.

### 5.1 How It Works
- Hashcat uses your **GPU's massive parallel processing power** (rather than the CPU) to compute hashes at a far higher rate — for a fast algorithm like MD5, this can mean billions of guesses per second on modern hardware, versus millions on a CPU.
- Every hash type in Hashcat is identified by a **numeric mode code** (e.g., `0` = MD5, `1000` = NTLM, `1800` = SHA-512 crypt) — Hashcat doesn't auto-detect by default the way John does, so correctly identifying the hash type (Section 1) is a mandatory manual step before running it.
- It supports the same core attack categories as John (dictionary, brute-force, hybrid) but adds a very powerful **mask attack** syntax (`-a 3`) for precisely defined brute-force patterns, and highly flexible **rule files** for advanced dictionary mangling.
- Like John, cracked results are stored (in a `hashcat.potfile`) and can be reviewed at any time with `--show`.

### 5.2 What It's Used For
- **Large-scale or time-sensitive cracking jobs** — where John's CPU-based speed isn't practical, Hashcat's GPU acceleration is the standard choice.
- Precisely targeted **mask attacks** — when you have strong intelligence about a password's likely structure (e.g., "always starts with a capital letter and ends in 2 digits" from a known company password policy).
- Cracking large hash dumps efficiently across many hashes at once (Hashcat is well-optimized for cracking many hashes of the same type in a single run).

### 5.3 Important Commands

```bash
hashcat -m 0 -a 0 hashes.txt rockyou.txt
```
The core command — `-m 0` sets the hash mode (MD5 here), `-a 0` sets the **attack mode** (0 = dictionary), and this runs a standard dictionary attack against the hash file using `rockyou.txt`.

```bash
hashcat --help | grep -i md5
```
Searches Hashcat's mode list for a specific algorithm name — the practical way to find the correct `-m` number for a hash type identified in Section 1.

```bash
hashcat -m 1000 -a 0 ntlm_hashes.txt rockyou.txt
```
Targets **NTLM hashes** (`-m 1000`) with a dictionary attack — the Hashcat equivalent of John's `--format=nt` example above.

```bash
hashcat -m 0 -a 3 hashes.txt ?u?l?l?l?l?d?d
```
Runs a **mask attack** (`-a 3`) — the mask `?u?l?l?l?l?d?d` defines an exact pattern: 1 uppercase letter, 4 lowercase letters, 2 digits — a far more efficient brute-force than trying every possible character blindly.

```bash
hashcat -m 0 -a 0 hashes.txt rockyou.txt -r rules/best64.rule
```
`-r` applies a **rule file** to the wordlist attack — Hashcat's equivalent of John's `--rules`, but with more extensive, customizable rule sets available (e.g., the widely-used `best64.rule`).

```bash
hashcat -m 0 -a 6 hashes.txt rockyou.txt ?d?d?d?d
```
Runs a **hybrid attack** (`-a 6`, wordlist + mask) — appends a 4-digit brute-forced mask to the end of each dictionary word, e.g. testing `password1234`, `password0001`, etc.

```bash
hashcat --show hashes.txt
```
Displays all passwords **already cracked** for the given hash file from Hashcat's potfile — run this any time to check results without re-running the attack.

```bash
hashcat --status --status-timer=10 -m 0 -a 0 hashes.txt rockyou.txt
```
Enables **live status updates** (every 10 seconds) during a running attack — shows current speed, progress percentage, and estimated time remaining.

```bash
hashcat -m 0 -a 0 hashes.txt rockyou.txt -O
```
`-O` enables **optimized kernel mode** — trades a small amount of maximum candidate password length for a significant speed boost, useful for most real-world password lengths.

```bash
hashcat -I
```
Lists available **GPU/hardware devices** Hashcat can use — a useful diagnostic/setup command to confirm your hardware is correctly detected before running a serious job.

---

## 6. Quick Cheat Sheet

| Task | Command |
|---|---|
| Auto-detect and crack a hash (CPU) | `john hashes.txt` |
| Find a hash's Hashcat mode number | `hashcat --help \| grep -i <algorithm>` |
| Dictionary attack (John) | `john --wordlist=rockyou.txt hashes.txt` |
| Dictionary attack (Hashcat, GPU) | `hashcat -m 0 -a 0 hashes.txt rockyou.txt` |
| Dictionary + mangling rules (John) | `john --wordlist=rockyou.txt --rules hashes.txt` |
| Dictionary + rule file (Hashcat) | `hashcat -m 0 -a 0 hashes.txt rockyou.txt -r rules/best64.rule` |
| Pattern-based brute-force (Hashcat mask) | `hashcat -m 0 -a 3 hashes.txt ?u?l?l?l?l?d?d` |
| Show cracked results | `john --show hashes.txt` / `hashcat --show hashes.txt` |
| Prep Linux hashes for cracking | `unshadow /etc/passwd /etc/shadow > combined.txt` |

**Typical Day 15 Password Audit Workflow (lab/authorized environment):**
```bash
# 1. Identify the hash type (length, prefix, character set — Section 1)
unshadow /etc/passwd /etc/shadow > combined.txt          # 2. Prep Linux hashes if needed
john --wordlist=rockyou.txt --rules combined.txt            # 3. Fast CPU dictionary + rules pass (John)
john --show combined.txt                                       # 4. Review what John already cracked
hashcat -m 1800 -a 0 combined.txt rockyou.txt -r rules/best64.rule -O   # 5. GPU pass on anything left (Hashcat)
hashcat -m 1800 -a 3 combined.txt ?u?l?l?l?l?l?d?d              # 6. Targeted mask attack on remaining hashes
hashcat --show combined.txt                                        # 7. Final results
# 8. Compare cracked passwords against policy expectations -> write the Password Policy Review (Section 3)
```

---

### Practice Suggestions for Day 15
1. Generate a few test hashes of different types (MD5, SHA-256, bcrypt) from known passwords, and practice identifying each by length/prefix before looking up the answer.
2. Crack a small MD5 hash list with John using default settings, then re-run the same list through Hashcat and compare completion time.
3. Add `--rules` (John) or `-r rules/best64.rule` (Hashcat) to a dictionary attack and note how many *additional* passwords get cracked versus a plain wordlist run.
4. Design a Hashcat mask attack for a password pattern you define yourself (e.g., "Company" + 2 digits + symbol) and time how long a full mask run takes versus a blind incremental brute-force.
5. After cracking a batch of practice hashes, write a short Password Policy Review (Section 3 format) summarizing what the cracked passwords reveal about weak patterns, and recommend 3 concrete policy changes.
