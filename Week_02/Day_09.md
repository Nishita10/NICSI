# Day 9 — Web Fingerprinting & Template-Based Security Assessment


**Tools**
● Nuclei ● WhatWeb ● Wapiti

**Topics**
● Web Fingerprinting ● Template-Based Checks ● Security Assessment

A complete reference covering web fingerprinting, template-based checks, and security assessment — using Nuclei, WhatWeb, and Wapiti. (Topics are explained first, followed by the tools — each explained by how it works and what it's used for, then its most important commands.)


---

## Table of Contents
1. [Web Fingerprinting](#1-web-fingerprinting)
2. [Template-Based Checks](#2-template-based-checks)
3. [Security Assessment](#3-security-assessment)
4. [Tool: WhatWeb](#4-tool-whatweb)
5. [Tool: Nuclei](#5-tool-nuclei)
6. [Tool: Wapiti](#6-tool-wapiti)
7. [Quick Cheat Sheet](#7-quick-cheat-sheet)

---

## 1. Web Fingerprinting

**Web fingerprinting** is the process of identifying exactly what technologies power a website — the web server, programming language/framework, CMS, JavaScript libraries, analytics tools, and third-party plugins — purely by examining what the site exposes publicly.

### 1.1 Where the Clues Come From
Websites constantly leak hints about their tech stack, often without meaning to:
- **HTTP response headers** — e.g., `Server: nginx/1.18.0`, `X-Powered-By: PHP/7.4.3`.
- **Cookies** — e.g., a cookie named `PHPSESSID` implies PHP; `wordpress_logged_in_` implies WordPress.
- **HTML source** — `<meta name="generator" content="WordPress 6.2">`, or file paths like `/wp-content/` or `/_next/`.
- **JavaScript files & error messages** — library filenames (`jquery-3.6.0.min.js`), framework-specific console errors, or stack traces revealing internal file paths.
- **Favicon hashes** — many fingerprinting tools match a site's favicon against a database of known favicons used by specific software (e.g., default Jenkins or GitLab icons).

### 1.2 Why It Matters
- Once you know the exact **software and version** in use, you immediately know **which CVEs and known misconfigurations apply** — fingerprinting is the essential first step that makes Day 8's CVE detection possible at the web layer.
- Different technologies need different testing approaches — assessing a WordPress site looks very different from assessing a custom Node.js API.
- Fingerprinting is almost entirely **passive/low-risk** — it mostly involves reading what the server already sends you, rather than actively probing for weaknesses.

---

## 2. Template-Based Checks

**Template-based checks** are a scanning approach where each individual vulnerability test is written as a small, standardized, reusable definition file (a "template") — describing exactly what request to send and what response pattern would confirm the issue — rather than being hardcoded into the scanner itself.

### 2.1 How the Concept Works
A template typically defines:
- **What to send** — a specific HTTP request (a path, method, headers, body).
- **What to look for** — a matcher, such as a specific string in the response, a status code, or a regex pattern.
- **Metadata** — severity, CVE ID (if applicable), affected software, and a description.

```yaml
# Simplified conceptual example of what a template describes:
id: example-exposed-env-file
request: GET /.env
matcher: response contains "DB_PASSWORD"
severity: high
```

Because templates are just structured text files (commonly YAML), anyone — the tool's maintainers, the security community, or you — can write a new check the moment a new vulnerability is disclosed, without needing to modify the scanner's actual code.

### 2.2 Why This Model Matters
- **Speed of updates** — when a new CVE drops, a template can be published and shared within hours, versus waiting for a new software release.
- **Community-driven scale** — thousands of checks can exist because many people contribute templates, not just one vendor's internal team.
- **Transparency** — since templates are human-readable, you can inspect *exactly* what a "finding" actually checked for, rather than trusting a black-box result.
- **Customization** — organizations can write private templates for their own internal applications' known quirks.

---

## 3. Security Assessment

In this context, **security assessment** refers to the broader process of **actively testing a web application** for real vulnerabilities — not just identifying technology (fingerprinting) or matching known signatures (template checks), but probing the application's actual behavior for exploitable weaknesses like injection flaws, exposed files, or insecure configurations.

### 3.1 How This Differs From Fingerprinting and Template Checks

| Approach | What It Answers |
|---|---|
| Fingerprinting | "What is this built with?" |
| Template-based checks | "Does this match a known, previously-documented issue?" |
| Security assessment (active testing) | "Does this application actually *behave* insecurely, right now, when tested directly?" |

A tool doing active security assessment doesn't just check for a known signature — it sends **crafted, sometimes malformed inputs** (e.g., a test SQL injection payload, a directory traversal attempt) and studies the *application's actual response* to determine if a real weakness exists, even for issues that have no published CVE.

### 3.2 Common Categories Covered
- **Injection flaws** — SQL injection, command injection (surface-level checks; deep testing comes on Day 12).
- **File/data exposure** — backup files, exposed `.git` folders, verbose error pages leaking stack traces.
- **Configuration weaknesses** — missing security headers, directory listing enabled, weak SSL/TLS setup.
- **Authentication weaknesses** — basic checks around login forms, session handling.

### 3.3 Why It Matters
- Moves beyond "is this software outdated" to "does this specific application, as configured, actually have exploitable weaknesses right now."
- Produces the evidence that a penetration test report is built on — not just a list of outdated versions, but confirmed, testable findings.

---

## 4. Tool: WhatWeb

**WhatWeb** is the fingerprinting specialist of this trio — a fast scanner purpose-built to identify a website's technology stack using a large plugin-based signature database.

### 4.1 How It Works
- WhatWeb sends one or more HTTP requests to the target and analyzes the response using **over 1,800 plugins**, each one knowing how to recognize a specific technology (a CMS, JS library, analytics tool, server software, etc.).
- Detection relies on combinations of: response headers, HTML content patterns, cookie names, meta tags, and JavaScript file references.
- Each plugin can report a **confidence level** and, where possible, an exact **version number**.
- WhatWeb supports different **aggression levels** — from a single passive request up to more thorough multi-request probing that follows links and tries common paths to gather more signatures.

### 4.2 What It's Used For
- The fast, standard **first step** of any web assessment — run before anything else to know what you're actually dealing with.
- Confirming a technology guess made during recon (e.g., verifying the Next.js/Vercel hint you might spot manually with `curl`, as in earlier days).
- Feeding accurate technology/version info into later CVE-lookup and template-based scanning steps.

### 4.3 Important Commands

```bash
whatweb example.com
```
The most basic scan — fingerprints the site using WhatWeb's default aggression level (light, single request). This is the command you'll reach for most often.

```bash
whatweb -v example.com
```
`-v` (verbose) shows **detailed plugin output**, including exactly which string/pattern triggered each detection — useful when you want to verify a finding isn't a false positive.

```bash
whatweb -a 3 example.com
```
`-a` sets the **aggression level** (1 = passive/stealthy, up to 4 = aggressive/thorough) — level 3 sends a few more requests to increase detection accuracy without being fully aggressive.

```bash
whatweb -a 4 example.com
```
Maximum aggression — follows more links and tries more paths for the most thorough fingerprint, at the cost of being slower and noisier.

```bash
whatweb --log-json=results.json example.com
```
Saves output in **structured JSON**, ideal for feeding into other tools/scripts (e.g., parsing with `jq` from Day 2) or into a larger automated pipeline.

```bash
whatweb -i targets.txt
```
`-i` reads a **list of targets from a file**, letting you fingerprint many sites/hosts in one run instead of one at a time.

```bash
whatweb --user-agent "Mozilla/5.0" example.com
```
Overrides the **User-Agent** string — some servers/WAFs respond differently based on this, so customizing it can sometimes reveal more (or avoid being blocked).

---

## 5. Tool: Nuclei

**Nuclei** is the template-based scanning specialist — an extremely fast, YAML-template-driven vulnerability scanner maintained by ProjectDiscovery, built around exactly the "template" concept described in Section 2.

### 5.1 How It Works
- Nuclei ships with (and auto-updates) a massive, community-maintained library of **templates** covering CVEs, exposed panels, misconfigurations, default credentials, and more.
- You point Nuclei at one or more targets; it loads a chosen set of templates and sends each template's defined request(s) to the target.
- Each template's **matcher** logic decides pass/fail based on the response — a matching template means a confirmed (or likely) finding.
- Because it's built for speed, Nuclei can run thousands of templates against many hosts in parallel — commonly used for scanning large numbers of targets quickly (bug bounty workflows, large asset inventories).

### 5.2 What It's Used For
- **Fast, broad vulnerability sweeps** across one host or thousands, using constantly-updated community templates — often the first active check run right after fingerprinting.
- Checking specifically for **known CVEs** with published exploit signatures — directly implementing Day 8's CVE detection concept, but at the web application layer and with far more granular, current templates than a general-purpose scanner.
- Continuous/scheduled scanning — since new templates are added constantly, re-running Nuclei periodically catches newly disclosed issues fast.

### 5.3 Important Commands

```bash
nuclei -u https://example.com
```
Runs Nuclei against a single target using its **default template set** — the standard starting command.

```bash
nuclei -list targets.txt
```
`-list` scans **multiple targets** from a file, one per line — used for scanning an entire asset inventory in one run.

```bash
nuclei -u https://example.com -t cves/
```
`-t` restricts the scan to a specific **template category/folder** — here, only templates checking for known CVEs.

```bash
nuclei -u https://example.com -t exposures/ -t misconfiguration/
```
Runs multiple specific template categories together — here, checks for exposed files/panels and general misconfigurations.

```bash
nuclei -u https://example.com -severity critical,high
```
`-severity` filters results to only the **most impactful findings**, cutting through noise when scanning at scale.

```bash
nuclei -u https://example.com -o nuclei_results.txt
```
`-o` saves scan output to a file — needed for building a report or feeding results into later analysis.

```bash
nuclei -u https://example.com -json-export results.json
```
Exports results in **structured JSON** instead of plain text, useful for programmatic parsing or integration with other tools.

```bash
nuclei -update-templates
```
Downloads the **latest template set** from the community repository — since new templates ship constantly (often the same day a CVE is disclosed), this should be run before every serious scan.

```bash
nuclei -u https://example.com -rate-limit 50
```
`-rate-limit` caps the number of requests per second — important for not overwhelming a fragile target or triggering unwanted alerts/blocks.

```bash
nuclei -u https://example.com -tags cve,exposed-panel
```
`-tags` filters templates by **tag** instead of folder path — a more flexible way to combine templates matching specific themes.

---

## 6. Tool: Wapiti

**Wapiti** is the active security-assessment specialist — a web application vulnerability scanner that works by actually **crawling** a site and then sending live attack-style test payloads to its forms and parameters, directly implementing the "does it actually behave insecurely" approach from Section 3.

### 6.1 How It Works
- Wapiti first **crawls (spiders)** the target site, discovering pages, forms, and URL parameters — building a map of every possible input point.
- For each discovered input (a form field, a URL parameter, a cookie), it sends a series of **test payloads** associated with different vulnerability modules (e.g., a basic SQL injection string, an XSS test string, a path traversal attempt).
- It analyzes the **application's actual response** to each payload — looking for behavior that confirms a real vulnerability (e.g., a database error message appearing after an SQLi payload), not just a version-based guess.
- Findings are categorized by vulnerability type and can be exported as a formal report.

### 6.2 What It's Used For
- **Black-box web application testing** — testing the live application's behavior directly, complementing Nuclei's known-signature approach and WhatWeb's passive fingerprinting.
- A lighter-weight, faster alternative to a full manual test (or to heavier tools like Burp Suite/OWASP ZAP, covered on Day 10) when you need a quick automated pass across many input points.
- Catching custom/unique vulnerabilities in an application's own code — issues that wouldn't have a public CVE or a Nuclei template, because they're specific to this one app.

### 6.3 Important Commands

```bash
wapiti -u https://example.com
```
The most basic scan — crawls the target and runs Wapiti's default set of vulnerability modules against every discovered input.

```bash
wapiti -u https://example.com --scope domain
```
`--scope` controls how far the crawler follows links — `domain` keeps it within the same domain rather than wandering off to external sites it links to.

```bash
wapiti -u https://example.com -m sql,xss
```
`-m` (modules) restricts the scan to **specific vulnerability types** — here, SQL injection and XSS checks only, instead of running every module.

```bash
wapiti -u https://example.com -m all
```
Runs **every available module** — the most thorough (and slowest) option, covering the full range of checks Wapiti supports.

```bash
wapiti -u https://example.com --depth 3
```
`--depth` controls how many **link-levels deep** the crawler goes from the starting page — deeper crawls find more input points but take longer.

```bash
wapiti -u https://example.com -o report_folder -f html
```
`-o` sets the output location and `-f` sets the **report format** (`html`, `json`, `xml`, `txt`) — essential for turning findings into a shareable deliverable.

```bash
wapiti -u https://example.com --auth-type post -a "user%password"
```
Supplies **authentication credentials** so Wapiti can crawl and test pages that require a login — critical for assessing anything beyond a public-facing homepage.

```bash
wapiti -u https://example.com --max-scan-time 600
```
`--max-scan-time` caps the total scan duration (in seconds) — useful for keeping automated/scheduled scans within a predictable time budget.

```bash
wapiti -u https://example.com -v 2
```
`-v` sets the **verbosity level** — level 2 shows more detail about what's being tested in real time, useful for understanding exactly what the scan is doing as it runs.

---

## 7. Quick Cheat Sheet

| Task | Command |
|---|---|
| Identify a site's tech stack | `whatweb -a 3 example.com` |
| Save fingerprint results as JSON | `whatweb --log-json=results.json example.com` |
| Run known-CVE template checks | `nuclei -u https://example.com -t cves/` |
| Update Nuclei's template library | `nuclei -update-templates` |
| Filter Nuclei to high-impact findings only | `nuclei -u https://example.com -severity critical,high` |
| Full active web app scan | `wapiti -u https://example.com -m all` |
| Focused SQLi/XSS scan | `wapiti -u https://example.com -m sql,xss` |
| Save Wapiti report as HTML | `wapiti -u https://example.com -o report_folder -f html` |

**Typical Day 9 Assessment Workflow (lab/authorized environment):**
```bash
whatweb -a 3 https://target.com --log-json=recon/whatweb.json     # 1. Fingerprint the stack first
nuclei -update-templates                                            # 2. Ensure templates are current
nuclei -u https://target.com -severity critical,high,medium -o nuclei_results.txt   # 3. Fast known-issue sweep
wapiti -u https://target.com -m all -o wapiti_report -f html           # 4. Active behavioral testing
# 5. Cross-reference all three outputs into one prioritized findings list
```

---

### Practice Suggestions for Day 9
1. Run WhatWeb against a lab target at increasing aggression levels (`-a 1` through `-a 4`) and note how many more technologies each level detects.
2. Open one Nuclei template file (`.yaml`) from its templates directory and identify its `id`, `matchers`, and `severity` fields — connect this back to Section 2's concept.
3. Run Nuclei against a deliberately vulnerable lab app (e.g., DVWA) filtered to `-severity critical,high` and review each finding's template source.
4. Run Wapiti against the same lab app with `-m all` and compare its findings to Nuclei's — which issues did only Wapiti catch (since they're behavior-based, not signature-based)?
5. Write a short comparison table: for each finding across all three tools, note whether it came from fingerprinting, a known template/signature, or active behavioral testing.
