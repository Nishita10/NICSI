# Day 16 — Online Password Auditing & Wordlist Generation


**Tools**
● Hydra ● Medusa ● CeWL ● Crunch ● CUPP ● Hash-Identifier

**Topics**
● Password Auditing ● Wordlist Generation

A complete reference covering online password auditing and wordlist generation — using Hydra, Medusa, CeWL, Crunch, CUPP, and Hash-Identifier. (Topics are explained first, followed by the tools — each explained by how it works and what it's used for, then its most important commands.)

> ⚠️ **Ethical Note:** Online password attacks send real login attempts to a live service and can lock out accounts, trigger alerts, or overwhelm a target if run aggressively. Only run these against systems you own or have explicit written authorization to test, and always agree on rate limits/account-lockout risk with the target owner beforehand.

---

## Table of Contents
1. [Password Auditing (Online)](#1-password-auditing-online)
2. [Wordlist Generation](#2-wordlist-generation)
3. [Tool: Hydra](#3-tool-hydra)
4. [Tool: Medusa](#4-tool-medusa)
5. [Tool: CeWL](#5-tool-cewl)
6. [Tool: Crunch](#6-tool-crunch)
7. [Tool: CUPP](#7-tool-cupp)
8. [Tool: Hash-Identifier](#8-tool-hash-identifier)
9. [Quick Cheat Sheet](#9-quick-cheat-sheet)

---

## 1. Password Auditing (Online)

Day 15 covered **offline** password recovery — cracking a hash you already possess, entirely on your own hardware. **Online password auditing** is the other half of the picture: testing password strength by actually attempting to **log in** to a live service, one guess at a time, over the network.

### 1.1 How This Differs From Offline Cracking

| | Offline Cracking (Day 15) | Online Auditing (Day 16) |
|---|---|---|
| What's needed | A password hash already extracted | Network access to a live login (SSH, FTP, a web login form, etc.) |
| Speed | Extremely fast (millions–billions/sec) | Slow — limited by network round-trip time and the service itself |
| Visibility to target | Invisible — nothing is sent to the target | Very visible — generates real failed-login attempts and log entries |
| Risk of side effects | None to the live system | Can trigger account lockouts, alerts, or rate-limiting/bans |

### 1.2 Why This Still Matters, Even Though It's Slower
- Many services **never expose a crackable hash at all** — a router's admin panel, an SSH server, an FTP server, or a web login form only ever show you a login prompt, never a hash. Online auditing is the *only* way to test password strength against these.
- It tests the **real-world, end-to-end** authentication path — including things offline cracking can't reveal, like whether account lockout policies or rate-limiting are actually configured and working.
- It's how weak/default credentials get discovered in practice — testing a short list of common defaults (`admin:admin`, `root:toor`) against a live service is one of the highest-value, lowest-effort checks in an assessment.

### 1.3 Core Techniques
- **Credential stuffing** — trying known username/password *pairs* (e.g., from a previous breach) against a login.
- **Password spraying** — trying one or a few common passwords against **many** usernames, deliberately slow and spread out to avoid triggering per-account lockouts.
- **Brute-force/dictionary login attempts** — systematically trying many password guesses against one or a few known usernames — exactly what Hydra and Medusa are built for.

---

## 2. Wordlist Generation

**Wordlist generation** is the process of building the actual list of candidate passwords or usernames that get fed into both online (Hydra/Medusa) and offline (Day 15's John/Hashcat) attacks. A generic wordlist finds generic passwords — a **well-built, targeted** wordlist is often what actually determines whether an audit succeeds.

### 2.1 Why Generic Wordlists Aren't Always Enough
A massive public wordlist like `rockyou.txt` is a great general-purpose starting point, but real people and organizations often build passwords around things that are **specific to them**:
- A company name, product name, or founding year (`AcmeCorp2024!`).
- Personal details for a specific target — pet names, birthdays, family names (relevant in targeted, authorized assessments).
- Words that are common *on the target's own website* — product names, internal jargon, staff names scraped straight from their public content.

### 2.2 Two Broad Generation Approaches

| Approach | Description |
|---|---|
| **Scraped/targeted** | Pull real words directly from a target's own content (a website, social media) — captures organization-specific vocabulary |
| **Pattern-based/synthetic** | Systematically generate every possible combination matching a defined pattern (length, character set) — guarantees full coverage of a defined space |
| **Profile-based** | Take specific known facts about one person (name, birthdate, pet) and generate the likely password variations a human would actually create from them |

### 2.3 Why It Matters
- Directly increases the **success rate** of both online and offline password attacks — a well-targeted wordlist of a few thousand entries often outperforms a generic list of millions.
- Demonstrates concretely, in a report, *why* a policy like "don't use company/product names in your password" exists — because tools like these can generate and test exactly those guesses in minutes.

---

## 3. Tool: Hydra

**Hydra** (THC-Hydra) is the most widely used **online login brute-forcing** tool — it supports a huge range of protocols and services, making it the default choice for testing live authentication systems.

### 3.1 How It Works
- Hydra takes a target service, a username (or list of usernames), and a password list, then systematically attempts each username/password combination against the live login — checking whether the service responds with success or failure.
- It supports dozens of protocols/services out of the box — SSH, FTP, RDP, HTTP(S) forms, MySQL, SMB, and many more — each handled by its own protocol module.
- It's built for **speed via parallelism** — running many login attempts concurrently (configurable) — but this must be balanced against the target's rate-limiting/lockout policies (Section 1.3).

### 3.2 What It's Used For
- Testing SSH, FTP, RDP, and other **service logins** for weak or default credentials.
- Testing **web application login forms** specifically, using its HTTP-form module.
- The standard tool for demonstrating, in a report, whether an organization's live services are actually protected against basic brute-force attempts.

### 3.3 Important Commands

```bash
hydra -l admin -P rockyou.txt ssh://192.168.1.10
```
The core command — `-l` sets a single username, `-P` supplies a password wordlist, targeting **SSH**. This is the pattern you'll adapt for most services.

```bash
hydra -L users.txt -P rockyou.txt ftp://192.168.1.10
```
`-L` (capital) supplies a **list of usernames** instead of one — combined with a password list, Hydra tries every username/password pair against **FTP**.

```bash
hydra -l admin -p "admin123" ssh://192.168.1.10
```
`-p` (lowercase) tests a **single specific password**, useful for quickly checking one known/default credential rather than running a full list.

```bash
hydra -l admin -P rockyou.txt 192.168.1.10 http-post-form "/login:username=^USER^&password=^PASS^:Invalid credentials"
```
Targets a **web login form** via HTTP POST — `^USER^`/`^PASS^` mark where Hydra substitutes each guess, and the final field defines the **failure string** Hydra looks for to know an attempt failed.

```bash
hydra -l admin -P rockyou.txt -t 4 ssh://192.168.1.10
```
`-t` sets the number of **parallel tasks/threads** — lower values (like 4) are gentler on the target and less likely to trigger lockouts, an important tuning choice discussed in Section 1.

```bash
hydra -l admin -P rockyou.txt -f ssh://192.168.1.10
```
`-f` tells Hydra to **stop as soon as one valid credential pair is found** — saves time when you just need to confirm a weakness exists, not enumerate every valid combination.

```bash
hydra -l admin -P rockyou.txt -o results.txt ssh://192.168.1.10
```
`-o` saves found credentials to an **output file** for reporting.

```bash
hydra -C userpass.txt ssh://192.168.1.10
```
`-C` uses a **combined username:password file** (one `user:pass` pair per line) instead of separate lists — useful for credential-stuffing-style tests with known real pairs.

---

## 4. Tool: Medusa

**Medusa** is a fast, modular online login brute-forcer very similar in purpose to Hydra — often described as Hydra's closest alternative, with a slightly different internal design and its own strengths around parallelism and modularity.

### 4.1 How It Works
- Like Hydra, Medusa takes a target service, username(s), and password list, and systematically attempts logins — but it's built around a strictly **modular architecture**, where each supported protocol is a separate loadable module.
- Medusa is designed to be **thread-per-host** in its parallelism model, which in some scenarios (especially testing many hosts at once) can behave more predictably/efficiently than Hydra's approach.
- Its command syntax is deliberately similar in spirit to Hydra's (same core concepts: user list, password list, target, module) but with its own flag names, which is a common point of confusion when switching between the two.

### 4.2 What It's Used For
- Functionally the **same use case as Hydra** — online brute-forcing of live services — chosen as an alternative when Medusa's specific module for a service performs better, or simply as a second tool to cross-verify Hydra's results.
- Testing across **multiple target hosts** at once, where its thread-per-host model is a natural fit.

### 4.3 Important Commands

```bash
medusa -h 192.168.1.10 -u admin -P rockyou.txt -M ssh
```
The core command — `-h` sets the host, `-u` sets a single username, `-P` sets the password list, and `-M` selects the **module** (`ssh` here). This is Medusa's equivalent of the basic Hydra command above.

```bash
medusa -h 192.168.1.10 -U users.txt -P rockyou.txt -M ftp
```
`-U` (capital) supplies a **username list** — combined with a password list, tests every combination against **FTP**.

```bash
medusa -H hosts.txt -u admin -P rockyou.txt -M ssh
```
`-H` supplies a **list of target hosts** instead of one — Medusa tests the same username/password list against every host in the file, useful for auditing many machines at once.

```bash
medusa -h 192.168.1.10 -u admin -p "admin123" -M ssh
```
`-p` (lowercase) tests a **single password**, same purpose as Hydra's `-p` — quick check of one known/default credential.

```bash
medusa -h 192.168.1.10 -u admin -P rockyou.txt -M ssh -t 4
```
`-t` sets the number of **parallel threads** — same speed/stealth trade-off consideration as Hydra's `-t`.

```bash
medusa -h 192.168.1.10 -u admin -P rockyou.txt -M ssh -f
```
`-f` stops after the **first successful login found** for a given host, same purpose as Hydra's `-f`.

```bash
medusa -d
```
Lists all **available modules** installed on your system — useful since Medusa's supported protocols depend on which modules are compiled/installed.

```bash
medusa -h 192.168.1.10 -u admin -P rockyou.txt -M ssh -O results.txt
```
`-O` saves output/results to a file for later reporting.

---

## 5. Tool: CeWL

**CeWL** (Custom Word List generator) is a targeted wordlist-generation tool that **crawls a website** and builds a wordlist directly from the actual words it finds on the pages — a practical implementation of Section 2's "scraped/targeted" approach.

### 5.1 How It Works
- CeWL spiders a given URL up to a specified depth, visiting linked pages within the site.
- As it crawls, it extracts every distinct word from the visible page text, filtering by a configurable **minimum word length** to avoid collecting noise like "a," "the," "is."
- It can optionally also extract **email addresses** and **metadata author names** found in documents linked from the site (e.g., PDF/DOCX authorship metadata), adding those directly to its output.
- The result is a wordlist built entirely from language the organization *actually uses* — product names, staff names, internal terminology — exactly the kind of vocabulary real employees tend to draw from when choosing passwords.

### 5.2 What It's Used For
- Building a **highly targeted wordlist** specific to one organization, to feed into Hydra/Medusa (online) or John/Hashcat (offline, Day 15) — usually far more effective per-entry than a generic wordlist for that specific target.
- Harvesting **email addresses and document authors** as a side effect — useful input for Day 5's email enumeration work too.

### 5.3 Important Commands

```bash
cewl https://example.com
```
The core command — crawls the target site with default settings and prints the extracted wordlist to the terminal.

```bash
cewl https://example.com -d 2 -m 5
```
`-d` sets the **crawl depth** (how many link-levels deep to follow), and `-m` sets the **minimum word length** to include — filters out short, low-value words.

```bash
cewl https://example.com -w wordlist.txt
```
`-w` saves the generated wordlist directly to a file, ready to feed into Hydra, Medusa, John, or Hashcat.

```bash
cewl https://example.com -e -w wordlist.txt
```
`-e` also extracts **email addresses** found during the crawl, alongside the normal wordlist output.

```bash
cewl https://example.com --with-numbers -w wordlist.txt
```
Includes words that contain **numbers** in the output (excluded by default) — useful since real passwords/usernames often mix letters and digits.

```bash
cewl https://example.com -a -w wordlist.txt
```
`-a` also extracts **author metadata** from linked documents (e.g., PDFs) — potential real names of staff who created target-related documents.

```bash
cewl https://example.com --lowercase -w wordlist.txt
```
Normalizes all extracted words to **lowercase**, which is often desirable before feeding a list into rule-based mangling in John/Hashcat (Day 15).

---

## 6. Tool: Crunch

**Crunch** is a straightforward **pattern-based wordlist generator** — instead of scraping content like CeWL, it systematically generates **every possible combination** of characters matching a defined length and character set, directly implementing Section 2's "pattern-based/synthetic" approach.

### 6.1 How It Works
- You specify a **minimum length**, **maximum length**, and a **character set** (letters, digits, symbols, or a custom set) — Crunch then generates every possible string matching those constraints, one per line.
- It supports **patterns/placeholders** for more targeted generation — e.g., forcing specific fixed characters in specific positions while only varying the rest.
- Because full character-set brute-force lists grow enormously fast (charset size ^ length), Crunch is most practical for short lengths or well-defined masks — for longer/complex patterns, Hashcat's built-in mask attack (Day 15) is usually more efficient since it doesn't need to pre-generate and store the entire list on disk.

### 6.2 What It's Used For
- Generating **exhaustive, pattern-defined password lists** — e.g., "all 4-digit PINs," or "all 6-character lowercase+digit combinations."
- Producing custom wordlists for known, narrow password formats (e.g., a company's known default password scheme like `Welcome123`, `Welcome124`...).

### 6.3 Important Commands

```bash
crunch 4 4 0123456789
```
Generates all possible **4-digit numeric combinations** (0000–9999) — a classic use case for testing PIN-style passwords.

```bash
crunch 6 8 abcdefghijklmnopqrstuvwxyz
```
Generates all combinations of **lowercase letters**, from 6 to 8 characters long — note how quickly this grows; always check the estimated output size first.

```bash
crunch 8 8 -f charset.lst mixalpha-numeric-all-space
```
`-f` loads a **predefined character set** from Crunch's bundled charset file, here using a mixed alphanumeric set — saves you from typing out large character sets manually.

```bash
crunch 6 6 -t Pass%%
```
`-t` defines a **pattern/template** — `%` represents a position that varies (numeric here), while everything else stays fixed; this example generates `Pass00` through `Pass99`.

```bash
crunch 8 8 -o wordlist.txt
```
`-o` saves the generated list to a file instead of printing it to the terminal — essential given how large these files can get.

```bash
crunch 6 6 0123456789 -c 1000000000 -o START
```
`-c` limits output to a specific **count** of entries per file, automatically splitting output into multiple numbered files (`START` names the file series) — useful for managing very large generation jobs.

```bash
crunch 4 6 0123456789abcdef -p
```
`-p` (**permutation mode**) generates every possible **order** of the given characters *without repeating any character* — a different mode than the default (which allows repeats), used for very specific pattern requirements.

---

## 7. Tool: CUPP

**CUPP** (Common User Passwords Profiler) is a **profile-based** wordlist generator — instead of scraping a website or brute-forcing patterns, it interactively asks you for known facts about **one specific person** (name, birthdate, pet, partner, etc.) and generates the realistic password variations a real person tends to build from that information.

### 7.1 How It Works
- CUPP runs an **interactive questionnaire**, prompting for details like: first/last name, nickname, birth year, partner's name, children's names, pet's name, and any favorite/relevant words.
- It then applies known **human password-creation patterns** to these inputs — combining words, appending birth years, capitalizing first letters, adding common special characters at the end — generating a list of realistic candidate passwords a person with that profile might actually choose.
- This directly encodes the same psychology described in Section 2 — humans build "unique-feeling" passwords from personal, memorable details, and CUPP systematizes that guesswork.

### 7.2 What It's Used For
- **Targeted, authorized social-engineering-adjacent assessments** — e.g., testing whether a specific, known individual's account uses a predictable, personally-themed password.
- A more personal complement to CeWL's organization-wide scraping — CUPP focuses on **one person**, CeWL focuses on **one organization's content**.

### 7.3 Important Commands

```bash
cupp -i
```
Launches CUPP's **interactive mode** — prompts you through a series of questions about the target individual, then generates a wordlist based on the answers. This is CUPP's primary, most-used mode.

```bash
cupp -w wordlist.txt -l
```
`-l` runs against an **existing wordlist**, applying CUPP's "leet mode" transformations (`a`→`@`, `e`→`3`, etc.) to expand it with common substitution variants.

```bash
cupp -a
```
Downloads/updates CUPP's **alecto database** — a bundled list of common default router/device usernames and passwords, useful as a quick reference alongside your generated list.

```bash
cupp --version
```
Displays the installed CUPP version — useful to confirm before relying on a specific feature set, since CUPP's option availability can vary between versions.

```bash
cat config/cupp.cfg
```
Not a run command, but a key file to know: CUPP's **configuration file** defines default appended patterns/suffixes it uses during generation — reviewing it helps you understand exactly what transformation logic is being applied to your answers.

---

## 8. Tool: Hash-Identifier

**Hash-Identifier** is a small, focused utility that directly automates Section 1's manual hash-identification process — you paste in a hash, and it tells you which algorithm(s) it's likely to be.

### 8.1 How It Works
- You provide a hash string as input (interactively or via a file).
- The tool checks the hash's **length** and **character set/format** against its internal database of known hash-type signatures (the same clues described in Day 15 — length, prefix, allowed characters).
- Since many hash types share the same length (e.g., a 32-character hex string could be MD5, NTLM, or several others), it typically returns a **ranked list of possible matches** rather than a single definitive answer — narrowing your options rather than always giving one certain answer.

### 8.2 What It's Used For
- The fast, first step whenever you're handed an unfamiliar hash and need to know what you're dealing with **before** choosing a John format or a Hashcat `-m` mode number (Day 15).
- A quick sanity check/second opinion alongside John the Ripper's own auto-detection.

### 8.3 Important Commands

```bash
hash-identifier
```
Launches the tool in **interactive mode** — it then prompts you to paste in a hash string and immediately returns its best-guess list of matching algorithms.

```bash
echo "5f4dcc3b5aa765d61d8327deb882cf99" | hash-identifier
```
Pipes a hash directly into the tool non-interactively — useful for quick, scriptable checks without needing to sit through the interactive prompt.

```bash
hashid <hash>
```
`hashid` is a modern, actively maintained alternative/successor tool with very similar purpose — often bundled alongside or instead of hash-identifier on newer Kali installs, worth knowing as the current equivalent command.

```bash
hashid -m <hash>
```
Requests `hashid` output the corresponding **Hashcat mode number** directly alongside each guessed algorithm — directly bridging the identification step into Day 15's Hashcat `-m` flag.

---

## 9. Quick Cheat Sheet

| Task | Command |
|---|---|
| Identify an unknown hash | `hash-identifier` or `hashid -m <hash>` |
| Online SSH brute-force | `hydra -l admin -P rockyou.txt ssh://target` |
| Online brute-force across multiple hosts | `medusa -H hosts.txt -u admin -P rockyou.txt -M ssh` |
| Scrape a targeted wordlist from a site | `cewl https://target.com -d 2 -m 5 -w wordlist.txt` |
| Generate all 4-digit PINs | `crunch 4 4 0123456789` |
| Generate a person-specific wordlist | `cupp -i` |
| Test one known default credential | `hydra -l admin -p admin123 ssh://target` |

**Typical Day 16 Online Auditing Workflow (lab/authorized environment):**
```bash
hashid -m "5f4dcc3b5aa765d61d8327deb882cf99"                        # 1. Identify any hash types on hand first
cewl https://target.com -d 2 -m 5 -e -w wordlists/cewl_target.txt      # 2. Build a targeted, scraped wordlist
cupp -i                                                                    # 3. Build a person-specific wordlist for a known individual
crunch 6 6 -t Target%% -o wordlists/crunch_pattern.txt                       # 4. Generate any known-pattern candidates
hydra -l admin -P wordlists/cewl_target.txt -t 4 -f ssh://target             # 5. Careful, rate-limited online test (Hydra)
medusa -h target -u admin -P wordlists/cewl_target.txt -M ssh -t 4              # 6. Cross-verify with Medusa
```

---

### Practice Suggestions for Day 16
1. Run `hash-identifier`/`hashid` against a few known hash examples (MD5, SHA-1, NTLM) and confirm it correctly narrows down the possibilities.
2. Run CeWL against a lab website and review how many of the extracted words are genuinely relevant (product names, staff names) versus generic noise.
3. Use CUPP's interactive mode with made-up (but realistic) profile details, and review how many of the generated passwords resemble ones you've seen used in practice.
4. Generate a small Crunch wordlist with a defined pattern (`-t`) and compare its size/runtime against generating the same length with no pattern at all.
5. Run Hydra and Medusa against the same lab SSH service with the same wordlist and low thread count, and compare their runtime and output format.
6. Combine a CeWL-generated wordlist with John's `--rules` (Day 15) and see how many additional variations get tested compared to the raw scraped list alone.
