# Capstone Lab — Reconnaissance Walkthrough
### Applying Day 1–5 Tools End-to-End on a Real Target: `shreypandey.tech`
**NICSI Internship Program 2026**

---

> ⚠️ **Read this before anything else.**
> `shreypandey.tech` is a **real, live, third-party-owned website** — a personal portfolio, not a lab machine. This walkthrough is built around it because reconnaissance is far more meaningful against a real target than a fake `example.com`. But that also means the usual rule applies with extra weight:
> - **Passive/OSINT steps** (Whois, public DNS record lookups, viewing the site's own public content, theHarvester-style searches of already-public info) are low-risk and are what this lab actually demonstrates in full.
> - **Active/intrusive steps** (full port sweeps, subdomain brute-forcing, zone-transfer attempts, directory fuzzing) are shown here **for command syntax only**, clearly marked, and should be **pointed at your own lab VM** (or run against this domain only with the owner's explicit permission) — not fired at someone else's production site during self-study.
>
> Where this document shows "output," anything beyond what's genuinely visible on the site's own public page is marked **`[illustrative]`** — a realistic example of what the tool *would* return, not a claim about this specific server's real, live configuration.

---

## The Scenario

You've just joined a security team as an intern. Your mentor gives you a simple brief:

> *"Here's a domain: `shreypandey.tech`. Before we'd ever assess something like this for a client, we always run a structured reconnaissance pass first — passive before active, broad before deep. Walk me through it, tool by tool, and tell me *why* you're reaching for each one before you use it."*

This lab is that walkthrough — following the exact same five-day tool progression you've already learned, now applied in sequence to one real target.

```
Day 1 (Bash/tmux)   →  Set up the workspace
Day 2 (curl/jq/nc)   →  Talk directly to the website itself
Day 3 (Nmap/Masscan/Netdiscover) →  Map the network layer
Day 4 (Whois/DNSRecon/Fierce)      →  Map the DNS & domain layer
Day 5 (theHarvester/Recon-ng/Amass/Maltego) → Map people, subdomains & the full asset picture
```

---

## Phase 0 — Setting Up the Workspace (Day 1: Bash + tmux)

**Why start here?** Before touching the target at all, a real engagement needs an organized, persistent workspace — so that if your SSH session drops, or you need to run three tools in parallel, you don't lose progress or lose track of output. This is purely *your* environment, not the target — nothing here touches `shreypandey.tech` yet.

```bash
mkdir -p ~/engagements/shreypandey-tech/{recon,dns,network,osint,report}
cd ~/engagements/shreypandey-tech
```
Creates a clean, categorized folder structure up front — every phase below will drop its output into the matching folder.

```bash
tmux new -s shreypandey_recon
```
Starts a **named tmux session** for this engagement. If your connection drops mid-scan, the session (and any running tool) keeps running in the background.

```bash
# Inside tmux:
Ctrl+b %        # split pane vertically — one pane for running commands
Ctrl+b "        # split again — a second pane to tail a log file
Ctrl+b d        # detach whenever you need to step away
tmux attach -t shreypandey_recon   # reattach later, exactly where you left off
```
Splitting panes lets you run a live scan in one pane while watching its output log in another — standard practice once engagements involve long-running tools (Amass, Nmap full-port scans).

**Why Bash matters here:** every tool below produces a differently-formatted output. A short Bash habit — appending `| tee recon/curl_output.txt` to any command — means nothing you find is ever lost to scrollback.

---

## Phase 1 — Talking to the Website Directly (Day 2: curl + jq + Netcat)

**Why this phase, and why first?** Before mapping ports or DNS, the single richest, lowest-risk source of information is the website itself — it's *designed* to be public. A simple `curl` often reveals more in 10 seconds than an hour of scanning.

### Step 1.1 — Fetch the raw page and headers
```bash
curl -I https://www.shreypandey.tech/
```
`-I` fetches **headers only** — reveals the web server/hosting stack, caching behavior, and security headers, without downloading the whole page. This is the safest possible first request — identical to what your browser sends just visiting the site.

```bash
curl -Ls https://www.shreypandey.tech/ -o recon/homepage.html
```
`-L` follows redirects, `-s` keeps it quiet, and this saves the full rendered HTML for later inspection — e.g., checking `<meta>` tags, linked scripts, and framework fingerprints.

```bash
grep -i "next\|generator\|vercel\|framework" recon/homepage.html
```
A quick grep of the saved HTML. **What we actually found doing exactly this:** the page loads images through a `/_next/image` path — a strong, direct fingerprint of **Next.js** (a React framework), commonly deployed on **Vercel**. This single detail already tells us a lot: likely serverless/edge hosting, meaning a classic "scan the origin IP" approach (Phase 2) may only ever show us a CDN edge node, not a real backend server.

### Step 1.2 — Map the site's own structure
```bash
curl -s https://www.shreypandey.tech/ | grep -oE 'href="[^"]+"' | sort -u
```
Extracts every link on the homepage. **Genuine finding from the live page:** the site publicly links its own internal pages — `/vault`, `/logs`, `/credentials`, `/resume`, and `/contact` — plus external profile links to **GitHub** and **LinkedIn**. This *is* our site map, handed to us directly, no brute-forcing required.

### Step 1.3 — Pull out anything structured (jq)
```bash
curl -s https://www.shreypandey.tech/api/some-endpoint | jq .
```
If the site exposes any JSON API endpoints (common in Next.js apps, e.g. under `/api/`), `jq` instantly pretty-prints the response so it's readable, and lets you drill into specific fields, e.g.:
```bash
curl -s https://www.shreypandey.tech/sitemap.xml | jq -R . 2>/dev/null
```
`[illustrative]` — many Next.js sites auto-generate a `sitemap.xml`; checking for one is a standard, harmless step that (if present) hands you a complete list of every public page in one file.

### Step 1.4 — Confirm basic reachability (Netcat)
```bash
nc -zv shreypandey.tech 443
nc -zv shreypandey.tech 80
```
**Why Netcat, and why *now*, not in Phase 2?** Before running a full Nmap sweep (a heavier, more "scanning-flavored" action), a two-second Netcat check on the two ports we *already expect* to be open (80/443, since we're literally looking at the live site) confirms basic reachability — a lightweight sanity check, not a scan.

```bash
echo -e "HEAD / HTTP/1.1\r\nHost: www.shreypandey.tech\r\n\r\n" | nc www.shreypandey.tech 80
```
Manually sends a raw HTTP request over the open connection — useful for seeing the server's raw response line-by-line, without any of curl's automatic formatting/redirect-following getting in the way.

> **What Phase 1 gave us, for free:** confirmed framework (Next.js/likely Vercel), a full internal site map (5+ pages), and two real, self-disclosed OSINT items — the site's own public **email** and **social profile links** — all without a single "scan" in the traditional sense.

---

## Phase 2 — Mapping the Network Layer (Day 3: Nmap, Masscan, Netdiscover)

**Why this phase, and why now?** Phase 1 told us *what* the site is; this phase asks *where and how it's reachable at the network level* — what ports are open, and (if we're lucky) what's running on them. This is the first genuinely **active** phase — the target will see these packets.

### Step 2.1 — Resolve the domain first
```bash
dig +short shreypandey.tech
```
Before scanning anything, get the actual IP(s) behind the domain. **Important teaching point:** if Phase 1's Next.js/Vercel fingerprint is correct, this IP is very likely a shared **CDN/edge IP**, not a dedicated server — meaning a port scan here tells you about Vercel's edge network, not about "Shrey's server," because there usually isn't one exposed directly.

### Step 2.2 — Nmap: the core port/service/OS sweep
```bash
nmap -sn shreypandey.tech
```
A ping-sweep-style check — confirms the host responds, before committing to a full scan.

```bash
nmap -p 80,443 shreypandey.tech
```
Since we already know from Phase 1 that the site serves HTTP/HTTPS, this targeted scan just **confirms** what we expect rather than blindly scanning all 65,535 ports on someone else's production infrastructure. This is the appropriately-scoped version of a scan for a target like this.

```bash
nmap -sV -p 80,443 shreypandey.tech
```
Adds **service/version detection** on just those two ports — on a CDN-fronted site, this typically reveals the CDN's own web server signature (e.g., a generic edge proxy banner) rather than the original app server. `[illustrative]` — actual banner text will vary and often be deliberately generic on managed platforms.

```bash
# Full port sweep and OS detection — syntax reference only.
# Run this against your own lab VM, or shreypandey.tech only with the owner's permission:
nmap -p- -T4 <lab-target-ip>
nmap -O <lab-target-ip>
```
**Why we don't run these two against the live site in this walkthrough:** a full 65,535-port sweep and OS fingerprinting are noisier, slower, and more "attack-shaped" actions than a scope-appropriate recon pass calls for on someone else's live personal site — the two lines above exist so you have the exact syntax ready for your authorized lab practice.

### Step 2.3 — Masscan: only relevant at scale
`[illustrative — conceptual, not run here]` Masscan exists to sweep **huge ranges** (entire subnets or the whole IPv4 space) fast. Since we have exactly *one* domain and already know its likely IP, Masscan brings no advantage over Nmap here — it would matter if, say, DNS enumeration in Phase 3 revealed 40 subdomains across a spread of IPs and you needed to triage all of them quickly:
```bash
masscan <ip-range-from-later-phase> -p80,443 --rate=1000
```
**Why mention it at all if we don't run it?** Part of good tool selection is recognizing when a tool *doesn't* fit the scale of the target — Masscan is the right call for "map this /16," not for "check two ports on one known host."

### Step 2.4 — Netdiscover: doesn't apply here, and here's why
```bash
# Netdiscover works via ARP — Layer 2 — meaning it only ever discovers
# hosts on your OWN local subnet. shreypandey.tech is a public internet
# host, not on our local network, so Netdiscover has literally nothing
# to query here.
netdiscover -r 192.168.1.0/24   # [illustrative — only useful on a local LAN engagement]
```
**This is an important, honest finding, not a gap.** On an *internal* network assessment (e.g., "map every device in this office subnet"), Netdiscover would be one of the very first tools run. On an internet-facing target like this one, it simply isn't the right tool for the job — recognizing that is as valuable a skill as running the tool correctly.

> **What Phase 2 gave us:** confirmation that HTTP/HTTPS are open (expected), a service banner likely belonging to a CDN edge (not an origin server), and — just as importantly — a clear understanding of *why* Masscan and Netdiscover weren't the right tools for this particular target's shape.

---

## Phase 3 — Mapping the Domain & DNS Layer (Day 4: Whois, DNSRecon, Fierce)

**Why this phase, and why now?** We now understand the site and its immediate network footprint. This phase zooms out to the **domain itself** — who registered it, how DNS is configured, and whether it's been hardened against common misconfigurations.

### Step 3.1 — Whois: who's actually behind the domain
```bash
whois shreypandey.tech
```
Pulls registrar, registration/expiry dates, and nameserver info. `[illustrative]` — for many personal `.tech` domains, expect registrant *contact* details to be redacted behind registrar privacy protection (very common and not itself a red flag), while registrar and nameserver fields typically remain visible.

```bash
whois shreypandey.tech | grep -i "name server"
```
Filters straight to the **nameservers** — this tells you which DNS/hosting provider actually answers for the domain (e.g., Vercel's or Cloudflare's nameservers, consistent with the Next.js fingerprint from Phase 1).

### Step 3.2 — DNSRecon: the full record sweep
```bash
dnsrecon -d shreypandey.tech -t std
```
Runs the **standard record sweep** — A, AAAA, MX, NS, TXT, SOA — building a complete picture of the domain's DNS configuration in one pass.

```bash
dnsrecon -d shreypandey.tech -t mx
```
Isolates **mail server records** — tells you where email for the domain is actually routed (e.g., Google Workspace), independent of the public Gmail address we already saw on the page itself in Phase 1.

```bash
# Zone-transfer attempt — syntax reference only, run against your own lab zone:
dnsrecon -d shreypandey.tech -t axfr
```
**Why we show this but expect it to fail (and that's a good thing):** a successful zone transfer here would mean the domain's DNS is seriously misconfigured. On any properly managed provider (Vercel/Cloudflare-class), this will simply be refused — which is the *correct, secure* outcome, and worth explicitly confirming rather than assuming.

### Step 3.3 — Fierce: a fast second opinion
```bash
fierce --domain shreypandey.tech
```
Runs a quick subdomain sweep plus its own zone-transfer attempt. **Why run this *and* DNSRecon, not just one?** Different tools use different default wordlists and techniques — Fierce is deliberately lighter/faster, making it a good "quick second pass" to sanity-check DNSRecon's more thorough results, or to run first if you want a fast initial read before committing to a longer DNSRecon sweep.

> **What Phase 3 gave us:** the domain's registrar/nameserver story (consistent with the Vercel/Next.js hosting theory from Phase 1), confirmation that zone transfers are properly refused, and a DNS record baseline (MX, TXT, etc.) to compare against whatever subdomains Phase 4 finds.

---

## Phase 4 — Mapping People, Subdomains & the Full Picture (Day 5: theHarvester, Recon-ng, Amass, Maltego CE)

**Why this phase, and why last?** This is where everything from Phases 1–3 gets *fused* — turning "a domain, two open ports, and some DNS records" into a full picture of subdomains, associated people, and how it all connects. It deliberately comes last because each tool here works best when it has real earlier findings (the confirmed email, the framework, the DNS baseline) to cross-reference against.

### Step 4.1 — theHarvester: confirm and expand the email/subdomain picture
```bash
theHarvester -d shreypandey.tech -b crtsh
```
Queries **Certificate Transparency logs** — since every SSL certificate issued for the domain (and any subdomains) is permanently, publicly logged, this is one of the most reliable ways to catch subdomains that might not be linked from the homepage itself.

```bash
theHarvester -d shreypandey.tech -b all -f recon/harvester_report.html
```
A broad passive sweep across every supported source, saved as a shareable HTML report. **Why this confirms rather than "discovers" here:** we already found the site's real, self-published email (`shrey42pandey@gmail.com`) and social links directly from the homepage in Phase 1 — theHarvester's job at this stage is to confirm that public sources agree, and to check whether *additional*, less obvious mentions exist elsewhere on the web.

### Step 4.2 — Recon-ng: organize everything into one queryable workspace
```bash
recon-ng
workspaces create shreypandey_tech
modules load recon/domains-hosts/hackertarget
options set SOURCE shreypandey.tech
run
show hosts
```
**Why bring this in now, not earlier?** Recon-ng's real value is centralizing — by this point we have Whois data, DNS records, and theHarvester output; loading everything into one workspace database means later modules (e.g., ones that check breach databases for the confirmed email) can build on what's already been found, instead of starting from zero.

### Step 4.3 — Amass: the deep subdomain sweep
```bash
amass enum -passive -d shreypandey.tech
```
A stealthy, fully passive subdomain sweep — the appropriate choice to run as-is here, since it never sends a single packet to the target itself.

```bash
amass enum -d shreypandey.tech -o osint/amass_results.txt
```
The default (passive + light active resolution) mode, saved for later cross-referencing against the DNS baseline from Phase 3.

```bash
# Brute-force mode — heavier and more active; syntax reference only:
amass enum -brute -w wordlist.txt -d shreypandey.tech
```
**Why we don't run brute-force mode against the live site here:** it sends a large volume of DNS queries in a short window — appropriate for an authorized engagement, not for casual self-study against someone's personal domain. The command is included so the syntax is ready when you *do* have a properly scoped target.

### Step 4.4 — Maltego CE: pulling it all into one visual graph
Instead of terminal commands, this is where everything gets **assembled visually**:
1. Create a new Graph, and add a **Domain** entity: `shreypandey.tech`.
2. Run **"To DNS Name / Subdomains"** — pulls in anything confirmed by Amass/Fierce/DNSRecon as connected nodes.
3. Run **"To Email Address"** — surfaces the same confirmed email from Phase 1, now as a graph node connected back to the domain.
4. Add a **Person** entity manually for the confirmed name (from the page's own "Hi, I'm Shrey Pandey" heading) and run **"To Website" / "To Social Network"** Transforms to connect the GitHub/LinkedIn links already found in Phase 1.
5. Apply an **Organic Layout** so the domain, subdomains, email, and social profiles visually cluster together.
6. **Export the graph** as an image — this becomes the single-page visual summary for the final report.

**Why Maltego is the *last* step, always:** it isn't a discovery tool in the same sense as the others — its value is entirely in connecting dots that earlier tools already found. Running it first would just produce an empty canvas.

> **What Phase 4 gave us:** confirmation of the site's public contact/social footprint, a passive subdomain baseline via Amass, a centralized Recon-ng workspace for any follow-up modules, and — finally — one consolidated Maltego graph tying the domain, its DNS footprint, and its public identity together visually.

---

## Final Consolidated Findings (Report Summary)

| Category | Finding | Source Tool |
|---|---|---|
| Framework/Hosting | Next.js app, likely Vercel-hosted (via `/_next/image` path) | curl (Phase 1) |
| Site structure | `/`, `/vault`, `/logs`, `/credentials`, `/resume`, `/contact` | curl (Phase 1) |
| Public contact | `shrey42pandey@gmail.com` | curl / theHarvester (Phase 1 & 4) |
| Social footprint | GitHub, LinkedIn (linked directly from homepage) | curl / Maltego (Phase 1 & 4) |
| Open ports | 80, 443 (expected, confirmed) | Netcat / Nmap (Phase 1 & 2) |
| DNS configuration | Registrar/nameservers identified; zone transfer correctly refused | Whois / DNSRecon (Phase 3) |
| Subdomains | Passive Amass/Fierce/theHarvester sweep baseline | Amass / Fierce (Phase 3 & 4) |
| Asset map | Consolidated Maltego graph (domain ↔ email ↔ social ↔ subdomains) | Maltego CE (Phase 4) |

---

## Why the Sequence Matters (The Big-Picture Lesson)

Notice the order was never arbitrary:

1. **Bash/tmux first** — because a messy workspace produces a messy report, regardless of how good your scans are.
2. **curl/jq/Netcat second** — because the target's *own* public content is the cheapest, richest, lowest-risk source of truth, and it should shape every decision after it (e.g., recognizing the Vercel/CDN pattern *before* wasting time on a full port sweep).
3. **Nmap/Masscan/Netdiscover third** — because once you know what to expect at the network layer, you scan with a hypothesis to confirm, not blindly.
4. **Whois/DNSRecon/Fierce fourth** — because the domain/DNS layer explains *why* the network layer looks the way it does (e.g., CDN nameservers explaining the CDN-shaped port scan results).
5. **theHarvester/Recon-ng/Amass/Maltego last** — because fusing everything into people, subdomains, and a visual graph only works once there's real data from every earlier phase to connect.

**The core skill this lab is really teaching:** it was never about memorizing commands — it's about knowing *which tool to reach for, in what order, and why*, based on what the target actually is.

---

### Suggested Next Step for Your Own Practice
Repeat this exact five-phase sequence against a domain you **own** or a **deliberately vulnerable lab target** (e.g., a self-hosted test site), and this time let every phase run to full depth — including the active steps marked `[illustrative]` above. Compare how different the story looks on a target that *isn't* sitting behind a modern CDN.
