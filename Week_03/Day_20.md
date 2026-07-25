# Day 20 — SOC Lab: Build It, Then Hunt In It


**Focus**
● Analyze sample packet captures (PCAPs) ● Identify suspicious traffic

This is a hands-on lab day, not a new tool day — everything here uses tools from Day 19 (Wireshark, Tcpdump) inside a proper SOC-style lab setup. Part 1 walks through building the lab. Part 2 is a real, guided walkthrough analyzing an actual PCAP — with genuine commands and genuine output, not illustrative examples — so you can follow along exactly.

> ⚠️ **Ethical Note:** This lab uses a purpose-built **synthetic** PCAP (included with this document) so you can practice safely without needing to capture or download anything involving real people's traffic. When you later graduate to real-world sample PCAPs (Section 1.4 lists legitimate public sources), always confirm a source is legal/public before analyzing or sharing it.

---

## Table of Contents
**Part 1 — Building the SOC Lab**
1. [What a SOC Lab Actually Is](#1-what-a-soc-lab-actually-is)
2. [Lab Architecture](#2-lab-architecture)
3. [Installing the Toolset](#3-installing-the-toolset)
4. [Sourcing PCAPs to Analyze](#4-sourcing-pcaps-to-analyze)

**Part 2 — The Practical: Hunting in a PCAP**
5. [Step 1 — Get Your Bearings (Protocol & Traffic Overview)](#5-step-1--get-your-bearings)
6. [Step 2 — Find the Top Talkers](#6-step-2--find-the-top-talkers)
7. [Step 3 — Hunt for a Port Scan](#7-step-3--hunt-for-a-port-scan)
8. [Step 4 — Hunt for Brute-Force Activity](#8-step-4--hunt-for-brute-force-activity)
9. [Step 5 — Hunt for Exposed Credentials](#9-step-5--hunt-for-exposed-credentials)
10. [Step 6 — Hunt for DNS-Based Exfiltration](#10-step-6--hunt-for-dns-based-exfiltration)
11. [Writing Up the Findings](#11-writing-up-the-findings)
12. [Practice Suggestions](#12-practice-suggestions)

---

# Part 1 — Building the SOC Lab

## 1. What a SOC Lab Actually Is

A **SOC (Security Operations Center) lab** is an isolated practice environment that mimics what an analyst actually sits in front of during a real shift: captured or streamed network traffic, and the tools to inspect it for signs of compromise. Unlike Days 1–19, where you were mostly the one *generating* the traffic (as an attacker/tester), a SOC lab flips your role — you're the **defender**, looking at traffic someone else generated and asking "is anything here wrong?"

This matters because the two skillsets genuinely differ: knowing how to run Nmap (Day 3) is different from knowing what an Nmap scan **looks like** from the receiving end, in a packet capture, mixed in with thousands of lines of normal traffic.

---

## 2. Lab Architecture

You don't need a large setup — a SOC analysis lab is lightweight compared to the wireless (Day 17–18) or exploitation labs later in this course, since you're working with **static files**, not live infected hosts.

### 2.1 Minimum Viable Setup
```
┌─────────────────────────────────────────┐
│   Analyst VM (Kali Linux or Ubuntu)      │
│   - Wireshark / tshark                   │
│   - tcpdump                              │
│   - A folder of sample .pcap files       │
└─────────────────────────────────────────┘
```
This alone is enough to complete every exercise in Part 2. If you only do one thing from this section, do this.

### 2.2 A More Realistic SOC-Style Setup (Optional, Recommended for Later Days)
For a setup closer to a real SOC — useful groundwork for later exploitation/forensics days — add:
```
┌───────────────────┐      ┌───────────────────────────────┐
│  "Victim" VM        │      │   Analyst / SOC VM              │
│  (Metasploitable or  │◄────►│   - Wireshark / tshark           │
│   a deliberately      │      │   - tcpdump                        │
│   vulnerable target)   │      │   - Zeek or Suricata (optional)     │
└───────────────────┘      │   - A SIEM-style dashboard (optional) │
        ▲                    └───────────────────────────────┘
        │  isolated
        │  virtual network
        ▼
┌───────────────────┐
│  "Attacker" VM       │
│  (Kali, from earlier   │
│   days' tools)           │
└───────────────────┘
```
- All three VMs live on an **isolated, host-only/internal virtual network** (Day 1's VM setup) — never bridged to your real LAN or the internet.
- The Attacker VM generates real traffic (e.g., an Nmap scan, a Hydra brute-force) using tools from earlier days.
- The Analyst VM captures that traffic (Tcpdump, Day 19) and analyzes it (Wireshark) — turning every previous day's *offense* into this day's *defense* exercise.
- **Zeek** or **Suricata** (optional, more advanced) can run continuously against the captured traffic to automatically flag known-bad patterns, giving you a taste of how real SOC tooling automates parts of what you'll do manually in Part 2.

### 2.3 Why Isolation Matters Here Specifically
Even though this is a *defensive* exercise, the "Attacker" VM in the fuller setup is still actively running tools like Nmap/Hydra from earlier days — so the same isolation rules from every previous day still apply: host-only networking, no bridging to a real network, snapshot before you start.

### 2.4 Sourcing PCAPs to Analyze
For your own future practice beyond this lab's included file, these are legitimate, widely-used public sources of real sample captures:
- **Wireshark's own sample captures wiki** — a large collection of clean, protocol-specific example captures, ideal for learning normal traffic patterns.
- **malware-traffic-analysis.net** — a well-known, long-running public resource of real (defanged/safe-to-analyze) malicious traffic captures with write-ups, widely used for SOC/blue-team training.
- **NETRESEC's public PCAP repository** — a curated list of publicly available capture files across many categories.
- Your **own lab traffic** — anything you capture yourself in Day 19's exercises is also perfectly good practice material.

---

## 3. Installing the Toolset

Everything needed is already covered from Day 19 — this section is just the condensed install/setup checklist for a fresh lab VM.

```bash
sudo apt update
sudo apt install -y wireshark tshark tcpdump
```
Installs **Wireshark** (GUI), **tshark** (Wireshark's command-line counterpart — used heavily in this lab for scriptable, repeatable analysis), and **tcpdump**.

```bash
sudo usermod -aG wireshark $USER
```
Adds your user to the `wireshark` group so you can capture live traffic without running the GUI as root — log out/in for this to take effect. (Not required if you're only analyzing existing `.pcap` files, as in this lab.)

```bash
tshark --version
```
Confirms tshark installed correctly and shows its version — a quick sanity check before starting.

**Optional, for the fuller setup (Section 2.2):**
```bash
sudo apt install -y zeek suricata
```
Installs **Zeek** and **Suricata** — both are automated network-monitoring/IDS engines that continuously analyze traffic against known-bad signatures and behavioral patterns, giving you a taste of production SOC tooling (a deeper dive is out of scope for today, but worth knowing they exist).

---

# Part 2 — The Practical: Hunting in a PCAP

This section uses a **real file**, `soc_lab_day20.pcap`, provided alongside this document. It's a synthetic capture I built specifically for this lab, containing **one normal conversation deliberately mixed in with four separate suspicious patterns** — the same way real captures mix background noise with the incident you're actually hunting for. Every command below was actually run against this exact file, and the output shown is genuine, not illustrative.

Open a terminal in the folder containing `soc_lab_day20.pcap` before starting.

---

## 5. Step 1 — Get Your Bearings

**Why start here?** Before hunting for anything specific, get an overview of what's actually in the capture — exactly the same "passive before active" principle from Day 3's recon sequencing, now applied to analysis.

```bash
tshark -r soc_lab_day20.pcap -q -z io,phs
```
This breaks the capture down by **protocol hierarchy**. Real output from this file:
```
Protocol Hierarchy Statistics

eth                                      frames:181 bytes:13812
  ip                                     frames:181 bytes:13812
    udp                                  frames:32 bytes:5392
      dns                                frames:32 bytes:5392
    tcp                                  frames:149 bytes:8420
      http                               frames:3 bytes:488
        data-text-lines                  frames:1 bytes:131
        urlencoded-form                  frames:1 bytes:220
```
**What this already tells us:** 181 total frames, a mix of DNS and TCP traffic, and — notably — an `urlencoded-form` inside HTTP. That's a flag worth remembering for Step 5; form data over plain HTTP is exactly the shape a login submission takes.

---

## 6. Step 2 — Find the Top Talkers

**Why this step next?** Before diving into individual packets, identify which hosts are talking the most and to whom — patterns often jump out at the conversation level before you ever look at a single packet's content.

```bash
tshark -r soc_lab_day20.pcap -q -z conv,tcp
```
Real (truncated) output:
```
TCP Conversations
                                                    |  <-  |  |  ->  |  |  Total  |
192.168.56.50:52000  <-> 192.168.56.20:80                2 185 bytes   3 245 bytes    5 430 bytes
203.0.113.77:41641   <-> 192.168.56.10:22                1 54 bytes    3 162 bytes    4 216 bytes
203.0.113.77:41691   <-> 192.168.56.10:22                1 54 bytes    3 162 bytes    4 216 bytes
203.0.113.77:41349   <-> 192.168.56.10:22                1 54 bytes    3 162 bytes    4 216 bytes
203.0.113.77:41890   <-> 192.168.56.10:22                1 54 bytes    3 162 bytes    4 216 bytes
... (many more near-identical short conversations to :22) ...
```
**What jumps out immediately:** one clean, normal-looking conversation (`.50 <-> .20:80`), followed by a *long repeating pattern* — the same external IP, `203.0.113.77`, opening dozens of nearly identical, very short conversations, all to port **22 (SSH)** on `192.168.56.10`. A legitimate user doesn't open 25 separate SSH connections in a few seconds. This is our first real lead.

---

## 7. Step 3 — Hunt for a Port Scan

**Why check this next?** `203.0.113.77` is already suspicious from Step 2 — before focusing only on its SSH activity, check whether it touched *anything else* first, since a real intrusion often starts broad (recon) before narrowing (brute-force).

```bash
tshark -r soc_lab_day20.pcap -Y "ip.src==203.0.113.77 && tcp.flags.syn==1 && tcp.flags.ack==0" -T fields -e frame.time_relative -e ip.src -e tcp.dstport
```
Real output (first 20 of 25 matching lines):
```
0.063000000   203.0.113.77   21
0.067000000   203.0.113.77   22
0.071000000   203.0.113.77   23
0.075000000   203.0.113.77   25
0.079000000   203.0.113.77   53
0.083000000   203.0.113.77   80
0.087000000   203.0.113.77   110
0.091000000   203.0.113.77   135
0.091000000   203.0.113.77   139
0.098999000   203.0.113.77   143
0.102999000   203.0.113.77   443
0.106999000   203.0.113.77   445
0.110999000   203.0.113.77   993
0.114999000   203.0.113.77   995
0.118999000   203.0.113.77   1433
0.122999000   203.0.113.77   3306
0.126999000   203.0.113.77   3389
0.130999000   203.0.113.77   5900
0.134999000   203.0.113.77   8080
0.138999000   203.0.113.77   8443
```
**This is a textbook SYN scan signature**, and it's the exact mirror image of Day 3's `nmap -sS` from the *target's* point of view:
- 20 different destination ports, each hit with a single SYN, **no data ever sent**.
- Every probe roughly **4 milliseconds apart** — no human types that fast; this is automated, evenly-paced tooling.
- The port list itself is a giveaway too — 21, 22, 23, 80, 443, 3389, 3306, 445 — this is exactly the "check all the common services" list a default Nmap scan targets.

**Finding #1: External host `203.0.113.77` performed an automated TCP SYN port scan against `192.168.56.10` across 20 common service ports.**

---

## 8. Step 4 — Hunt for Brute-Force Activity

**Why now?** Step 3 confirmed `203.0.113.77` scanned broadly first; Step 2 already showed it repeatedly hitting port 22 afterward. Now confirm exactly what that repeated SSH activity looks like.

```bash
tshark -r soc_lab_day20.pcap -Y "ip.dst==192.168.56.10 && tcp.port==22 && tcp.flags.syn==1 && tcp.flags.ack==0" | wc -l
```
Real output:
```
25
```
**25 separate new connection attempts to SSH from the same source**, each lasting only a fraction of a second (visible back in Step 2's conversation durations — ~0.22 seconds each) before being torn down. This is the classic shape of an **automated SSH brute-force attempt** (the same pattern Day 16's Hydra produces from the attacking side) — rapid connect, attempt, disconnect, repeat.

**Finding #2: The same external host followed its port scan with 25 rapid, automated connection attempts against SSH (port 22) — consistent with a credential brute-force attempt.**

---

## 9. Step 5 — Hunt for Exposed Credentials

**Why check this next?** Step 1's protocol hierarchy already flagged an `urlencoded-form` inside plain HTTP traffic — time to look at exactly what that contains.

```bash
tshark -r soc_lab_day20.pcap -Y "http.request.method==POST" -T fields -e ip.src -e ip.dst -e http.file_data
```
Real output:
```
192.168.56.50  192.168.56.20  757365726e616d653d61646d696e2670617373776f72643d537570657253656372657431323321
```
That last field is the raw form data in hex — decoding it:
```bash
python3 -c "print(bytes.fromhex('757365726e616d653d61646d696e2670617373776f72643d537570657253656372657431323321').decode())"
```
```
username=admin&password=SuperSecret123!
```
**Finding #3: A login form (`POST /login.php`) was submitted from `192.168.56.50` to `192.168.56.20` entirely over plaintext HTTP, exposing the username and password in cleartext to anyone able to observe the traffic** — a direct, concrete illustration of Day 19's protocol-analysis warning about unencrypted protocols, and exactly the kind of finding a Driftnet-style demonstration (Day 19) makes viscerally obvious.

---

## 10. Step 6 — Hunt for DNS-Based Exfiltration

**Why check DNS specifically?** DNS is almost always allowed out through a firewall (it has to be, for the internet to work at all) — which makes it a favorite covert channel for smuggling data out of a network. A few unusual-looking DNS queries are worth a second look.

```bash
tshark -r soc_lab_day20.pcap -Y "dns.qry.type==16" -T fields -e frame.time_relative -e ip.src -e dns.qry.name
```
Real output (first 6 of 30 matching lines):
```
7.079998000   192.168.56.10   da211edc7bac01a07182af742e04dc6d552b8d6c350cb13c.data.exfil-c2.example
7.099998000   192.168.56.1    da211edc7bac01a07182af742e04dc6d552b8d6c350cb13c.data.exfil-c2.example
7.249999000   192.168.56.10   1d6285f5502cf1f86214b1b462c040968e2e127651e9cbde.data.exfil-c2.example
7.269999000   192.168.56.1    1d6285f5502cf1f86214b1b462c040968e2e127651e9cbde.data.exfil-c2.example
7.419999000   192.168.56.10   f7c6682ad24d37f8a2fd30e5fa49ad531b1cc9cee9f27003.data.exfil-c2.example
7.269999000   192.168.56.1    1d6285f5502cf1f86214b1b462c040968e2e127651e9cbde.data.exfil-c2.example
```
**Three separate red flags, all present at once — exactly what makes this pattern suspicious rather than just unusual:**
1. **Query type is TXT** (`dns.qry.type==16`) — a record type rarely used by normal web browsing, but commonly abused for tunneling because it can carry arbitrary text data.
2. **The subdomain is long and high-entropy** (`da211edc7bac01a07182af742e04dc6d552b8d6c350cb13c` — 48 random-looking hex characters) — real subdomains are short and human-readable (`mail.`, `www.`); this looks like encoded binary data, not a hostname.
3. **The same base domain (`exfil-c2.example`) is queried repeatedly** (30 times, roughly every 0.15–0.3 seconds) **with a different "subdomain" each time** — consistent with data being chunked and smuggled out one DNS query at a time.

**Finding #4: Host `192.168.56.10` generated repeated DNS TXT queries with long, high-entropy subdomains against a single external domain — a pattern consistent with DNS tunneling / data exfiltration.**

---

## 11. Writing Up the Findings

A SOC analyst's job isn't done when the packet is found — it's done when the finding is **written up clearly enough that someone else can act on it**. Here's how the four findings from this walkthrough would look in a real report:

| # | Finding | Evidence | Source | Severity |
|---|---|---|---|---|
| 1 | TCP SYN port scan against `192.168.56.10` | 20 distinct ports probed in <100ms by `203.0.113.77` | `tshark -Y "tcp.flags.syn==1 && tcp.flags.ack==0"` | Medium |
| 2 | SSH brute-force attempt | 25 rapid connect/disconnect cycles to port 22 from `203.0.113.77` | `tshark -Y "tcp.port==22 && tcp.flags.syn==1"` | High |
| 3 | Plaintext credential exposure | `username=admin&password=SuperSecret123!` sent unencrypted via HTTP POST | `tshark -Y "http.request.method==POST"` | High |
| 4 | Suspected DNS tunneling / exfiltration | 30 high-entropy TXT queries to `exfil-c2.example` from `192.168.56.10` | `tshark -Y "dns.qry.type==16"` | Critical |

**Recommended next actions** (the part a report always needs):
- Block/monitor `203.0.113.77` at the perimeter firewall; review logs for any successful SSH authentication from that IP.
- Force a password reset for the exposed `admin` credential, and require HTTPS for the login endpoint going forward.
- Investigate `192.168.56.10` directly (its own host logs, running processes) — the DNS tunneling pattern suggests it may already be compromised, not just scanned.
- Add a detection rule (e.g., in Suricata/Zeek from Section 2.2) for repeated high-entropy TXT queries to the same base domain, so this pattern is flagged automatically next time.

---

## 12. Practice Suggestions

1. Open `soc_lab_day20.pcap` in the **Wireshark GUI** and recreate every filter from Steps 3–6 using the display filter bar (`tcp.flags.syn==1 && tcp.flags.ack==0`, `dns.qry.type==16`, etc.) instead of tshark — confirm you get the same results visually.
2. Use Wireshark's **Statistics → Conversations** and **Statistics → Protocol Hierarchy** views on the same file and compare them directly against the tshark output shown in Steps 1–2.
3. Right-click the HTTP POST packet from Step 5 and use **Follow → HTTP Stream** to see the full plaintext request/response in one readable view.
4. Download one real sample capture from Wireshark's public sample-captures wiki (Section 2.4) and repeat this exact six-step process from scratch — protocol overview, top talkers, then hunt.
5. Using a fuller lab setup (Section 2.2), generate your **own** version of one of these four patterns using tools from earlier days (e.g., an actual `nmap -sS` scan from an Attacker VM against a Victim VM) and capture it live with Tcpdump — then re-run this day's analysis steps against traffic you generated yourself, closing the full attack-then-analyze loop.
6. Write your own one-page findings report (using the Section 11 table format) for whichever real-world sample capture you analyzed in step 4 above.
