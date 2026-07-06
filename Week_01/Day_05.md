# Day 5 — Advanced Reconnaissance


**Tools**
● theHarvester ● Recon-ng ● Amass ● Maltego CE

**Topics**
● Email Enumeration ● Subdomain Discovery ● Asset Mapping

A complete reference covering email enumeration, subdomain discovery, and asset mapping — using theHarvester, Recon-ng, Amass, and Maltego CE. 

> ⚠️ **Ethical Note:** These tools mostly gather **passive/OSINT data** — information already public on the internet (search engines, certificate logs, public APIs). That makes them low-risk, but the moment you start actively probing a target's infrastructure (port scans, brute-forcing), you're back in "authorization required" territory. Always practice on your own domains or authorized lab targets.

---

## Table of Contents
1. [Email Enumeration](#1-email-enumeration)
2. [Subdomain Discovery](#2-subdomain-discovery)
3. [Asset Mapping](#3-asset-mapping)
4. [Tool: theHarvester](#4-tool-theharvester)
5. [Tool: Recon-ng](#5-tool-recon-ng)
6. [Tool: Amass](#6-tool-amass)
7. [Tool: Maltego CE](#7-tool-maltego-ce)
8. [Quick Cheat Sheet](#8-quick-cheat-sheet)

---

## 1. Email Enumeration

**Email enumeration** is the process of discovering email addresses associated with a target organization — usually as a first step toward understanding its people, structure, and potential attack surface (phishing, credential stuffing, social engineering).

### 1.1 Where This Data Comes From
Since directly asking a company "give me your employees' emails" isn't an option, enumeration tools pull from **publicly indexed sources**:
- **Search engines** (Google, Bing, DuckDuckGo) — cached pages sometimes expose emails in contact pages, PDFs, or old blog posts.
- **Public code repositories** (GitHub) — developers accidentally commit config files with real emails.
- **Data breach dumps** (aggregated, publicly searchable databases) — old leaked emails tied to a domain.
- **PGP key servers** — people who publish public encryption keys often list their email.
- **Social media & professional networks** (LinkedIn) — naming conventions can often be inferred (e.g., `firstname.lastname@company.com`).
- **Certificate Transparency logs** — SSL certs sometimes list internal email contacts.

### 1.2 Why It Matters
- Emails often reveal an organization's **naming convention** (e.g. `j.smith@`, `jsmith@`, `smith.j@`), which lets an assessor (or attacker) predict *other* employees' addresses even without seeing them directly.
- Feeds directly into **phishing simulations**, **password spraying** (using known usernames), and **social engineering** assessments.
- Helps build an org chart — which is useful both offensively and for defenders doing their own exposure audit.

---

## 2. Subdomain Discovery

**Subdomain discovery** is the process of finding all the subdomains (`mail.`, `dev.`, `api.`, `vpn.`, `staging.`) that belong to a target's main domain — expanding a single domain into a full map of an organization's internet-facing presence.

### 2.1 Why Subdomains Matter So Much
A company's main website (`example.com`) is usually well-secured and closely monitored. Its **forgotten subdomains** often aren't:
- `dev.example.com` — a development server, sometimes with debug mode on or weak/no authentication.
- `old-admin.example.com` — a legacy panel nobody remembered to decommission.
- `test-api.example.com` — an API without the same rate-limiting/auth as production.

In real-world assessments and breaches, **subdomain sprawl** is one of the most common ways attackers find a soft entry point into an otherwise hardened organization.

### 2.2 Discovery Techniques

| Technique | Description |
|---|---|
| **Passive (OSINT)** | Pull subdomains from Certificate Transparency logs (crt.sh), search engines, DNS aggregator APIs — no packets sent to the target |
| **Active brute-forcing** | Try a wordlist of common subdomain names against the target's DNS |
| **Zone transfer** | Ask nameservers directly (works only if misconfigured — see Day 4) |
| **Web archives** | Check historical crawls (e.g., Wayback Machine) for subdomains that existed in the past |

### 2.3 Why It Matters
- Massively expands the visible **attack surface** compared to just the root domain.
- Reveals internal naming conventions, tech stack hints (e.g. `jenkins.example.com` reveals CI/CD tooling), and organizational structure.

---

## 3. Asset Mapping

**Asset mapping** is the process of piecing together *everything* discovered (domains, subdomains, IPs, emails, employees, technologies, social media accounts) into a single connected picture of an organization's digital footprint — this is the "putting it all together" phase of reconnaissance.

### 3.1 What Gets Mapped
- **Infrastructure assets** — domains, subdomains, IP ranges, cloud services (AWS/Azure buckets), hosting providers.
- **Human assets** — employee names, roles, email addresses, social media presence.
- **Relationships** — which IPs belong to which subdomains, which subdomains share hosting, which employees belong to which department.
- **Technology footprint** — CMS platforms, web frameworks, third-party services in use (inferred from DNS TXT records, headers, subdomain names).

### 3.2 Why Visualize It?
Raw text lists of 200 subdomains and 50 emails are hard to reason about. Asset mapping tools (especially **Maltego**) turn this into a **graph/network diagram** — nodes (entities) connected by edges (relationships) — making patterns, clusters, and unexpected connections visually obvious (e.g., spotting that three "unrelated" companies actually share the same hosting IP).

### 3.3 Why It Matters
- Gives both attackers and defenders a **single consolidated view** of an organization's real exposure — far more useful than a dozen separate tool outputs.
- Often reveals **unexpected connections** — shared infrastructure, shadow IT, or forgotten legacy assets tied to the same organization.
- Forms the backbone of a professional reconnaissance report deliverable.

---

## 4. Tool: theHarvester

**theHarvester** is a passive OSINT tool built specifically for gathering **emails, subdomains, hostnames, employee names, and IPs** from public sources — it's often the very first tool run in a recon phase because it's fast and entirely passive.

### 4.1 How It Works
theHarvester doesn't scan or probe the target at all — it queries **third-party public sources** on the target's behalf and aggregates whatever they already know:
- **Search engines** (Google, Bing, DuckDuckGo) — crawled/cached pages mentioning the domain.
- **Certificate Transparency logs** (crt.sh) — every SSL certificate ever issued for the domain, which reveals subdomains.
- **PGP key servers** — public encryption keys often list an email and organization.
- **Other OSINT APIs** (Shodan, Hunter.io, etc., depending on version/API keys configured).

It sends your query to each selected source, parses the responses, deduplicates the results, and prints a consolidated list of emails, hosts, subdomains, and IPs.

### 4.2 What It's Used For
- The **first pass** of any recon engagement — quick, safe, and needs no authorization since it's fully passive.
- Building an initial list of employee emails for phishing-simulation or password-spraying planning.
- Getting a fast, rough subdomain list before running deeper active tools like Amass.

### 4.3 Important Commands

```bash
theHarvester -d example.com -b google
```
Runs a basic scan against `example.com` using **Google** as the data source. `-b` specifies the source/engine to query — this is the core command you'll use constantly.

```bash
theHarvester -d example.com -b all
```
Queries **every supported source** at once (Google, Bing, crt.sh, Yahoo, etc.) — broadest possible passive sweep, but slower and more likely to hit rate limits.

```bash
theHarvester -d example.com -b bing
```
Uses **Bing** as the source — useful as a secondary source since different search engines index different pages, so results won't always overlap with Google.

```bash
theHarvester -d example.com -b crtsh
```
Queries **crt.sh** (Certificate Transparency logs) — extremely reliable for finding subdomains, since every SSL certificate issued for a subdomain gets permanently logged publicly.

```bash
theHarvester -d example.com -l 500 -b google
```
`-l` limits the **number of results** fetched (here, 500) — useful to control runtime and avoid overwhelming output on large domains.

```bash
theHarvester -d example.com -b google -f report.html
```
`-f` **saves output to a file** — here an HTML report — so results can be reviewed/shared later instead of just scrolling the terminal.

```bash
theHarvester -d example.com -b google -f report.xml
```
Same as above but exports as **XML**, useful for feeding results into other automation scripts or tools programmatically.

```bash
theHarvester -d example.com -b linkedin
```
Searches **LinkedIn** specifically — good for gathering employee names and job titles (useful for social engineering assessments and org-chart mapping).

```bash
theHarvester -d example.com -b google -s
```
`-s` enables a **screenshot** option in some versions/plugins — captures visual evidence of found web assets (varies by version/build).

```bash
theHarvester -h
```
Displays the **help menu** — lists all supported data sources (`-b` options) and flags, since available sources change between versions/updates.

---

## 5. Tool: Recon-ng

**Recon-ng** is a full-featured, **modular** web reconnaissance framework — think "Metasploit, but for OSINT." It has a command-line interface similar to Metasploit, with **modules** for different recon tasks organized into **workspaces**.

### 5.1 How It Works
Recon-ng runs as an interactive shell, not a single-shot command. The workflow is always:
1. Create/select a **workspace** (an isolated project database for one target).
2. Search the **marketplace** for a relevant module and install it.
3. **Load** the module and set its required **options** (e.g., the target domain).
4. **Run** it — the module queries its data source (an API, a scraping technique, a breach database, etc.) and stores structured results (hosts, contacts, credentials, etc.) directly into the workspace's database.
5. Chain more modules together, each one enriching the same dataset — e.g., one module finds hosts, another resolves them to IPs, another finds contacts tied to those hosts.

Because every module's output feeds the same underlying database, Recon-ng is built for **layered, incremental** recon rather than one-off lookups.

### 5.2 What It's Used For
- Running **repeatable, structured OSINT workflows** instead of manually juggling many separate tools.
- Centralizing all findings (hosts, contacts, credentials, social profiles) about one target into a single queryable database.
- Automating recon pipelines — modules can be scripted/chained for repeated engagements.

### 5.3 Important Commands

```bash
recon-ng
```
Launches the **Recon-ng interactive console** — the starting point for every session; everything else happens inside this shell.

```bash
workspaces create example_recon
```
Creates a new **workspace** — an isolated project/database to store all data collected about one specific target, keeping engagements separate.

```bash
workspaces select example_recon
```
Switches into a previously created workspace to continue or review a specific engagement's data.

```bash
marketplace search domain
```
Searches the **module marketplace** for available modules matching a keyword (e.g., "domain") — Recon-ng's modules must be installed before use.

```bash
marketplace install recon/domains-hosts/hackertarget
```
**Installs a specific module** — here, one that finds hosts/subdomains via the HackerTarget API. Modules are organized as `category/input-output/module_name`.

```bash
modules load recon/domains-hosts/hackertarget
```
**Loads** an installed module into the current session so its options can be configured and run.

```bash
options set SOURCE example.com
```
Sets a module's **input option** — here, telling the loaded module which domain to target. Every module has its own required options (check with `info`).

```bash
run
```
**Executes** the currently loaded module against the configured options — the actual "go" command after setup.

```bash
show hosts
```
Displays all **hosts/subdomains** discovered so far and stored in the current workspace's database — the running result set.

```bash
show contacts
```
Displays all **contacts/emails** gathered so far in the workspace — useful after running email-harvesting modules like `recon/domains-contacts/whois_pocs`.

```bash
info
```
Shows detailed **information about the currently loaded module** — description, required options, and source/author — essential for understanding what a module actually needs before running it.

---

## 6. Tool: Amass

**Amass** (OWASP Amass) is one of the most powerful **subdomain enumeration and attack-surface mapping** tools available — it combines passive OSINT sources, active DNS resolution, and brute-forcing into one tool, and is widely considered an industry standard for asset discovery at scale.

### 6.1 How It Works
Amass works in layers, and you control how many of them run:
- **Passive layer** — queries dozens of external OSINT sources and APIs (certificate transparency logs, public DNS datasets, search engines) simultaneously, without ever touching the target's own DNS servers.
- **Active layer** — once candidate subdomains are gathered, Amass resolves them directly against DNS to confirm they're real and live, and can attempt techniques like zone transfers.
- **Brute-force layer** — tries large wordlists of common subdomain names against the target's DNS to catch names no passive source ever indexed.
- Internally, Amass builds a **graph database** of everything it finds (domains, subdomains, IPs, ASNs, netblocks) so it can show not just a flat list, but how assets relate to each other.

### 6.2 What It's Used For
- The **deepest and most complete subdomain enumeration** available in this tool set — used when a quick pass (like theHarvester) isn't thorough enough.
- Mapping an organization's full external attack surface, including shared hosting and infrastructure relationships.
- Feeding a clean, deduplicated subdomain list into downstream tools (Nmap, httpx, Nuclei) later in an assessment.

### 6.3 Important Commands

```bash
amass enum -d example.com
```
Runs the **default enumeration** — combines passive sources with active DNS resolution to find subdomains; this is the command you'll use most.

```bash
amass enum -passive -d example.com
```
Runs in **passive-only mode** — no direct queries hit the target's DNS infrastructure at all, purely OSINT sources (fastest, stealthiest, but slightly less complete).

```bash
amass enum -active -d example.com
```
Enables **active techniques** — includes things like zone transfer attempts and DNS resolution validation, in addition to passive sources (more thorough, more noise).

```bash
amass enum -brute -d example.com
```
Adds **brute-force subdomain guessing** using Amass's built-in wordlists — finds subdomains that don't appear in any passive source.

```bash
amass enum -brute -w wordlist.txt -d example.com
```
Same as above but with a **custom wordlist** (`-w`) — lets you use larger or more specialized wordlists (e.g., industry-specific naming patterns).

```bash
amass enum -d example.com -o results.txt
```
Saves discovered subdomains to an **output file** (`-o`) — needed for feeding results into other tools (Nmap, httpx, etc.) later in the pipeline.

```bash
amass enum -d example.com -json results.json
```
Exports results in **JSON format** — structured output useful for automation, scripting, or importing into other platforms.

```bash
amass intel -d example.com
```
Runs **Amass Intel mode** — a separate mode focused on discovering *related* domains/organizational infrastructure (not just subdomains of one domain), useful for early-stage recon when you don't even have a full domain list yet.

```bash
amass intel -org "Example Corp"
```
Searches Intel mode by **organization name** instead of a domain — useful when you know the company name but need to discover which domains they actually own.

```bash
amass viz -d3 -d example.com
```
Generates a **visualization** (`viz`) of discovered assets and their relationships — here as an interactive D3.js graph — turning raw enumeration data into a visual asset map.

---

## 7. Tool: Maltego CE

**Maltego Community Edition** is a graphical **link-analysis and OSINT platform** — instead of terminal output, it represents entities (domains, IPs, emails, people, organizations) as **nodes** on a graph, connected by relationships discovered through "Transforms" (small scripts/API calls that expand a node into related data). It's the tool most associated with visual asset mapping.

### 7.1 Top 10 Concepts/Commands (GUI-Driven Tool)

Maltego is primarily a **graphical application**, so its "commands" are core actions/workflows rather than terminal syntax:

```
1. Create a new Graph
```
Starts a blank canvas — the workspace where your investigation graph will be built entity by entity.

```
2. Drag an Entity onto the graph (e.g., Domain)
```
Adds a starting point — e.g., type `example.com` into a **Domain** entity — every investigation starts from one or more seed entities.

```
3. Run Transform → "To DNS Name" / "To Subdomains"
```
Right-clicking an entity and running a **Transform** expands it — e.g., a Domain entity expanding into all known subdomains as new connected nodes.

```
4. Run Transform → "To Email Address"
```
Expands a Domain or Person entity into associated **email addresses** found via connected OSINT sources — core to the email enumeration workflow.

```
5. Run Transform → "To IP Address [DNS]"
```
Resolves a Domain/DNS Name entity to its **IP address**, adding infrastructure nodes to the graph.

```
6. Run Transform → "To Website [Quick Lookup]"
```
Expands a domain into associated **website/URL entities**, useful for mapping the actual web presence tied to a domain.

```
7. Run Transform → "To Phrases/Person" (via Social sources)
```
Expands organizational entities into associated **people**, useful for building an org chart from public profiles.

```
8. Use "Machines" (automated Transform chains)
```
Runs a **pre-built sequence of Transforms automatically** (e.g., the "Footprint L1/L2/L3" machines) — instead of manually running each Transform, a Machine runs an entire recon workflow in one click.

```
9. Apply Layout (e.g., "Organic Layout" / "Block Layout")
```
Rearranges the graph visually so clusters and relationships are easier to read — critical once a graph grows to hundreds of nodes.

```
10. Export Graph (as image / MTGX file)
```
Saves the finished graph as an **image** (for reports) or a native `.mtgx` file (to reopen and continue the investigation later).

---

## 8. Quick Cheat Sheet

| Task | Command |
|---|---|
| Passive email/subdomain sweep | `theHarvester -d example.com -b all` |
| Certificate-log based subdomain find | `theHarvester -d example.com -b crtsh` |
| Modular recon framework session | `recon-ng` → `workspaces create` → `modules load ...` → `run` |
| Full subdomain enumeration | `amass enum -d example.com` |
| Passive-only, stealthy subdomain enum | `amass enum -passive -d example.com` |
| Brute-force subdomains | `amass enum -brute -w wordlist.txt -d example.com` |
| Visualize discovered assets | `amass viz -d3 -d example.com` or build a graph in **Maltego** |
| Org-wide asset discovery by company name | `amass intel -org "Example Corp"` |

**Typical Advanced Recon Workflow (lab/authorized environment):**
```bash
theHarvester -d example.com -b all -f harvester_report.html   # 1. Passive email + subdomain sweep
amass enum -passive -d example.com -o amass_passive.txt          # 2. Passive subdomain enumeration
amass enum -active -brute -d example.com -o amass_full.txt          # 3. Active + brute-force enumeration
recon-ng                                                                # 4. Deep-dive specific modules (contacts, breaches, etc.)
# 5. Feed all findings into Maltego to build a visual asset map & final report graph
```

---

### Practice Suggestions for Day 5
1. Run `theHarvester` against a domain using 2–3 different sources (`-b google`, `-b bing`, `-b crtsh`) and compare how much overlap vs. unique results each source finds.
2. Set up a Recon-ng workspace, install 2 modules (one subdomain, one contact-related), and run them against a lab/practice domain.
3. Run `amass enum -passive` and `amass enum -active -brute` on the same domain and compare speed vs. completeness.
4. Export Amass results as JSON and try parsing them with `jq` (from Day 2) to extract just the subdomain names.
5. Open Maltego CE, start from a single Domain entity, and use the "Footprint L1" machine to auto-build a small asset graph.
6. Combine results from all four tools into a single written asset-mapping report — domains, subdomains, IPs, and emails found — as practice for the Day 6 lab.
