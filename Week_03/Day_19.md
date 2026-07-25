# Day 19 — Network & Packet Analysis


**Tools**
● Wireshark ● Tcpdump ● Ettercap ● Netcat ● Socat ● Driftnet

**Topics**
● Packet Capture ● Protocol Analysis ● Network Troubleshooting

A complete reference covering packet capture, protocol analysis, and network troubleshooting — using Wireshark, Tcpdump, Ettercap, Netcat, Socat, and Driftnet. (Topics are explained first, followed by the tools — each explained by how it works and what it's used for, then its most important commands.)

> ⚠️ **Ethical Note:** Packet capture and traffic interception tools can expose other people's private data (credentials, messages, browsing activity) the instant you point them at a shared network. Only capture/intercept traffic on networks you own or administer, or with explicit written authorization — capturing traffic that isn't yours to see is illegal in most jurisdictions even if no other "attack" occurs.

---

## Table of Contents
1. [Packet Capture](#1-packet-capture)
2. [Protocol Analysis](#2-protocol-analysis)
3. [Network Troubleshooting](#3-network-troubleshooting)
4. [Tool: Tcpdump](#4-tool-tcpdump)
5. [Tool: Wireshark](#5-tool-wireshark)
6. [Tool: Ettercap](#6-tool-ettercap)
7. [Tool: Netcat](#7-tool-netcat)
8. [Tool: Socat](#8-tool-socat)
9. [Tool: Driftnet](#9-tool-driftnet)
10. [Quick Cheat Sheet](#10-quick-cheat-sheet)

---

## 1. Packet Capture

**Packet capture** is the process of recording the raw data traveling across a network, frame by frame, exactly as it appears on the wire — everything from Day 2's OSI/TCP-IP theory made directly observable.

### 1.1 How Capture Actually Happens
- A network interface normally only processes traffic addressed **to it**. To capture everything passing by (not just your own traffic), the interface must be switched into **promiscuous mode** — accepting and passing up every frame it physically receives, regardless of destination.
- On a **switched** network, promiscuous mode alone often isn't enough — a switch only forwards traffic to the port it's actually destined for, so a plain "connect and listen" capture will mostly see only your own machine's traffic. This is exactly why techniques like Ettercap's ARP spoofing (Section 6) exist: to redirect *other* devices' traffic through your machine first, making it visible to capture at all.
- Captured data is typically saved in the **pcap/pcapng** format — a standard file format that virtually every capture and analysis tool in this space (Wireshark, Tcpdump, Kismet from Day 17, hcxdumptool from Day 18) can read and write, making captures fully portable between tools.

### 1.2 Why It Matters
- Every layer of theory from Day 2 (TCP handshakes, HTTP requests, DNS lookups) becomes **directly visible and verifiable** once you can capture real packets — this is where abstract networking concepts turn into concrete evidence.
- It's the foundation both protocol analysis (Section 2) and network troubleshooting (Section 3) are built on — you can't analyze or diagnose what you haven't captured.

---

## 2. Protocol Analysis

**Protocol analysis** is the process of interpreting captured packets — decoding the raw bytes back into the structured, human-readable meaning defined by each protocol's specification (TCP, HTTP, DNS, TLS, and so on).

### 2.1 What "Analysis" Actually Involves
- **Header decoding** — turning raw bytes into recognizable fields: source/destination IP, port numbers, TCP flags (SYN, ACK, FIN), sequence numbers.
- **Following a conversation** — reconstructing an entire exchange (e.g., a full HTTP request and its response) from the individual packets that carried it, rather than viewing them as isolated frames.
- **Spotting anomalies** — traffic that doesn't match expected protocol behavior: malformed packets, unexpected port usage (e.g., HTTP traffic on a non-standard port), or repeated retransmissions suggesting packet loss.
- **Content inspection** — for unencrypted protocols, viewing the actual data being transmitted (HTTP bodies, plaintext FTP/Telnet credentials); for encrypted protocols (HTTPS/TLS), only the metadata (handshake details, certificate info, timing) remains visible without the private key.

### 2.2 Why It Matters
- This is the skill that turns a pile of captured packets into an actual **finding** — confirming a suspected data exfiltration, proving a plaintext credential was transmitted, or understanding exactly how an application behaves over the network.
- Directly extends Day 10's request/response analysis concept (examining HTTP traffic through a proxy) down to the raw packet level, covering *every* protocol, not just HTTP.

---

## 3. Network Troubleshooting

**Network troubleshooting** uses the same packet capture and analysis skills, but points them at a different goal: not finding a security weakness, but diagnosing **why something isn't working** — a slow connection, a failed request, a service that won't connect.

### 3.1 Common Things Packet Capture Reveals During Troubleshooting

| Symptom | What to Look For in a Capture |
|---|---|
| Connection fails immediately | No SYN-ACK response — suggests a firewall block or the service isn't listening |
| Connection is slow | Retransmissions, duplicate ACKs, or a large gap in time between request and response |
| Intermittent failures | Packet loss patterns, or a specific host/route consistently failing |
| Application "hangs" | A completed TCP handshake but no application-layer response — points to an app-layer issue, not a network one |
| DNS-related failures | No DNS response at all, or a response pointing to an unexpected/incorrect IP |

### 3.2 Why It Matters
- The exact same skillset used for security analysis (Section 2) is what a network/system administrator uses daily just to keep systems running — this dual-purpose nature is why packet analysis tools are considered core, general-purpose IT skills, not just security tools.
- Being able to say "the network delivered the request correctly, so the problem is in the application" (or vice versa) is often the single fastest way to correctly assign an issue to the right team.

---

## 4. Tool: Tcpdump

**Tcpdump** is a lightweight, command-line packet capture tool — the standard, always-available choice for capturing traffic directly on a server or over an SSH session, where a full graphical tool isn't practical.

### 4.1 How It Works
- Tcpdump uses **libpcap** (the same underlying capture library Wireshark uses) to put an interface into promiscuous mode and capture raw packets.
- It can either print a live, real-time summary of each packet directly to the terminal, or write the full raw capture to a `.pcap` file for later, deeper analysis (commonly in Wireshark).
- Its **filter syntax** (Berkeley Packet Filter / BPF) lets you narrow capture down to exactly the traffic you care about *before* it's even captured — critical on busy networks where capturing everything would be overwhelming.

### 4.2 What It's Used For
- **Remote/server-side capture** — since it's command-line and lightweight, it's the natural choice when you're working over SSH on a machine with no GUI.
- **Quick, targeted troubleshooting** — a fast way to confirm "is this traffic even arriving at this server" without needing to open a full analysis tool.
- **Capturing to a file** for later, more detailed analysis in Wireshark — a very common two-step workflow (`tcpdump` to capture, Wireshark to analyze).

### 4.3 Important Commands

```bash
sudo tcpdump -i eth0
```
The core command — captures and prints live traffic on interface `eth0` to the terminal. This is the starting point for almost every Tcpdump session.

```bash
sudo tcpdump -i eth0 -w capture.pcap
```
`-w` writes the **raw capture to a file** instead of printing a summary — the standard way to save traffic for later analysis in Wireshark.

```bash
sudo tcpdump -r capture.pcap
```
`-r` **reads** a previously saved capture file back for review, instead of capturing live traffic.

```bash
sudo tcpdump -i eth0 port 80
```
Filters capture to only **port 80** traffic — narrowing focus to a specific service instead of all traffic.

```bash
sudo tcpdump -i eth0 host 192.168.1.10
```
Filters to only traffic involving a **specific host** — useful when troubleshooting one particular machine's connectivity.

```bash
sudo tcpdump -i eth0 -n
```
`-n` disables **DNS/name resolution** for addresses in the output — keeps output fast and avoids generating extra DNS lookups of its own while capturing.

```bash
sudo tcpdump -i eth0 -v
```
`-v` (verbose) shows **more protocol detail** per packet — additional header fields beyond the default summary line.

```bash
sudo tcpdump -i eth0 -c 100
```
`-c` **limits capture to a specific number of packets** (100 here) — useful for a quick, bounded sample instead of an open-ended capture.

```bash
sudo tcpdump -i eth0 'tcp[tcpflags] & (tcp-syn) != 0'
```
Uses a more advanced **BPF filter expression** to capture only packets with the **SYN flag** set — isolating new connection attempts specifically, useful for troubleshooting connection establishment issues.

---

## 5. Tool: Wireshark

**Wireshark** is the industry-standard **graphical** packet analysis tool — where Tcpdump excels at fast, lightweight command-line capture, Wireshark excels at deep, visual, interactive analysis of that same captured data.

### 5.1 How It Works
- Like Tcpdump, Wireshark uses libpcap to capture live traffic (or open `.pcap`/`.pcapng` files captured by Tcpdump, Kismet, or hcxdumptool) — meaning the two tools are fully interoperable.
- Every captured packet is automatically **decoded and color-coded** by protocol, and displayed in three linked panes: a packet list, a detailed protocol tree for the selected packet, and the raw hex/ASCII bytes — directly implementing Section 2's "protocol analysis" concept visually.
- Its **"Follow Stream"** feature reconstructs an entire conversation (e.g., a full TCP session or HTTP exchange) from the individual packets that made it up, displaying it as a single readable transcript.
- A powerful **display filter** language (different from Tcpdump's capture-time BPF syntax) lets you filter/search *already-captured* traffic interactively, without needing to re-capture.

### 5.2 What It's Used For
- **Deep-dive analysis** of captured traffic — the standard tool reached for once Tcpdump/hcxdumptool-style capture has produced a file worth investigating closely.
- Visually reconstructing full conversations (HTTP requests, DNS lookups, TCP sessions) to understand exactly what happened, in order.
- Both security investigation (spotting suspicious traffic, extracting plaintext credentials from insecure protocols) and troubleshooting (diagnosing exactly where a connection or protocol exchange broke down).

### 5.3 Important Commands & Workflow

```bash
wireshark
```
Launches the Wireshark GUI — from here, most work happens through the interface rather than further terminal commands.

```
Capture → choose an interface → click the blue shark-fin "Start" icon
```
Begins a **live capture** directly within Wireshark, equivalent to running `tcpdump -i <interface>` but with the GUI's live decoding and display.

```
File → Open → select a .pcap/.pcapng file
```
Opens a **previously captured file** (from Tcpdump, Kismet, hcxdumptool, etc.) for offline analysis — the other half of the "capture with Tcpdump, analyze with Wireshark" workflow.

```
Display filter bar: http
```
Filters the currently loaded/captured traffic to show only **HTTP** packets — Wireshark's display filters use protocol names directly, unlike Tcpdump's BPF syntax.

```
Display filter bar: ip.addr == 192.168.1.10
```
Filters to only traffic involving a **specific IP address** — the Wireshark GUI equivalent of Tcpdump's `host` filter.

```
Display filter bar: tcp.port == 443
```
Filters to a **specific port** — same purpose as Tcpdump's `port` filter, but applied interactively after capture.

```
Right-click a packet → Follow → TCP Stream
```
Reconstructs the **entire conversation** that packet belongs to into one readable transcript — Wireshark's signature feature for understanding a full exchange at once.

```
Statistics → Protocol Hierarchy
```
Shows a **breakdown of all traffic by protocol** in the current capture — a fast way to get an overview of what kinds of traffic are present before diving into specifics.

```
Statistics → Conversations
```
Lists every distinct **conversation** (pair of endpoints) in the capture, along with packet/byte counts — useful for quickly spotting which hosts are talking the most, a common troubleshooting/investigation starting point.

```
File → Export Objects → HTTP...
```
Extracts any **files transferred over HTTP** within the capture (images, documents, downloads) directly to disk — useful evidence-gathering during an investigation.

---

## 6. Tool: Ettercap

**Ettercap** is a mature, dedicated **man-in-the-middle (MITM) framework** for wired and wireless LANs — where Bettercap (Day 18) is the modern, modular successor, Ettercap remains widely used for its mature, well-tested ARP spoofing and traffic manipulation capabilities.

### 6.1 How It Works
- Ettercap performs **ARP spoofing** (the same core technique described in Day 18, Section 1.2) to position itself between two hosts on a LAN — typically a victim machine and the network gateway — causing their traffic to route through the attacker's machine first.
- Once positioned, it can **sniff** the intercepted traffic, and its **plugin/filter system** allows active manipulation — modifying content in transit (e.g., injecting content into unencrypted HTTP responses) rather than just passively observing.
- It supports both a **text-based CLI mode** for quick/scripted use and a **GUI mode** for interactive target selection and monitoring.

### 6.2 What It's Used For
- Demonstrating and testing **MITM exposure** on a LAN — the same underlying purpose as Bettercap's ARP spoofing from Day 18, giving you a second, more traditional tool for the same assessment task.
- Capturing **unencrypted credentials** (Telnet, FTP, old HTTP logins) as direct, concrete proof that a network segment lacks encryption where it should have it.
- Testing whether a network has ARP spoofing detection/protection in place (e.g., dynamic ARP inspection on managed switches).

### 6.3 Important Commands

```bash
sudo ettercap -G
```
Launches Ettercap's **graphical interface** — the most common way to use it interactively: scan for hosts, select targets, and start sniffing through menus.

```bash
sudo ettercap -T -i eth0
```
Launches **text-mode** (`-T`) on a specified interface (`-i`) — useful for a lighter-weight, terminal-only session or scripted use.

```bash
ettercap -T -M arp:remote /192.168.1.1// /192.168.1.50//
```
Runs a **text-mode ARP MITM attack** (`-M arp:remote`) between two specific hosts — here, the gateway (`192.168.1.1`) and a victim (`192.168.1.50`) — the direct command-line equivalent of Bettercap's `arp.spoof` from Day 18.

```bash
sudo ettercap -T -q -M arp:remote /192.168.1.1// /192.168.1.50//
```
`-q` runs in **quiet mode**, suppressing packet content output — useful when you only want to position the MITM attack without flooding the terminal with every packet's content.

```bash
sudo ettercap -T -M arp:remote -P dns_spoof /192.168.1.1// /192.168.1.50//
```
`-P` loads a specific **plugin** during the attack — here, `dns_spoof`, redirecting the victim's DNS lookups to attacker-controlled answers, on top of the base ARP spoofing.

```
# Inside the GUI:
Sniff → Unified sniffing → select interface
Hosts → Scan for hosts
Hosts → Host list → select two targets → add as Target 1 / Target 2
Mitm → ARP poisoning → OK
```
The GUI-mode workflow equivalent of the CLI ARP-spoof command above — scan the LAN, pick your two targets, and start the MITM attack through the menu system.

---

## 7. Tool: Netcat

**Netcat** was introduced in Day 2 as a lightweight TCP/UDP utility for basic connectivity checks. In the context of Day 19, it earns a second look as a genuinely useful **troubleshooting and traffic-generation** tool — not for capturing traffic, but for creating traffic worth capturing and analyzing.

### 7.1 How It Works (Recap + New Angle)
- Netcat can act as either a **client** (connecting out to a host/port) or a **listener/server** (`-l`, waiting for an incoming connection) over TCP or UDP.
- In a troubleshooting context, this makes it perfect for **generating a controlled, predictable piece of test traffic** — you know exactly what you sent, so any capture of that traffic in Wireshark/Tcpdump becomes a clean, easy-to-interpret example, rather than trying to make sense of unpredictable real application traffic.

### 7.2 What It's Used For (In This Day's Context)
- Generating simple, known test traffic to **verify a capture setup is actually working** before trying to capture something more complex (e.g., "send a Netcat packet, confirm Tcpdump sees it, *then* trust the rest of the capture").
- Quickly testing whether a firewall rule or network path allows a specific port through, while simultaneously capturing the attempt to see exactly what happens at the packet level.

### 7.3 Important Commands

```bash
nc -l -p 4444
```
Starts Netcat **listening** on port 4444 — combined with a capture running on the same machine, this creates a clean, predictable target to verify your capture setup is working correctly.

```bash
nc 192.168.1.10 4444
```
Connects **out** to a listener on another host — anything typed is sent as raw TCP data, ideal as a simple, observable test payload for a capture.

```bash
echo "test123" | nc 192.168.1.10 4444
```
Sends a single, **known test string** to a listener — capture this with Tcpdump/Wireshark and you have a guaranteed, easy-to-find reference packet in your capture for verification purposes.

```bash
nc -zv 192.168.1.10 22
```
The connectivity-check command from Day 2, revisited here for **troubleshooting**: confirms whether a port is reachable, while a simultaneous capture shows exactly what the connection attempt (or failure) looks like at the packet level.

```bash
nc -u -l -p 5000
```
`-u` switches to **UDP mode** while listening — useful for testing/troubleshooting UDP-based services and observing their distinct (connectionless) packet pattern compared to TCP.

---

## 8. Tool: Socat

**Socat** ("SOcket CAT") is a far more powerful and flexible relative of Netcat — where Netcat connects two simple endpoints, Socat can connect **almost any two types of data channels** to each other (TCP, UDP, files, serial ports, SSL, pipes), making it a favorite for advanced network troubleshooting and traffic redirection.

### 8.1 How It Works
- Socat's core concept is connecting **two "addresses"** together, where an address can be a huge range of things: a TCP/UDP socket, a file, standard input/output, a Unix socket, an SSL connection, or even another program's input/output.
- This makes it capable of things Netcat simply can't do — like transparently **relaying/proxying** traffic from one port to another, or wrapping a plain connection in SSL/TLS, or bridging a network service to a serial device.
- In a troubleshooting context, Socat is often used to build a **traffic relay** — sitting between a client and server so you can observe (and optionally log) traffic passing through, without the client or server needing any reconfiguration beyond pointing at the relay.

### 8.2 What It's Used For
- **Port forwarding/relaying** — redirecting traffic from one port/host to another, useful for testing or working around network restrictions during troubleshooting.
- **Protocol/encryption wrapping** — e.g., adding SSL to a service that doesn't natively support it, for testing purposes.
- Building simple relays to **sit in the middle of a connection** for observation, complementing full MITM tools like Ettercap/Bettercap with a much lighter-weight, targeted approach for a single service.

### 8.3 Important Commands

```bash
socat TCP-LISTEN:8080,fork TCP:192.168.1.10:80
```
The core relay pattern — listens on local port 8080 and **forwards** every connection to `192.168.1.10:80`; `fork` allows handling multiple simultaneous connections. Useful for redirecting traffic through a point where you can also run a capture.

```bash
socat - TCP:192.168.1.10:80
```
Connects your terminal (`-` means stdin/stdout) directly to a **TCP service**, similar to Netcat's basic client mode, letting you manually send/receive raw data.

```bash
socat TCP-LISTEN:443,fork,reuseaddr OPENSSL:backend.local:443,verify=0
```
Relays traffic while wrapping the backend connection in **SSL/TLS** (`OPENSSL:`) — useful for testing scenarios involving encrypted backend services.

```bash
socat -v TCP-LISTEN:8080,fork TCP:192.168.1.10:80
```
`-v` enables **verbose mode**, printing the data passing through the relay to the terminal — turning Socat itself into a simple, live traffic-inspection tool for whatever it's relaying.

```bash
socat UDP-LISTEN:5000,fork UDP:192.168.1.10:5000
```
The same relay pattern applied to **UDP** traffic instead of TCP.

```bash
socat TCP-LISTEN:2222,fork EXEC:/bin/bash
```
`EXEC:` connects the socket to a **running program's input/output** instead of another network address — a powerful, advanced capability with no direct Netcat equivalent (useful to know exists, though such setups need careful access control).

---

## 9. Tool: Driftnet

**Driftnet** is a narrowly-focused traffic analysis tool that watches network traffic and **extracts images** from any unencrypted traffic it observes — a very concrete, visual demonstration of what "protocol analysis" (Section 2) can reveal when a protocol carries plaintext content.

### 9.1 How It Works
- Driftnet listens on a specified interface (often combined with Ettercap/Bettercap's traffic-redirection so it can see other hosts' traffic, not just its own) in promiscuous mode.
- It reconstructs **image files** (JPEG, GIF, PNG, and similar) it identifies inside unencrypted network streams — most commonly plain HTTP traffic — and displays them in a live pop-up window as they're found.
- Because HTTPS-encrypted traffic can't be read this way, Driftnet only ever reveals images that are being transferred **without encryption** — making it, in practice, a very direct and memorable demonstration of exactly what plaintext HTTP exposes to anyone who can see the traffic.

### 9.2 What It's Used For
- **Demonstrating the real-world impact of unencrypted traffic** in a training/awareness setting — few things make "always use HTTPS" more convincing than watching someone's plaintext images pop up on screen in real time.
- Quick, visual confirmation during an assessment that a given network segment/device is transmitting genuinely unencrypted image traffic.

### 9.3 Important Commands

```bash
sudo driftnet -i eth0
```
The core command — watches interface `eth0` and pops up any images found in unencrypted traffic as they're captured.

```bash
sudo driftnet -i eth0 -v
```
`-v` (verbose) prints additional information about what's being captured/found to the terminal alongside the image pop-ups.

```bash
sudo driftnet -i eth0 -d /tmp/driftnet_images
```
`-d` saves extracted images to a specified **directory** on disk instead of only showing them in a temporary pop-up window — useful for keeping evidence for a report.

```bash
sudo driftnet -i eth0 -p
```
`-p` runs in **passive mode**, meaning it does not itself set the interface to promiscuous mode — useful when another tool (e.g., Ettercap already running) has already positioned/configured capture, and you don't want Driftnet to alter interface settings again.

```bash
sudo driftnet -i eth0 -m 100
```
`-m` limits the maximum number of images **kept in memory/temp storage** at once, preventing runaway resource usage during a long capture session.

---

## 10. Quick Cheat Sheet

| Task | Command |
|---|---|
| Quick live capture (CLI) | `sudo tcpdump -i eth0` |
| Save capture to file | `sudo tcpdump -i eth0 -w capture.pcap` |
| Deep visual analysis | `wireshark` → open the `.pcap` file |
| Reconstruct a full conversation | Wireshark: right-click → Follow → TCP Stream |
| ARP-spoof MITM (CLI) | `sudo ettercap -T -M arp:remote /gateway// /victim//` |
| Generate known test traffic | `echo "test123" \| nc 192.168.1.10 4444` |
| Relay/forward traffic for inspection | `socat -v TCP-LISTEN:8080,fork TCP:target:80` |
| Extract images from unencrypted traffic | `sudo driftnet -i eth0` |

**Typical Day 19 Analysis & Troubleshooting Workflow (lab/authorized environment):**
```bash
sudo tcpdump -i eth0 -w capture.pcap                         # 1. Start a baseline capture
echo "test123" | nc 192.168.1.10 4444                            # 2. Generate known test traffic to verify capture is working
wireshark capture.pcap                                              # 3. Open in Wireshark for deep analysis
# ... use display filters, Follow TCP Stream, Protocol Hierarchy ...
sudo ettercap -T -M arp:remote /gateway// /victim//                    # 4. (Lab only) Position MITM to see a victim's traffic
sudo driftnet -i eth0                                                     # 5. Visually confirm unencrypted image traffic exposure
socat -v TCP-LISTEN:8080,fork TCP:webserver:80                              # 6. Relay + inspect a specific service's traffic directly
```

---

### Practice Suggestions for Day 19
1. Run `tcpdump -i eth0 -w capture.pcap` while browsing a lab website, then open the file in Wireshark and use "Follow → TCP Stream" to reconstruct one full HTTP request/response.
2. Use Netcat to send a known test string between two lab machines while Tcpdump is capturing, and confirm you can find that exact string in the resulting `.pcap`.
3. Compare a capture of plain HTTP traffic against HTTPS traffic in Wireshark — note what remains visible (IP, port, TLS handshake/certificate) versus what's hidden.
4. In an isolated lab, run Ettercap's ARP-spoof MITM between two lab machines, then run Driftnet on the attacker machine while the "victim" browses a plain-HTTP test image gallery.
5. Use Socat to build a simple TCP relay between two lab services, run it with `-v`, and compare its inspection output against a Wireshark capture of the same traffic.
6. Use Wireshark's Statistics → Conversations and Protocol Hierarchy views on a capture from a busy lab network, and write a short summary of what kinds of traffic dominate.
7. Deliberately break something in a lab (e.g., block a port with a firewall rule) and use a fresh Tcpdump/Wireshark capture to diagnose exactly where the connection attempt fails — practicing Section 3's troubleshooting approach.
