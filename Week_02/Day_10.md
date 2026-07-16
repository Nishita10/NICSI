# Day 10 — Web Security (Interception & Manual Testing)


**Tools**
● Burp Suite Community ● OWASP ZAP

**Topics**
● Proxy ● Repeater ● Intruder (Community limitations) ● Request/Response Analysis

A complete reference covering proxying, request replay, automated request manipulation, and request/response analysis — using Burp Suite Community and OWASP ZAP. (Topics are explained first, followed by the tools — each explained by how it works and what it's used for, then its most important commands/workflows.)



---

## Table of Contents
1. [Proxy](#1-proxy)
2. [Repeater](#2-repeater)
3. [Intruder (and Community Edition Limitations)](#3-intruder-and-community-edition-limitations)
4. [Request/Response Analysis](#4-requestresponse-analysis)
5. [Tool: Burp Suite Community](#5-tool-burp-suite-community)
6. [Tool: OWASP ZAP](#6-tool-owasp-zap)
7. [Quick Cheat Sheet](#7-quick-cheat-sheet)

---

## 1. Proxy

A **proxy**, in this context, is a piece of software that sits **between your browser and the target web application**, intercepting every request and response as it passes through — letting you see, pause, and modify traffic that would normally be invisible to you.

### 1.1 Why This Changes Everything About Web Testing
Every tool from Days 8–9 (Nikto, Nuclei, Wapiti) worked by sending requests *for* you, automatically. A proxy flips that model: **you** browse the application normally, and the proxy quietly captures the exact, real HTTP requests your browser sends and the exact responses it gets back — including things a normal browser hides from you entirely, like:
- Hidden form fields and parameters.
- Full request headers, cookies, and authentication tokens.
- The raw, unrendered API responses behind a page (not just what's visually displayed).

### 1.2 How It Works, Conceptually
```
Your Browser  →  Proxy (Burp/ZAP)  →  Target Web Application
Your Browser  ←  Proxy (Burp/ZAP)  ←  Target Web Application
```
1. Your browser is configured to send all traffic through the proxy (usually `127.0.0.1:8080`).
2. The proxy tool installs a **local certificate** so it can also intercept and decrypt **HTTPS** traffic (otherwise encrypted traffic would be unreadable to it).
3. Every request/response can be viewed in real time, held for inspection, edited before forwarding, or simply logged for later review.

### 1.3 Why It Matters
- It's the **foundation** every other feature in this document is built on — Repeater and Intruder both start from a request the Proxy captured.
- Reveals the application's *true* attack surface — parameters and endpoints a simple visual browse-through would never show you.
- Essential for testing anything that isn't a simple public GET request — logged-in workflows, multi-step forms, API-driven single-page apps.

---

## 2. Repeater

**Repeater** is the feature that lets you take a single captured request, **manually edit it**, and **resend it as many times as you want** — observing exactly how the server's response changes with each variation.

### 2.1 How It Works
1. You send a request to Repeater (usually via right-click → "Send to Repeater" from the Proxy history).
2. Repeater shows the request in an editable pane — you can change parameters, headers, cookies, the HTTP method, the body, anything.
3. Clicking **Send** fires that exact modified request and displays the raw response alongside it.
4. You repeat this — tweak, send, observe — as many times as needed, with every attempt kept in history for comparison.

### 2.2 Why It Matters
- This is where **manual vulnerability testing** actually happens — e.g., manually testing a single parameter for SQL injection by trying `' OR 1=1--` and directly reading how the response changes, rather than trusting an automated tool's verdict.
- Lets you **isolate variables** — change exactly one thing at a time and see its precise effect, which automated scanners can't always explain as clearly.
- Essential for **verifying** a finding from Nuclei/Wapiti (Day 9) — confirming a suspected vulnerability is real by manually crafting the exact proof-of-concept request.

---

## 3. Intruder (and Community Edition Limitations)

**Intruder** is Burp Suite's feature for **automating repeated requests with varying payloads** — instead of manually changing one value and clicking Send a hundred times in Repeater, Intruder does that for you: substituting a list of values into a chosen position and firing the request once per value.

### 3.1 How It Works (Conceptually)
1. Take a base request (again, usually sent from the Proxy) and mark specific **positions** within it (e.g., a parameter value, a header value).
2. Choose an **attack type** — how the payload list(s) get applied to the marked position(s):

| Attack Type | Behavior |
|---|---|
| **Sniper** | One payload set, cycled through one position at a time |
| **Battering ram** | One payload set, same payload inserted into *all* positions simultaneously |
| **Pitchfork** | Multiple payload sets, moving through them in parallel (position 1 uses list A, position 2 uses list B, in lockstep) |
| **Cluster bomb** | Multiple payload sets, tried in every possible combination |

3. Supply a **payload list** — e.g., a wordlist of usernames, a list of SQLi test strings, a numeric range.
4. Run the attack — Intruder sends one request per payload and records every response (status code, length, timing) for comparison, so you can quickly spot the *one* response that looks different (e.g., a login that returns "200 OK" instead of the usual "401").

### 3.2 Community Edition Limitations — Important to Understand
This is a deliberate product limitation, not a bug, and it shapes how you actually use Burp in practice:
- **Intruder is heavily rate-limited** in Burp Suite **Community Edition** — the Professional edition runs Intruder attacks at full speed, while Community throttles it significantly, making large payload lists (thousands of entries) impractically slow.
- There is **no built-in resolution** to this in Community — it's one of the main paid-feature distinctions of Burp Professional.
- **Practical workaround:** for small, targeted payload sets (a handful to a few dozen values) Community's Intruder is still usable; for large-scale brute-forcing or fuzzing, security professionals typically reach for a **dedicated tool instead** — e.g., `ffuf` or ability within OWASP ZAP's own automation, both of which run without this artificial throttle.

### 3.3 Why It Matters (Despite the Limitation)
- Even throttled, Intruder is invaluable for **small-scale, precise testing** — e.g., trying 10 different SQLi payloads against one parameter, or testing a short list of default credentials.
- Understanding *why* Community is slower here is a genuinely useful, practical lesson: it teaches you to **pick the right tool for the job's scale**, exactly like the Netdiscover/Masscan lesson from Day 3.

---

## 4. Request/Response Analysis

**Request/Response Analysis** is the practice of carefully reading the raw HTTP traffic captured by the Proxy — not just whether a page "looks right" in the browser, but examining the actual headers, status codes, cookies, and body content for security-relevant details.

### 4.1 What You're Looking For

| Element | What to Check |
|---|---|
| **Status codes** | Does a "failed" login really return 401/403, or does it silently return 200 with an error message in the body (a common logic flaw)? |
| **Response headers** | Are security headers present — `Content-Security-Policy`, `X-Frame-Options`, `Strict-Transport-Security`? Is the `Server` header leaking version info? |
| **Cookies** | Are session cookies flagged `HttpOnly` and `Secure`? Is the session token predictable? |
| **Request parameters** | Are there hidden/extra parameters (`isAdmin=false`, `debug=true`) that a normal user flow never shows but still get processed? |
| **Response body/timing** | Does the response body or response time differ subtly based on input — a classic sign of a blind vulnerability (e.g., a slower response for a valid vs. invalid username, revealing user enumeration)? |
| **Error messages** | Do errors leak stack traces, file paths, database types, or internal IPs? |

### 4.2 Why It Matters
- This is the skill that turns raw proxy captures into actual **findings** — the tool shows you the traffic, but a human still has to recognize what's *wrong* with it.
- Many real vulnerabilities (broken access control, business logic flaws, information disclosure) are **invisible to automated scanners** and only surface through careful manual analysis of requests and responses.
- Forms the evidentiary basis of a professional report — screenshots of a raw request/response pair are the standard way to prove a finding is real.

---

## 5. Tool: Burp Suite Community

**Burp Suite** is the industry-standard **web application testing platform**, built entirely around the proxy-centered workflow described above. The **Community Edition** is free and includes Proxy, Repeater, a rate-limited Intruder, and manual analysis tools — but omits the automated active scanner and a few other features reserved for the paid Professional tier.

### 5.1 How It Works
Burp runs as a desktop application containing multiple integrated tools sharing one traffic history:
- **Proxy listener** — runs locally (default `127.0.0.1:8080`); your browser (often configured via **FoxyProxy** or similar) routes traffic through it.
- **HTTP history** — every captured request/response is logged here, searchable and filterable.
- **Right-click actions** — send anything from the Proxy history to Repeater, Intruder, or Comparer with one click, keeping your whole workflow inside one tool.
- **Burp's CA certificate** must be installed in your browser once, to allow HTTPS interception.

### 5.2 What It's Used For
- The **primary manual web application testing tool** used industry-wide for penetration testing and bug bounty work.
- Intercepting and analyzing traffic from complex, authenticated, multi-step web apps that automated scanners handle poorly.
- Manually crafting and verifying proof-of-concept exploit requests (via Repeater) before reporting a finding.

### 5.3 Important Workflow & Commands

```
1. Proxy → Options → confirm listener is running on 127.0.0.1:8080
```
Verifies Burp's proxy is active before configuring your browser — the starting point of every session.

```
2. Install Burp's CA Certificate in your browser (visit http://burp on the proxied browser)
```
Required once per browser profile — without this, all HTTPS traffic will show certificate errors instead of being readable.

```
3. Proxy → Intercept → toggle "Intercept is on/off"
```
When **on**, every request pauses in Burp before being forwarded — letting you inspect/edit it live. Most of the time you'll leave this **off** and just review the passive **HTTP history** log instead.

```
4. Proxy → HTTP History → right-click a request → "Send to Repeater"
```
The core habit of manual testing — pick any captured request and move it to Repeater for detailed, repeated experimentation.

```
5. Repeater tab → edit the request → click "Send"
```
Fires your modified request and shows the raw response side-by-side — repeat as needed while testing a hypothesis.

```
6. Proxy → HTTP History → right-click a request → "Send to Intruder"
```
Moves a request into Intruder for automated payload substitution (see Section 3 for attack types and Community's rate limit).

```
7. Intruder tab → Positions sub-tab → highlight a value → click "Add §"
```
Marks the exact spot in the request where payloads will be inserted — Burp wraps the position in `§` markers.

```
8. Intruder tab → Payloads sub-tab → load or type a payload list → click "Start attack"
```
Defines what values to try in the marked position(s), then launches the (rate-limited, in Community) attack.

```
9. Proxy → HTTP History → right-click two requests → "Send to Comparer"
```
Highlights the **exact differences** between two requests/responses — useful for spotting subtle changes (e.g., comparing a valid vs. invalid login response).

```
10. Target tab → Site map
```
Builds a structured tree of every URL/endpoint Burp has observed during your session — a passively-built map of the application's structure as you browse.

---

## 6. Tool: OWASP ZAP

**OWASP ZAP (Zed Attack Proxy)** is a free, fully open-source web application security scanner and proxy — functionally similar to Burp in its Proxy/Repeater-equivalent workflow, but **without Burp's Community-edition Intruder throttling**, and with automated active/passive scanning built in for free.

### 6.1 How It Works
- Like Burp, ZAP runs a local **proxy listener** (default `127.0.0.1:8080`) that your browser routes through, capturing and displaying every request/response.
- ZAP additionally runs **Passive Scanning automatically** in the background the moment traffic flows through it — flagging basic issues (missing headers, insecure cookies) without you doing anything extra.
- Its **Active Scan** feature (free, unlike Burp Pro's scanner) actively crawls and attacks discovered endpoints automatically, similar in spirit to Wapiti (Day 9) but integrated directly into the proxy workflow.
- ZAP also has a scriptable **Fuzzer**, which fills the role Intruder plays in Burp — without Community-edition-style rate limiting.

### 6.2 What It's Used For
- A **fully free alternative** to Burp Suite Professional's automated scanning capability — a common reason teams choose ZAP when budget is a constraint.
- Automated baseline scans as part of a CI/CD pipeline (ZAP has strong automation/scripting support for this).
- The same manual proxy-based workflow as Burp — intercepting, replaying, and analyzing requests by hand — for teams or students who prefer a fully open-source toolchain.

### 6.3 Important Workflow & Commands

```
1. Tools → Options → Local Proxy → confirm address/port (default 127.0.0.1:8080)
```
Confirms ZAP's proxy listener settings before pointing your browser at it — same first step as Burp.

```
2. Install the ZAP Root CA Certificate (Tools → Options → Dynamic SSL Certificates → Save)
```
Required once per browser, to allow ZAP to intercept and decrypt HTTPS traffic.

```
3. Automated Scan tab → enter target URL → click "Attack"
```
ZAP's quickest option — a one-click combination of spidering the site and running an active scan against everything it finds, ideal for a fast initial pass.

```
4. Sites tab → browse the tree → right-click a request → "Open/Resend with Request Editor"
```
ZAP's equivalent of Burp's Repeater — edit a captured request manually and resend it, observing the response.

```
5. Sites tab → right-click a target → Attack → "Spider..."
```
Crawls the site to discover pages/endpoints, building out the site map — run this before an Active Scan for full coverage.

```
6. Sites tab → right-click a target → Attack → "Active Scan..."
```
Launches ZAP's built-in automated vulnerability scanner against the target — free in ZAP, unlike Burp Community.

```
7. Alerts tab
```
Lists every finding ZAP's Passive and Active scanning has flagged so far, each with a description and risk rating — ZAP's equivalent of a live, running findings report.

```
8. Sites tab → right-click a request → "Fuzz..."
```
Opens ZAP's Fuzzer — mark a position in the request, attach a payload list, and run automated substitution testing (the direct equivalent of Intruder, without Community's throttling).

```
9. Tools → Options → HUD (Heads Up Display) → enable
```
Overlays ZAP's controls directly inside your proxied browser window — lets you trigger scans and see alerts without switching back to the desktop app.

```
10. Report → Generate Report...
```
Exports all findings from the session into a formatted report (HTML/PDF/XML/JSON) — ZAP's built-in reporting, free in every edition.

---

## 7. Quick Cheat Sheet

| Task | Burp Suite Community | OWASP ZAP |
|---|---|---|
| View intercepted traffic | Proxy → HTTP History | Sites tab / History tab |
| Manually edit & resend a request | Send to Repeater | "Open/Resend with Request Editor" |
| Automated payload substitution | Send to Intruder *(rate-limited)* | "Fuzz..." *(not throttled)* |
| Built-in automated vulnerability scan | Not available (Pro-only) | Attack → "Active Scan..." |
| Compare two requests/responses | Send to Comparer | Manual diff via History tab |
| Generate a findings report | Export via extensions/manual | Report → Generate Report |

**Typical Day 10 Manual Testing Workflow (lab/authorized environment):**
```
1. Configure browser proxy → 127.0.0.1:8080, install the tool's CA certificate
2. Browse the target application normally, logging in and using every feature once
3. Review the Proxy/HTTP History to understand the full set of real requests being made
4. Pick a suspicious parameter → Send to Repeater → manually test a few crafted values
5. For a small, specific test set → Send to Intruder/Fuzzer → run and compare response lengths/codes
6. Read through response headers, cookies, and error messages for anything leaking information
7. Export a report (ZAP) or document Repeater request/response pairs manually (Burp Community)
```

---

### Practice Suggestions for Day 10
1. Set up Burp Suite Community, configure your browser to proxy through it, and install its CA certificate.
2. Browse a lab web app (e.g., DVWA) fully through the proxy, then review the HTTP History to see every real request your actions generated.
3. Pick one login form request, send it to Repeater, and manually try 3–4 different payloads in the username/password fields — observe how responses differ.
4. Send the same request to Intruder, mark the password field as a position, and run a small (10–15 entry) payload list — time how long it takes, and note the Community rate limit in action.
5. Repeat the same manual workflow in OWASP ZAP, then run its free Active Scan against the same lab app and compare the Alerts tab results to what you found manually in Burp.
6. Use ZAP's Fuzzer with the same 10–15 entry payload list from step 4 and compare its speed against Burp Community's Intruder.
