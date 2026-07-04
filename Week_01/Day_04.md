# Day 4 — DNS & OSINT

**Tools**
● Whois ● DNSRecon ● Fierce

**Topics**
● DNS Enumeration ● Zone Information ● Domain Intelligence

A complete reference covering DNS enumeration, zone information, and domain intelligence gathering (OSINT) — using Whois, DNSRecon, and Fierce. (Topics are explained first, followed by the tools.)

> ⚠️ **Ethical Note:** These techniques query publicly available information and public DNS infrastructure, so they're generally low-risk — but always get authorization before running enumeration against a target as part of any engagement, and respect rate limits so you don't disrupt the DNS servers you're querying.

---

## Table of Contents
1. [DNS Enumeration](#1-dns-enumeration)
2. [Zone Information](#2-zone-information)
3. [Domain Intelligence](#3-domain-intelligence)
4. [Tool: Whois](#4-tool-whois)
5. [Tool: DNSRecon](#5-tool-dnsrecon)
6. [Tool: Fierce](#6-tool-fierce)
7. [Quick Cheat Sheet](#7-quick-cheat-sheet)

---

## 1. DNS Enumeration

**DNS enumeration** is the process of discovering all the DNS records, subdomains, and infrastructure associated with a target domain. It's usually the very first step in reconnaissance (OSINT) because DNS data is public by design — it has to be, for the internet to work — but it often reveals far more than intended.

### 1.1 What You're Looking For
- **Subdomains** — `mail.example.com`, `vpn.example.com`, `dev.example.com`, `staging.example.com` — each one is a potential extra attack surface, and dev/staging subdomains are notoriously under-secured.
- **IP addresses** behind each subdomain — reveals hosting provider, possibly the real server IP behind a CDN/WAF like Cloudflare.
- **Mail servers (MX records)** — reveals the email provider (Google Workspace, Microsoft 365, self-hosted).
- **Nameservers (NS records)** — reveals the DNS/hosting provider.
- **TXT records** — often contain SPF/DKIM/DMARC (email security config) or ownership-verification strings (e.g., for Google, AWS) that leak which third-party services a company uses.

### 1.2 Common Enumeration Techniques
| Technique | Description |
|---|---|
| **Brute-forcing** | Try a wordlist of common subdomain names (`www`, `mail`, `api`, `dev`...) against the target domain |
| **Zone transfer attempt** | Ask a nameserver to hand over its *entire* zone file (works only if misconfigured — see Section 2) |
| **Reverse DNS lookup** | Given an IP, find what domain(s) resolve to it |
| **Search engine / certificate transparency** | Passive methods — searching public cert logs (e.g., crt.sh) or search engines for `site:example.com` |
| **Public record queries** | Standard record lookups (A, AAAA, MX, TXT, NS, CNAME, SOA) |

### 1.3 Why It Matters
A single forgotten subdomain (like an old admin panel or unpatched dev server) is one of the most common real-world entry points in breaches — DNS enumeration is how both attackers *and* defenders find these before something bad happens.

---

## 2. Zone Information

A **DNS zone** is a portion of the DNS namespace that a specific nameserver is authoritative for — essentially, the "master record book" for a domain, stored in a **zone file**.

### 2.1 What's in a Zone File
A zone file starts with an **SOA (Start of Authority)** record, defining:
- The primary nameserver for the zone.
- The admin contact email.
- **Serial number** (version — incremented on each change).
- **Refresh/Retry/Expire/TTL** timers — control how secondary (backup) nameservers sync and cache data.

Followed by all the individual records for that zone: A, AAAA, MX, NS, CNAME, TXT, SRV, etc.

### 2.2 Zone Transfers (AXFR)
A **zone transfer** is the mechanism by which a **secondary/slave nameserver** copies the entire zone file from the **primary/master nameserver**, to stay in sync. This is meant to happen *only* between authorized nameservers.

**The vulnerability:** If a nameserver is **misconfigured** to allow zone transfers to *anyone* who asks, an attacker (or student in a lab!) can request `AXFR` and receive the domain's **entire DNS record set in one shot** — every subdomain, every internal IP, all at once. This is one of the most impactful (and avoidable) DNS misconfigurations.

```bash
dig axfr @ns1.example.com example.com     # attempt a zone transfer (works only if misconfigured)
```

A properly configured DNS server will **refuse** this request from unauthorized IPs — which is the expected, secure outcome.

### 2.3 Why Zone Information Matters
- A successful zone transfer is essentially "game over" for the enumeration phase — it hands over the full DNS map instantly instead of requiring slow brute-forcing.
- Reviewing SOA timers (refresh/expire) helps understand how quickly DNS changes propagate — relevant during incident response (e.g., after taking down a malicious domain's records).

---

## 3. Domain Intelligence

**Domain intelligence** is the broader OSINT process of piecing together who owns a domain, how it's hosted, and what infrastructure/organization sits behind it — combining DNS data with registration and public records.

### 3.1 What You're Looking For
- **Registrant information** — who registered the domain, when, and through which registrar (via Whois).
- **Registration/expiry dates** — old domains suggest established infrastructure; domains registered very recently alongside a live campaign can be a red flag (common in phishing investigations).
- **Registrar & nameserver provider** — reveals which hosting/DNS company manages the domain (GoDaddy, Cloudflare, AWS Route 53, etc.).
- **Related domains** — same registrant email/organization often owns multiple related domains — useful for mapping an organization's full external footprint.
- **Historical data** — previous ownership, previous IPs (via services like archived Whois or passive DNS databases).

### 3.2 Privacy Protections
Many registrars now offer **WHOIS privacy/proxy services** that mask the real registrant's contact details behind the registrar's own information — meaning much of this data may be redacted for privacy-conscious or GDPR-covered domains. This is expected and not itself suspicious.

### 3.3 Why It Matters
- Attribution — figuring out who actually controls a domain (important in phishing/fraud investigations).
- Attack surface mapping — for a large organization, finding *every* domain and subdomain it owns, not just the main one.
- Due diligence — verifying a domain's legitimacy/age before trusting it (common in threat intelligence).

---

## 4. Tool: Whois

**Whois** queries public registry databases to retrieve registration details about a domain name or IP address — registrant, registrar, dates, nameservers.

### 4.1 Key Commands
```bash
whois example.com                    # full registration info for a domain
whois 8.8.8.8                          # registration info for an IP address (who owns that IP block)

whois example.com | grep -i "registrar"     # filter to just the registrar name
whois example.com | grep -i "name server"     # filter to just the nameservers
whois example.com | grep -i "creation date"     # filter to registration date
whois example.com | grep -i "expiry"              # filter to expiry date
whois example.com | grep -i "registrant"            # filter to registrant details (if not privacy-protected)

whois -h whois.iana.org example.com         # query a specific whois server directly
whois --verbose example.com                    # more detailed output (varies by client)
```

> Results vary depending on the domain's TLD (`.com` vs `.io` vs country-code TLDs use different registries) and whether privacy protection is enabled.

---

## 5. Tool: DNSRecon

**DNSRecon** is a Python-based DNS enumeration tool that automates many of the techniques from Section 1 — standard record lookups, zone transfer attempts, subdomain brute-forcing, and reverse lookups — all in one tool.

### 5.1 Key Commands
```bash
dnsrecon -d example.com                        # standard enumeration (all default record types)
dnsrecon -d example.com -t std                    # explicitly run standard record scan

dnsrecon -d example.com -t axfr                  # attempt a zone transfer against the domain's nameservers

dnsrecon -d example.com -t brt -D wordlist.txt      # brute-force subdomains using a wordlist
dnsrecon -d example.com -t brt                        # brute-force using the tool's default wordlist

dnsrecon -d example.com -t rvl -r 192.168.1.0/24        # reverse DNS lookup across an IP range

dnsrecon -d example.com -t srv                    # enumerate SRV records (useful for finding services like VoIP, LDAP)
dnsrecon -d example.com -t mx                       # enumerate mail server (MX) records only

dnsrecon -d example.com -n 8.8.8.8                    # specify a custom nameserver to query against

dnsrecon -d example.com -x report.xml               # export results to an XML report
dnsrecon -d example.com -c report.csv                 # export results to CSV
```

---

## 6. Tool: Fierce

**Fierce** is a lightweight, fast DNS reconnaissance tool focused primarily on discovering **non-contiguous IP space and subdomains** of a target — it was built specifically to help find hosts on a corporate network quickly, without the noise of a full port scan.

### 6.1 Key Commands
```bash
fierce --domain example.com                        # basic scan — subdomain discovery + zone transfer attempt

fierce --domain example.com --subdomains www,mail,dev    # check specific subdomain names
fierce --domain example.com --wordlist wordlist.txt         # brute-force using a custom wordlist

fierce --domain example.com --dns-servers 8.8.8.8              # use a specific DNS server for queries

fierce --domain example.com --range 192.168.1.0-192.168.1.255    # limit search to a specific IP range

fierce --domain example.com --traverse 10                          # traverse IPs near found hosts (+/-10) looking for neighbors

fierce --domain example.com --connect                                 # attempt to connect to discovered hosts to verify they're live

fierce --domain example.com --tcp                                       # use TCP instead of UDP for DNS queries (bypasses some UDP filtering)
```

> Fierce is intentionally simple and fast compared to DNSRecon — good for a quick first pass before running a deeper, more thorough tool.

---

## 7. Quick Cheat Sheet

| Task | Command |
|---|---|
| Get domain registration info | `whois example.com` |
| Attempt a zone transfer | `dig axfr @ns1.example.com example.com` or `dnsrecon -d example.com -t axfr` |
| Full automated DNS enumeration | `dnsrecon -d example.com` |
| Brute-force subdomains | `dnsrecon -d example.com -t brt -D wordlist.txt` or `fierce --domain example.com --wordlist wordlist.txt` |
| Quick subdomain/zone-transfer check | `fierce --domain example.com` |
| Reverse DNS lookup on a range | `dnsrecon -d example.com -t rvl -r 192.168.1.0/24` |
| Check MX records | `dnsrecon -d example.com -t mx` or `whois`/`dig MX` |

**Typical OSINT/DNS Recon Workflow (lab/authorized environment):**
```bash
whois example.com                          # 1. Identify registrar, nameservers, registration dates
dig axfr @ns1.example.com example.com          # 2. Try zone transfer (often fails on hardened targets)
fierce --domain example.com                      # 3. Quick pass — subdomains + zone transfer attempt
dnsrecon -d example.com -t brt -D wordlist.txt      # 4. Deeper brute-force enumeration of subdomains
dnsrecon -d example.com -t mx                          # 5. Identify mail infrastructure
```

---

### Practice Suggestions for Day 4
1. Run `whois` on a domain you own or a well-known public domain and identify the registrar, nameservers, and creation date.
2. Attempt a zone transfer (`dig axfr`) against a few real domains and observe that modern, properly configured servers refuse it.
3. Set up a test DNS zone in a lab (e.g., using BIND) and intentionally misconfigure it to allow AXFR — then confirm you can pull it with `dnsrecon -t axfr`.
4. Run `fierce` and `dnsrecon` against the same lab domain and compare how many subdomains each discovers.
5. Use `dnsrecon -t mx` and `-t srv` to map out a domain's email and service infrastructure.
6. Cross-reference a discovered subdomain's IP with `whois <IP>` to identify its hosting provider.
