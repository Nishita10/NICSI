Day 11 — Web Security (Content Discovery & Enumeration)
Tools ● Gobuster ● Dirsearch ● ffuf ● Feroxbuster
Topics ● Content Discovery ● Directory Enumeration
A complete reference covering the discovery of hidden files, directories, and endpoints on a web server — using Gobuster, Dirsearch, ffuf, and Feroxbuster. 

## Topics

### 1. Content Discovery

**What it is:** The process of finding files, endpoints, and resources on a web application that aren't linked from the visible UI — backup files, admin panels, API endpoints, config files, old versions of pages, documentation, etc. These "hidden" resources often lack the same security scrutiny as the main application flow, making them common sources of information disclosure or direct vulnerabilities.

**Why it matters:** A page might not link to `/backup.zip`, `/admin/`, `/.git/`, or `/api/v2/debug`, but if the resource exists on the server and is reachable, an attacker who finds it can access it directly. Content discovery is about systematically uncovering that "unlinked but accessible" surface area.

**What's typically searched for:**
- Admin/login panels (`/admin`, `/manager`, `/wp-admin`)
- Backup and archive files (`.bak`, `.zip`, `.tar.gz`, `~` backups)
- Configuration files (`.env`, `config.php.bak`, `web.config`)
- Version control leftovers (`.git/`, `.svn/`)
- API documentation/endpoints (`/api/`, `/swagger.json`, `/graphql`)
- Debug/test pages (`/test.php`, `/debug`, `/phpinfo.php`)
- Source code and log files (`.log`, `.sql`, `.old`)

**Method:** Uses a wordlist (common file/directory names, e.g. SecLists) and sends a request for each combination of base URL + wordlist entry, then inspects the HTTP status code and response size to determine what actually exists.

---

### 2. Directory Enumeration

**What it is:** A specific subset of content discovery focused on finding *directories* (as opposed to individual files) — recursively walking a target's structure to map out the full directory tree, since directories often contain further hidden files or even directory listings that expose everything inside.

**Core technique — brute forcing:**
1. Take a wordlist of common directory/file names.
2. Send an HTTP request for each: `http://target.com/<wordlist-entry>`.
3. Filter responses by status code:
   - `200 OK` → exists and accessible.
   - `301/302` → redirect, often meaning a directory exists (e.g., `/admin` → `/admin/`).
   - `403 Forbidden` → exists but access is blocked (still useful info).
   - `404 Not Found` → doesn't exist (default, to be filtered out).
4. **Recursion:** once a directory is confirmed, re-run the same wordlist inside it to go deeper (`/admin/` → `/admin/users/`).
5. **Extension fuzzing:** append common extensions (`.php`, `.html`, `.bak`, `.js`) to wordlist entries to catch files, not just directories.

**Key considerations:**
- **Wordlist quality matters** more than the tool — SecLists' `raft-*` and `common.txt` lists are industry standard.
- **Rate limiting / thread count:** too aggressive scanning can crash a server, trip WAFs/IDS, or get an IP banned — throttle appropriately.
- **False positives:** some servers return `200 OK` for every request (soft-404 pages); tools must be told to filter by response size/content to avoid this.
- **Case sensitivity and file extensions:** vary by server OS/config (Linux is case-sensitive, Windows/IIS often isn't).

---

## Tools

### 1. Gobuster

**How it works:** Gobuster is a fast, Go-based brute-forcing tool with multiple modes: `dir` (directory/file brute-forcing), `dns` (subdomain enumeration), `vhost` (virtual host discovery), `s3`/`gcs` (cloud bucket enumeration). For `dir` mode, it sends concurrent HTTP requests combining the base URL with each wordlist entry and reports based on status code filters.

**Purpose:** Lightweight, fast, and simple — a good default choice for straightforward directory/file brute-forcing and subdomain enumeration.

**Important commands:**

```bash
# Basic directory brute-force
gobuster dir -u http://target.com -w /usr/share/wordlists/dirb/common.txt

# Specify file extensions to check
gobuster dir -u http://target.com -w wordlist.txt -x php,html,bak,txt

# Set thread count (default 10)
gobuster dir -u http://target.com -w wordlist.txt -t 50

# Filter by status codes (only show these)
gobuster dir -u http://target.com -w wordlist.txt -s 200,204,301,302,403

# Exclude specific status codes
gobuster dir -u http://target.com -w wordlist.txt -b 404

# Add custom headers / cookies (for authenticated scanning)
gobuster dir -u http://target.com -w wordlist.txt -H "Cookie: session=abc123"

# Subdomain enumeration (DNS mode)
gobuster dns -d target.com -w subdomains.txt

# Virtual host discovery
gobuster vhost -u http://target.com -w vhosts.txt
```

**Notes:** Gobuster does not follow redirects or crawl recursively by default — it's a flat brute-forcer; you re-run it manually on discovered subdirectories.

---

### 2. Dirsearch

**How it works:** Dirsearch is a Python-based directory/file brute-forcer with built-in recursive scanning, smart extension handling, and a large default wordlist. It automatically detects and filters out "soft-404" responses (pages that return `200 OK` for non-existent paths) by comparing response signatures.

**Purpose:** Favored for its recursive scanning out of the box and good default reporting — often considered more "batteries-included" than Gobuster for quick assessments.

**Important commands:**

```bash
# Basic scan
python3 dirsearch.py -u http://target.com

# Specify a wordlist
python3 dirsearch.py -u http://target.com -w wordlist.txt

# Specify extensions
python3 dirsearch.py -u http://target.com -e php,html,js,bak

# Enable recursive scanning (scan discovered directories automatically)
python3 dirsearch.py -u http://target.com -r

# Set thread count
python3 dirsearch.py -u http://target.com -t 50

# Filter/exclude status codes
python3 dirsearch.py -u http://target.com -i 200,204,301,302,403
python3 dirsearch.py -u http://target.com -x 404,500

# Output results to a file
python3 dirsearch.py -u http://target.com -o results.txt --format=simple

# Add custom headers/cookies
python3 dirsearch.py -u http://target.com --header="Cookie: session=abc123"
```

**Notes:** The `-r` recursive flag combined with a large wordlist can be slow — tune `--recursion-depth` if scans take too long on deep sites.

---

### 3. ffuf (Fuzz Faster U Fool)

**How it works:** ffuf is a highly flexible, high-performance fuzzing tool built around a generic `FUZZ` keyword placeholder — meaning it's not limited to directory brute-forcing. The `FUZZ` marker can be placed anywhere in the URL, headers, POST body, or even the Host header, making it useful for directory enumeration, parameter fuzzing, subdomain/vhost discovery, and more, all with the same engine.

**Purpose:** The go-to tool when discovery needs go beyond simple directory brute-forcing — e.g., fuzzing parameter names/values, headers, or virtual hosts, all using one consistent syntax.

**Important commands:**

```bash
# Basic directory brute-force (FUZZ marks the injection point)
ffuf -u http://target.com/FUZZ -w wordlist.txt

# Specify extensions
ffuf -u http://target.com/FUZZ -w wordlist.txt -e .php,.html,.bak

# Filter by response size (hide soft-404 pages of a known size)
ffuf -u http://target.com/FUZZ -w wordlist.txt -fs 1234

# Filter by status code
ffuf -u http://target.com/FUZZ -w wordlist.txt -mc 200,204,301,302,403

# Fuzz a parameter value (e.g., testing an ID parameter)
ffuf -u http://target.com/item?id=FUZZ -w ids.txt

# Fuzz POST data
ffuf -u http://target.com/login -X POST -d "username=admin&password=FUZZ" -w passwords.txt

# Virtual host discovery (fuzz the Host header)
ffuf -u http://target.com -H "Host: FUZZ.target.com" -w subdomains.txt -fs 1234

# Recursive scanning
ffuf -u http://target.com/FUZZ -w wordlist.txt -recursion -recursion-depth 2

# Set thread count and delay (rate limiting)
ffuf -u http://target.com/FUZZ -w wordlist.txt -t 40 -p 0.1
```

**Notes:** ffuf's `-fs` (filter size), `-fw` (filter word count), and `-fc` (filter status code) flags are essential for cutting through noisy false positives — always do a baseline request first to know what a "not found" response looks like.

---

### 4. Feroxbuster

**How it works:** Feroxbuster is a Rust-based recursive content discovery tool, conceptually similar to Dirsearch but built for speed and heavy-duty recursive scanning by default. It automatically detects wildcard responses (soft-404s), supports recursion out of the box, and can auto-filter based on response similarity.

**Purpose:** Built for large-scale, deep recursive enumeration efficiently — a strong choice when scanning large sites where recursion depth and speed both matter.

**Important commands:**

```bash
# Basic scan
feroxbuster -u http://target.com -w wordlist.txt

# Specify extensions
feroxbuster -u http://target.com -w wordlist.txt -x php,html,bak

# Control recursion depth (0 = unlimited)
feroxbuster -u http://target.com -w wordlist.txt --depth 2

# Set thread count
feroxbuster -u http://target.com -w wordlist.txt -t 50

# Filter status codes
feroxbuster -u http://target.com -w wordlist.txt -s 200,301,302,403

# Exclude specific status codes
feroxbuster -u http://target.com -w wordlist.txt -C 404,500

# Silent/quiet mode (only show results, minimal noise)
feroxbuster -u http://target.com -w wordlist.txt -q

# Output results to file
feroxbuster -u http://target.com -w wordlist.txt -o results.txt

# Add headers/cookies for authenticated scans
feroxbuster -u http://target.com -w wordlist.txt -H "Cookie: session=abc123"
```

**Notes:** Feroxbuster recurses by default (unlike Gobuster), so on large applications it's worth setting `--depth` explicitly to avoid runaway scan times.

---

### Quick Summary Table

| Tool | Language | Recursive by Default | Standout Feature |
|---|---|---|---|
| Gobuster | Go | No | Fast, simple, multi-mode (dir/dns/vhost/s3) |
| Dirsearch | Python | No (flag-enabled) | Smart soft-404 detection, easy reporting |
| ffuf | Go | No (flag-enabled) | Generic `FUZZ` keyword — works anywhere (URL, headers, body) |
| Feroxbuster | Rust | Yes | Built for deep, fast recursive scans by default |

Let me know if you'd like Day 13 next, continuing into a related area (e.g., authentication testing, API security, or advanced automation).
