# Day 3 — Network Discovery


**Tools**
● Nmap ● Masscan ● Netdiscover

**Topics**
● Host Discovery ● Port Scanning ● Service Detection ● OS Detection

A complete reference covering host discovery, port scanning, service detection, and OS detection — using Nmap, Masscan, and Netdiscover. (Topics are explained first, followed by the tools.)

---

## Table of Contents
1. [Core Concepts](#1-core-concepts)
2. [Host Discovery](#2-host-discovery)
3. [Port Scanning](#3-port-scanning)
4. [Service Detection](#4-service-detection)
5. [OS Detection](#5-os-detection)
6. [Tool: Nmap](#6-tool-nmap)
7. [Tool: Masscan](#7-tool-masscan)
8. [Tool: Netdiscover](#8-tool-netdiscover)
9. [Quick Cheat Sheet](#9-quick-cheat-sheet)

---

## 1. Core Concepts

**Network discovery** is the process of finding out what's actually on a network — which hosts are alive, what ports/services they're running, and what software/OS powers them. It's the first phase of both network administration (asset inventory) and penetration testing (reconnaissance).

The typical discovery workflow, in order:
```
1. Host Discovery      → "Which IPs are alive on this network?"
2. Port Scanning        → "Which ports are open on those IPs?"
3. Service Detection      → "What software/version is running on each open port?"
4. OS Detection             → "What operating system is the target running?"
```
Each stage narrows the picture and increases the "noise" (detectability) of the scan — host discovery is quiet and fast, OS detection is slower and more intrusive.

---

## 2. Host Discovery

**Host discovery** (a.k.a. "ping sweeping") determines which IP addresses on a network are actually up and responding, before wasting time port-scanning dead hosts.

### 2.1 How It Works
Common techniques, roughly from least to most reliable:
- **ICMP Echo (ping)** — sends an ICMP Echo Request; many firewalls block this.
- **ARP requests** — on a local network (same subnet), ARP is nearly 100% reliable since it can't be firewalled the way IP traffic can.
- **TCP SYN/ACK probes** — send a SYN or ACK to a common port (e.g. 80, 443) and see if anything responds.
- **UDP probes** — send to a UDP port and check for an ICMP "port unreachable" reply (implies host is alive).

### 2.2 Why It Matters
- Avoids wasting time/bandwidth scanning IPs that don't exist.
- On large networks (e.g. `/16`), scanning every port on every possible IP is impractical — discovery narrows the target list first.

---

## 3. Port Scanning

**Port scanning** probes a target's ports to determine which are **open**, **closed**, or **filtered** (blocked by a firewall, so no response).

### 3.1 Port States

| State | Meaning |
|---|---|
| **Open** | A service is actively listening and responding |
| **Closed** | Port is reachable but nothing is listening |
| **Filtered** | A firewall/ACL is blocking probes — can't tell if open or closed |
| **Unfiltered** | Reachable, but scanner can't determine open/closed (rare, ACK scans) |

### 3.2 Common Scan Techniques

| Scan Type | How It Works | Notes |
|---|---|---|
| **TCP Connect Scan** | Completes full 3-way handshake | Reliable but noisy/loggable, no root needed |
| **SYN Scan ("half-open")** | Sends SYN, reads SYN-ACK, then sends RST (never completes handshake) | Fast, stealthier, needs root/admin |
| **UDP Scan** | Sends UDP packets, waits for ICMP unreachable | Slow, since UDP is connectionless |
| **FIN/NULL/Xmas Scan** | Sends unusual flag combinations to sneak past basic firewalls | Used to evade simple packet filters |

### 3.3 Why Scan Ports?
- Identify attack surface (open ports = potential entry points).
- Verify firewall rules are working as intended.
- Confirm expected services are running (e.g., web server on 80/443).

---

## 4. Service Detection

Once a port is known to be **open**, service detection (a.k.a. **banner grabbing / version detection**) figures out *what* is actually running on it — not just "port 80 is open" but "Apache httpd 2.4.41 is running on port 80."

### 4.1 How It Works
- The scanner sends **probes** specific to known protocols and compares the response against a signature database.
- Simple form: connect and read the **banner** the service announces on connection (many services self-identify, e.g. SSH sends `SSH-2.0-OpenSSH_8.2p1` immediately).
- Advanced form (Nmap's `-sV`): tries multiple protocol probes, ranks results by likelihood.

### 4.2 Why It Matters
- Version numbers reveal known vulnerabilities (e.g., "Apache 2.4.29 has CVE-XXXX").
- Confirms a service isn't running on a non-standard port to hide it ("security through obscurity").

---

## 5. OS Detection

**OS Detection (fingerprinting)** attempts to determine a target's operating system based on subtle differences in how different OS network stacks respond to crafted packets.

### 5.1 How It Works
- Different OSes implement TCP/IP slightly differently: initial **TTL** values, **TCP window size**, **IP ID sequencing**, response to malformed packets, ordering of TCP options, etc.
- The scanner sends a series of unusual/edge-case probes and compares the fingerprint against a database of known OS signatures.

### 5.2 Two Broad Approaches

| Type | Description |
|---|---|
| **Active fingerprinting** | Scanner sends custom packets and analyzes responses (e.g., Nmap `-O`) — more accurate, more detectable |
| **Passive fingerprinting** | Just observes existing traffic without sending anything (e.g., `p0f`) — stealthier, less certain |

### 5.3 Limitations
- Firewalls, NAT, and traffic shaping can distort or block fingerprinting probes.
- Modern OSes increasingly resist fingerprinting or return generic results.
- Results are a **best guess with a confidence %**, not a guarantee.

---

## 6. Tool: Nmap

**Nmap ("Network Mapper")** is the industry-standard, all-in-one tool for host discovery, port scanning, service/version detection, and OS fingerprinting.

### 6.1 Key Commands
```bash
nmap 192.168.1.1                      # basic scan — top 1000 TCP ports on one host
nmap 192.168.1.0/24                     # scan an entire subnet

nmap -sn 192.168.1.0/24                    # host discovery only (ping sweep, no port scan)
nmap -sL 192.168.1.0/24                       # list targets only, no actual scan (DNS check)

nmap -p 80,443 192.168.1.1                  # scan specific ports
nmap -p- 192.168.1.1                           # scan ALL 65535 ports
nmap -F 192.168.1.1                              # fast scan — top 100 ports only

nmap -sS 192.168.1.1                          # SYN (stealth/half-open) scan — needs root
nmap -sT 192.168.1.1                            # TCP connect scan (no root required)
nmap -sU 192.168.1.1                              # UDP scan

nmap -sV 192.168.1.1                          # service/version detection
nmap -O 192.168.1.1                             # OS detection (needs root)
nmap -A 192.168.1.1                               # aggressive: OS + version + scripts + traceroute

nmap -T4 192.168.1.1                          # timing template (0=slowest/stealthiest, 5=fastest)
nmap -oN scan.txt 192.168.1.1                   # save output to a normal text file
```

> `-A` combines detection features into one convenient (but loud/slow) scan — great for labs, not for stealth.

---

## 7. Tool: Masscan

**Masscan** is an extremely fast, asynchronous port scanner — built to scan the **entire internet** in minutes by sending packets as fast as the network card allows. It trades depth (accuracy/features) for raw speed compared to Nmap.

### 7.1 Key Commands
```bash
masscan 192.168.1.0/24 -p80                    # scan a subnet for port 80
masscan 192.168.1.0/24 -p80,443,22                # scan multiple specific ports
masscan 192.168.1.0/24 -p1-65535                    # scan all ports on a subnet

masscan 192.168.1.0/24 -p80 --rate=1000              # control packets per second (rate-limit)
masscan 192.168.1.0/24 -p80 --rate=10000               # very fast scan (use carefully — can flood networks)

masscan 192.168.1.0/24 -p80 -oL results.txt              # save output to a list file
masscan 192.168.1.0/24 -p80 -oJ results.json                # save output as JSON

masscan 192.168.1.0/24 -p80 --banners                     # attempt basic banner grabbing
masscan -e eth0 192.168.1.0/24 -p80                          # specify network interface

masscan --top-ports 100 192.168.1.0/24                    # scan only the top 100 common ports
masscan --excludefile exclude.txt 192.168.1.0/24 -p80         # exclude specific IPs from the scan
```

> ⚠️ Masscan's default rate can be dangerously fast on real networks (it can act like a DoS if unrestricted) — always set `--rate` deliberately, especially on shared/production networks.

---

## 8. Tool: Netdiscover

**Netdiscover** is an active/passive **ARP-based** host discovery tool, primarily used on local networks (LANs). Since it works at Layer 2 (ARP), it's extremely reliable for finding live hosts on the same subnet — it can't be blocked by IP-level firewall rules.

### 8.1 Key Commands
```bash
netdiscover                          # auto-detect interface & scan local network (interactive)
netdiscover -r 192.168.1.0/24           # scan a specific IP range

netdiscover -i eth0                       # specify network interface
netdiscover -i eth0 -r 192.168.1.0/24        # combine interface + range

netdiscover -p                              # passive mode — only listen, don't send ARP requests
netdiscover -f                                # fast mode — skip a full scan cycle

netdiscover -c 3                          # number of scan cycles/repeats
netdiscover -s 1                            # delay (in ms) between ARP requests — slower/stealthier

netdiscover -L                          # display known OUI/vendor list, then exit
netdiscover -d                            # ignore local ARP cache, force fresh requests
```

> Netdiscover only works on **local/directly connected subnets** — since ARP doesn't route across networks, it's not useful for scanning remote/internet-facing hosts (use Nmap/Masscan for those).

---

## 9. Quick Cheat Sheet

| Task | Command |
|---|---|
| Ping sweep a subnet | `nmap -sn 192.168.1.0/24` |
| ARP-based local discovery | `netdiscover -r 192.168.1.0/24` |
| Fast full-port scan (huge range) | `masscan 10.0.0.0/8 -p80 --rate=1000` |
| Scan all TCP ports on one host | `nmap -p- 192.168.1.1` |
| Stealth SYN scan | `nmap -sS 192.168.1.1` |
| Service/version detection | `nmap -sV 192.168.1.1` |
| OS fingerprinting | `nmap -O 192.168.1.1` |
| All-in-one aggressive scan | `nmap -A 192.168.1.1` |
| Save scan results | `nmap -oN scan.txt 192.168.1.1` |

**Typical Recon Workflow (lab/authorized environment):**
```bash
netdiscover -r 192.168.1.0/24         # 1. Find live hosts on LAN
nmap -sn 192.168.1.0/24                 # 2. Confirm with a ping sweep
nmap -p- -T4 192.168.1.10                 # 3. Full port scan on a target
nmap -sV -O 192.168.1.10                    # 4. Service + OS detection on open ports
```

---

### Practice Suggestions for Day 3
1. Spin up 2–3 VMs on the same virtual network (e.g. one Metasploitable, one Ubuntu).
2. Run `netdiscover` to identify all live hosts by ARP.
3. Run `nmap -sn` on the same subnet and compare results with netdiscover.
4. Pick one target VM: run a full port scan (`-p-`), then `-sV` and `-O` on the open ports found.
5. Use `masscan` on the same subnet with a controlled `--rate` and compare speed/output against Nmap.
6. Try `-T2` vs `-T4` timing templates on Nmap and observe the speed/stealth trade-off.
