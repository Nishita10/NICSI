
# Day 8 — Vulnerability Assessment & Web Security


**Tools**
● OpenVAS (Greenbone) ● Nikto

**Topics**
● Infrastructure Scanning ● CVE Detection

A complete reference covering infrastructure scanning and CVE detection — using OpenVAS (Greenbone) and Nikto. (Topics are explained first, followed by the tools — each explained by how it works and what it's used for, then its most important commands.)

> ⚠️ **Ethical Note:** Vulnerability scanning is more intrusive than reconnaissance — it actively probes services, sends crafted requests, and can occasionally trigger crashes or false alarms on fragile systems. Always run these tools only against systems you own or have **explicit written authorization** to test, and ideally in an isolated lab first.

---

## Table of Contents
1. [Infrastructure Scanning](#1-infrastructure-scanning)
2. [CVE Detection](#2-cve-detection)
3. [Tool: OpenVAS (Greenbone)](#3-tool-openvas-greenbone)
4. [Tool: Nikto](#4-tool-nikto)
5. [Quick Cheat Sheet](#5-quick-cheat-sheet)

---

## 1. Infrastructure Scanning

**Infrastructure scanning** is the process of systematically examining hosts, servers, network devices, and services to find security weaknesses — misconfigurations, outdated software, weak protocols, missing patches, exposed services — before an attacker does.

### 1.1 Where This Fits in the Bigger Picture
Reconnaissance (Days 1–5) told us *what exists* — hosts, subdomains, open ports, running services. Infrastructure scanning is the next logical step: it asks *is any of what we found actually vulnerable?*

```
Recon (Days 1-5)         →  "Here's what's out there"
Infrastructure Scanning    →  "Is any of it exploitable?"
```

### 1.2 What Gets Checked
- **Missing patches** — software versions with known, unfixed security holes.
- **Default/weak credentials** — services still using factory-default logins.
- **Insecure configurations** — services exposing more than they should (e.g., anonymous FTP, verbose error messages, directory listing enabled).
- **Outdated protocols** — servers still supporting deprecated, insecure protocol versions (e.g., old SSL/TLS versions, SMBv1).
- **Open/unnecessary services** — ports running services that shouldn't be internet-facing at all.

### 1.3 Types of Infrastructure Scans

| Type | Description |
|---|---|
| **Unauthenticated scan** | Scanner has no credentials — sees exactly what an external attacker would see |
| **Authenticated/credentialed scan** | Scanner logs in with valid credentials — can inspect installed packages, patch levels, and local configs far more accurately |
| **Network-based scan** | Probes services remotely over the network (most common) |
| **Agent-based scan** | A lightweight agent runs directly on the host and reports back — used at scale in enterprise environments |

### 1.4 Why It Matters
- Turns a long list of open ports/services (from Day 3's Nmap work) into a **prioritized list of actual risks**.
- Forms the evidence base for a vulnerability assessment report — the deliverable that tells an organization exactly what to patch first.

---

## 2. CVE Detection

A **CVE (Common Vulnerabilities and Exposures)** is a standardized, publicly cataloged identifier for a specific known security vulnerability — e.g., `CVE-2021-44228` (the "Log4Shell" vulnerability). **CVE detection** is the process of matching what's actually running on a target against this public database to see which known vulnerabilities apply.

### 2.1 How CVE Matching Works
1. The scanner first identifies the exact **software and version** running on a service (similar to Day 3's service detection, but deeper).
2. It compares that version against vulnerability databases — the **NVD (National Vulnerability Database)**, vendor advisories, and the scanner's own signature feed.
3. Each match is reported with its **CVE ID**, a description, and typically a **severity score**.

### 2.2 CVSS — Understanding Severity
Most CVEs are scored using **CVSS (Common Vulnerability Scoring System)**, a 0–10 scale:

| CVSS Score | Severity |
|---|---|
| 0.1 – 3.9 | Low |
| 4.0 – 6.9 | Medium |
| 7.0 – 8.9 | High |
| 9.0 – 10.0 | Critical |

The score factors in things like: is it exploitable remotely? does it need authentication? does it impact confidentiality, integrity, or availability?

### 2.3 Why It Matters
- Turns "this server is running Apache 2.4.29" into "this server is running Apache 2.4.29, which is vulnerable to CVE-2021-XXXXX, a **Critical**, remotely-exploitable issue" — the difference between a fact and an actionable finding.
- CVE IDs give a **common language** — a specific CVE means the exact same thing to every security team, vendor, and researcher worldwide, unlike vague descriptions.
- Prioritization: with limited time to patch, teams fix **Critical/High** CVEs on internet-facing systems first.

---

## 3. Tool: OpenVAS (Greenbone)

**OpenVAS** (now part of the **Greenbone Vulnerability Management** suite) is a full-featured, open-source **vulnerability scanner** — one of the most comprehensive free tools for infrastructure scanning and CVE detection, capable of scanning entire networks against a constantly-updated feed of tens of thousands of vulnerability tests.

### 3.1 How It Works
OpenVAS runs as a **client-server architecture**, not a single command-line tool:
- A **scan engine** runs continuously as a service, maintaining a feed of **NVTs (Network Vulnerability Tests)** — individual scripts, each checking for one specific vulnerability or misconfiguration.
- You define a **Target** (an IP, range, or hostname) and a **Scan Config** (which NVTs to run — e.g., "Full and fast" vs. a targeted subset).
- The engine launches the scan, running relevant NVTs against every open port/service it finds — first doing its own light port/service discovery, then layering vulnerability checks on top.
- Results are matched against **CVE data** automatically, and each finding is given a **severity score (CVSS)**, a description, and — often — remediation guidance.
- Everything is managed through a **web-based interface (Greenbone Security Assistant)** rather than raw terminal commands, though a command-line client (`gvm-cli`) also exists for automation.

### 3.2 What It's Used For
- **Full infrastructure vulnerability assessments** — the standard tool for scanning a network of servers and getting a prioritized, CVE-backed report.
- Scheduled, recurring scans in an organization to catch newly-disclosed vulnerabilities as soon as the NVT feed updates.
- Producing formal vulnerability assessment reports for clients or internal security teams.

### 3.3 Important Commands

```bash
sudo gvm-setup
```
Runs the **initial setup** — downloads the vulnerability feed, initializes the database, and creates the admin account. This is a one-time setup step (can take a while the first time, since the NVT feed is large).

```bash
sudo gvm-check-setup
```
Verifies that all OpenVAS/Greenbone components (scanner, manager, feed) are correctly installed and running — the standard troubleshooting first step if something isn't working.

```bash
sudo gvm-start
```
Starts all Greenbone services (manager, scanner, web UI) — needed before you can log in or run any scan.

```bash
sudo gvm-stop
```
Stops all Greenbone services — useful when freeing up system resources or before an update.

```bash
sudo runuser -u _gvm -- greenbone-feed-sync
```
Manually **syncs the latest vulnerability feed** (NVTs, CVEs) — since new vulnerabilities are disclosed constantly, keeping this feed current is essential for accurate detection.

```bash
gvm-cli socket --xml "<get_version/>"
```
A command-line sanity check confirming the scanner is reachable and reporting its version — useful for scripting/automation checks.

**Typical usage is via the web UI** (`https://localhost:9392` by default) rather than the CLI:
1. **Log in** to the Greenbone Security Assistant.
2. **Configuration → Targets → New Target** — enter the IP/hostname to scan.
3. **Scans → Tasks → New Task** — select the target and a scan config (e.g., "Full and fast").
4. **Start the scan** and monitor progress from the Tasks dashboard.
5. **Review Results** — each finding lists its CVE(s), CVSS severity, affected host/port, and suggested remediation.
6. **Export a report** (PDF/HTML/XML) for documentation.

---

## 4. Tool: Nikto

**Nikto** is a fast, focused **web server scanner** — unlike OpenVAS's broad infrastructure coverage, Nikto specializes specifically in web servers, checking for dangerous files, outdated server software, misconfigurations, and thousands of known web-specific vulnerability signatures.

### 4.1 How It Works
Nikto works as a straightforward command-line tool (no server/client setup required):
- It sends a large series of **HTTP requests** to the target web server, each testing for a specific known issue — e.g., requesting `/admin/`, checking for default files left over from installers, testing for outdated server banners.
- It compares the **server's response headers/banner** (e.g., `Server: Apache/2.4.29`) against its database of known-vulnerable versions.
- It flags **dangerous or interesting files/directories** that respond with a "found" status instead of a 404.
- Because it's signature-based and lightweight, a full Nikto scan typically finishes in minutes, rather than the longer runtimes typical of a full OpenVAS sweep.

### 4.2 What It's Used For
- **Quick, targeted web server assessments** — the go-to first step before a deeper web application test (e.g., before bringing in Burp Suite or ZAP on Day 10).
- Spotting outdated server software, missing security headers, and leftover default/dangerous files.
- A fast complementary check alongside OpenVAS — Nikto for the web layer specifically, OpenVAS for the broader infrastructure.

### 4.3 Important Commands

```bash
nikto -h https://example.com
```
The most basic scan — `-h` (host) points Nikto at a target URL and runs its full default check suite. This is the command you'll use most often.

```bash
nikto -h example.com -p 80,443
```
`-p` specifies which **ports** to scan — useful when a server runs on non-standard ports, or to limit/expand scope beyond the default.

```bash
nikto -h example.com -ssl
```
Forces Nikto to connect over **SSL/TLS**, useful when scanning an HTTPS-only service that Nikto might not auto-detect correctly.

```bash
nikto -h example.com -o report.html -Format htm
```
`-o` saves output to a file, and `-Format` sets its type (`htm`, `csv`, `txt`, `xml`) — essential for turning scan results into a shareable report.

```bash
nikto -h example.com -Tuning 1,2,3
```
`-Tuning` limits the scan to **specific test categories** (e.g., 1 = file upload, 2 = misconfiguration, 3 = information disclosure) — useful for a faster, more focused scan instead of running every check.

```bash
nikto -h example.com -Display V
```
`-Display V` shows **verbose output**, including redirects and additional detail as the scan runs — helpful for understanding exactly what Nikto is doing in real time.

```bash
nikto -h example.com -useragent "Mozilla/5.0"
```
Overrides the **User-Agent** string Nikto sends — some servers/WAFs behave differently (or block scanning tools) based on the User-Agent, so this can help get more representative results.

```bash
nikto -h example.com -timeout 10
```
Sets a **timeout** (in seconds) per request — useful for slow targets, preventing the scan from hanging indefinitely on unresponsive requests.

```bash
nikto -h example.com -evasion 1
```
Enables an **IDS evasion technique** (several numbered options exist) — primarily useful for testing whether your own organization's intrusion detection system actually catches scanning activity.

```bash
nikto -update
```
Updates Nikto's **vulnerability/plugin database** — like OpenVAS's feed sync, this should be run regularly so scans check against the latest known issues.

---

## 5. Quick Cheat Sheet

| Task | Command |
|---|---|
| Set up OpenVAS (one-time) | `sudo gvm-setup` |
| Start/stop Greenbone services | `sudo gvm-start` / `sudo gvm-stop` |
| Update vulnerability feed | `sudo runuser -u _gvm -- greenbone-feed-sync` |
| Run a full infrastructure scan | Web UI → New Target → New Task → "Full and fast" |
| Basic web server scan | `nikto -h https://example.com` |
| Save Nikto results as a report | `nikto -h example.com -o report.html -Format htm` |
| Focused/faster Nikto scan | `nikto -h example.com -Tuning 1,2,3` |
| Update Nikto's database | `nikto -update` |

**Typical Vulnerability Assessment Workflow (lab/authorized environment):**
```bash
nikto -update                                     # 1. Ensure Nikto's checks are current
nikto -h https://target.com -o nikto_report.html -Format htm   # 2. Fast web-layer scan first
sudo runuser -u _gvm -- greenbone-feed-sync           # 3. Ensure OpenVAS's CVE feed is current
# 4. In the Greenbone web UI: create Target -> create Task -> run "Full and fast" scan
# 5. Cross-reference Nikto's web findings against OpenVAS's broader infrastructure findings
# 6. Prioritize remediation by CVSS severity (Critical/High first)
```

---

### Practice Suggestions for Day 8
1. Install OpenVAS/Greenbone in a lab VM and run `gvm-check-setup` until every component reports OK.
2. Sync the feed, then scan a deliberately vulnerable VM (e.g., Metasploitable) with a "Full and fast" scan config.
3. Review the results and pick the 3 highest-CVSS findings — look up their CVE IDs and read the official descriptions (e.g., on the NVD website).
4. Run Nikto against the same lab VM's web server and compare what it finds versus OpenVAS's web-related findings.
5. Re-run Nikto with `-Tuning` to isolate just "information disclosure" checks and see how the scan time and result count change.
6. Export both an OpenVAS report and a Nikto HTML report, and write a short summary combining both into one prioritized findings list.
