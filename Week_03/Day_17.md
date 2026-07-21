# Day 17 — Wireless Security Fundamentals


**Tools**
● Aircrack-ng ● Wifite ● Kismet

**Topics**
● Wireless Architecture ● WPA/WPA2 Concepts ● Wireless Monitoring

A complete reference covering wireless architecture, WPA/WPA2 concepts, and wireless monitoring — using Aircrack-ng, Wifite, and Kismet. (Topics are explained first, followed by the tools — each explained by how it works and what it's used for, then its most important commands.)

> ⚠️ **Ethical Note:** Wireless assessment tools interact with radio spectrum shared by everyone nearby, not just your target — capturing traffic or testing a network you don't own/administer, or interfering with networks around you, can be illegal even in a "just looking" capacity in many jurisdictions. Only test networks you own or have explicit written authorization to assess, ideally using a dedicated, isolated lab access point.

---

## Table of Contents
1. [Wireless Architecture](#1-wireless-architecture)
2. [WPA/WPA2 Concepts](#2-wpawpa2-concepts)
3. [Wireless Monitoring](#3-wireless-monitoring)
4. [Tool: Kismet](#4-tool-kismet)
5. [Tool: Aircrack-ng](#5-tool-aircrack-ng)
6. [Tool: Wifite](#6-tool-wifite)
7. [Quick Cheat Sheet](#7-quick-cheat-sheet)

---

## 1. Wireless Architecture

**Wireless architecture** describes how Wi-Fi networks are physically and logically structured — the roles, terminology, and communication patterns you need to understand before any wireless security concept makes sense.

### 1.1 Core Components

| Term | Meaning |
|---|---|
| **AP (Access Point)** | The device broadcasting the Wi-Fi network — the router/hardware clients connect to |
| **SSID** | The network's human-readable name (e.g., "HomeWiFi") |
| **BSSID** | The AP's unique MAC address — the actual, unspoofable-by-name identifier of a specific radio broadcasting a network |
| **Client/Station** | Any device connecting to the AP (laptop, phone, IoT device) |
| **Channel** | The specific radio frequency the AP operates on (e.g., channel 6 on 2.4GHz) — multiple nearby APs share the limited channel space |
| **BSS (Basic Service Set)** | One AP and all the clients associated with it |
| **ESS (Extended Service Set)** | Multiple APs sharing the same SSID, working together to extend coverage (common in offices/campuses) |

### 1.2 How a Client Connects (Simplified)
```
1. AP broadcasts "Beacon frames" advertising its SSID/BSSID/capabilities
2. Client sends a Probe Request (or listens for beacons) to find nearby networks
3. Client sends an Authentication Request, then an Association Request
4. AP accepts, and the client becomes associated with the BSS
5. (If secured) A key exchange happens to establish encrypted communication — see Section 2
```

### 1.3 Why It Matters
- Every wireless attack technique targets a **specific point** in this architecture — deauthentication attacks exploit the management-frame handshake, rogue APs exploit SSID trust, and handshake capture (Section 2) exploits the key-exchange step.
- Understanding BSSID vs. SSID is critical: an attacker can trivially clone an SSID (the *name*), but the BSSID (tied to the physical radio's MAC) is what tools actually use to distinguish the real AP from an impostor.

---

## 2. WPA/WPA2 Concepts

**WPA (Wi-Fi Protected Access)** and its successor **WPA2** are the security protocols that encrypt wireless traffic and authenticate clients — understanding how they establish a secure connection is essential to understanding both their strength and their well-known weak point.

### 2.1 The Evolution (Briefly)
| Protocol | Status |
|---|---|
| WEP | Broken/deprecated — trivially crackable regardless of password strength, due to fundamental cryptographic flaws |
| WPA | Improved over WEP but has its own known weaknesses (TKIP) |
| WPA2 | Current widespread standard — uses AES-CCMP encryption, strong when configured correctly |
| WPA3 | Newest standard — replaces the vulnerable handshake step described below with a stronger method (SAE) |

This lab focuses on **WPA/WPA2-Personal (PSK)** — the pre-shared-key mode used in most home/small-business networks — since it's what Aircrack-ng and Wifite are built around.

### 2.2 The 4-Way Handshake
When a client connects to a WPA/WPA2-PSK network, the AP and client perform a **4-way handshake** to prove both sides know the network password (without ever sending the password itself over the air) and to derive session encryption keys:
```
AP    → Client : Message 1 (ANonce - a random value from the AP)
Client → AP    : Message 2 (SNonce - a random value from the client + MIC)
AP    → Client : Message 3 (Group key + MIC, confirming the exchange)
Client → AP    : Message 4 (Acknowledgment)
```
Both sides combine the network's PSK (password) with the exchanged nonces (and each device's MAC address) to independently derive the same **Pairwise Transient Key (PTK)** — used to encrypt actual traffic from that point on.

### 2.3 Why This Handshake Is the Attack Target
This is the single most important concept in WPA/WPA2-Personal security assessment:
- The handshake itself doesn't transmit the password directly — but it **does** transmit enough cryptographic material that, once captured, an attacker can test password guesses **offline** (directly connecting to Day 15's offline cracking concept): for each candidate password, derive what the PTK/MIC *would* be, and check if it matches what was actually captured.
- This means: **capture the handshake once, then crack entirely offline**, at whatever speed your hardware allows — no further interaction with the AP needed.
- The practical implication: **WPA2-Personal's real-world security is only as strong as the password chosen** — a weak, dictionary-guessable password can be cracked from a captured handshake regardless of how strong the underlying AES encryption is.

### 2.4 Why It Matters
- Explains exactly *why* wireless assessments focus so heavily on capturing this one specific handshake exchange (Section 3/Aircrack-ng below), rather than trying to break the encryption algorithm itself (which remains computationally infeasible).
- Directly justifies strong, random Wi-Fi passwords as a security control — ties back into Day 15/16's password strength concepts, just applied to a different protocol.

---

## 3. Wireless Monitoring

**Wireless monitoring** is the process of observing all Wi-Fi activity in radio range — every AP, every client, every frame passing through the air — regardless of which network you're actually connected to.

### 3.1 Monitor Mode vs. Normal Mode
A standard Wi-Fi adapter, in its default **managed mode**, only processes traffic for the network it's actively connected to, and only shows you application-level data — it hides the underlying 802.11 frames entirely.

**Monitor mode** puts the wireless adapter into a special state where it captures **raw 802.11 frames from every network in range**, not just one it's associated with — including management frames (beacons, handshakes) that are invisible in normal mode. This requires a wireless adapter/driver that specifically supports monitor mode — not all hardware does.

### 3.2 What Gets Observed
- **Beacon frames** — revealing every nearby AP's SSID, BSSID, channel, and security type.
- **Probe requests** — revealing which SSIDs *client devices* are actively searching for (sometimes leaking networks a device has previously connected to, like a home network name, even while it's out in public).
- **Data frames** — the actual (usually encrypted) traffic passing between AP and clients.
- **The 4-way handshake frames** — the specific exchange described in Section 2, which is what gets captured for later offline cracking.

### 3.3 Why It Matters
- Monitoring is the **prerequisite step** for almost every other wireless technique — you can't target a specific handshake, deauth a specific client, or assess a specific AP's security without first observing it.
- It also has a purely defensive use — an organization can monitor its own airspace to detect **rogue access points** (unauthorized APs someone has plugged in) or unexpected/unauthorized client devices.

---

## 4. Tool: Kismet

**Kismet** is a passive wireless network detector, sniffer, and intrusion-detection framework — it's the **monitoring specialist** of this trio, focused on comprehensively observing and cataloging wireless activity rather than attacking it.

### 4.1 How It Works
- Kismet puts a supported wireless adapter into **monitor mode** and passively listens across Wi-Fi channels (hopping between them to cover the full spectrum), building a live, continuously updated database of every AP and client it observes.
- Because it's purely **passive** by design (it doesn't need to send any packets to detect a network), it can be run indefinitely with essentially no footprint of its own — it never has to associate with or send traffic to any network to build its picture.
- It runs as a background **server process** with a **web-based UI** for viewing results in real time, and can also log everything to disk (in `.kismet`/pcap-compatible formats) for later analysis.
- Beyond Wi-Fi, Kismet can also be extended to monitor other wireless protocols (Bluetooth, some SDR-based signals) depending on configuration/plugins.

### 4.2 What It's Used For
- Building a **complete situational picture** of the wireless environment before deciding what (if anything) to target further — the natural first step in any wireless engagement.
- **Rogue AP / unauthorized device detection** — a defensive use case, continuously watching an organization's own airspace for anything that shouldn't be there.
- Passive, low-footprint reconnaissance when you specifically want to avoid sending any packets at all.

### 4.3 Important Commands

```bash
sudo airmon-ng start wlan0
```
Puts the wireless adapter into **monitor mode** (usually creating `wlan0mon`) — a prerequisite step shared by all three tools in this document, so it's introduced here first.

```bash
sudo kismet
```
Launches the **Kismet server** — starts capturing on all detected compatible wireless interfaces and serves its web UI (by default at `http://localhost:2501`).

```bash
sudo kismet -c wlan0mon
```
`-c` explicitly specifies **which interface** to capture on, rather than relying on Kismet's auto-detection — useful when multiple wireless adapters are present.

```bash
kismet_client
```
Launches Kismet's **text-based client UI** — an alternative to the web interface for viewing live results directly in the terminal.

```bash
# Inside the Kismet web UI:
# View → Channels
```
Shows real-time **channel activity/utilization** — useful for understanding which channels are busiest in your environment.

```bash
# Inside the Kismet web UI:
# Devices tab → filter by "Access Points"
```
Lists every discovered **AP**, along with SSID, BSSID, channel, encryption type, and signal strength — the core "what's out there" view.

```bash
sudo kismet --no-ncurses -c wlan0mon -t my_capture
```
`-t` sets a custom **log file name/title** for the capture session, and disables the ncurses console output — useful for scripted/background logging runs.

```bash
# Kismet log files (.kismet / pcapng) can be opened later in Wireshark (Day 19)
```
Kismet's captured data is compatible with Wireshark's deeper packet-analysis workflow, which you'll build on later in the course.

---

## 5. Tool: Aircrack-ng

**Aircrack-ng** is actually a **suite** of tools (not a single program) built specifically around the WPA/WPA2 handshake-capture-and-crack workflow described in Section 2 — it's the toolkit that turns wireless monitoring into an actual security assessment result.

### 5.1 How It Works
The suite's components map directly onto the phases of a wireless assessment:

| Component | Role |
|---|---|
| `airmon-ng` | Enables/manages **monitor mode** on a wireless adapter |
| `airodump-ng` | **Captures** raw 802.11 traffic — the monitoring workhorse, used to find target APs and capture the handshake |
| `aireplay-ng` | **Injects/replays** packets — most notably, sends deauthentication frames to force a client to reconnect (generating a fresh handshake to capture) |
| `aircrack-ng` | **Cracks** a captured handshake against a wordlist — the offline password-recovery step, directly applying Day 15's cracking concepts to a `.cap` file instead of a hash file |

The typical flow: put the adapter in monitor mode → use `airodump-ng` to find and focus on a target AP → optionally use `aireplay-ng` to speed up handshake capture by forcing a reconnect → run `aircrack-ng` against the resulting capture file with a wordlist.

### 5.2 What It's Used For
- The **standard, foundational toolkit** for assessing WPA/WPA2-Personal network password strength.
- Manually, step-by-step wireless assessments where you want full control over each phase (as opposed to Wifite's automated approach, below).
- Teaching/demonstrating exactly how the handshake-capture-and-crack process works, since each step is a separate, visible command.

### 5.3 Important Commands

```bash
sudo airmon-ng start wlan0
```
Enables **monitor mode**, same first step as Kismet — creates a monitor-mode interface (e.g., `wlan0mon`).

```bash
sudo airodump-ng wlan0mon
```
Starts **scanning** across all channels, listing every visible AP (BSSID, channel, encryption, SSID) and connected clients — the reconnaissance step to identify your target.

```bash
sudo airodump-ng -c 6 --bssid AA:BB:CC:DD:EE:FF -w capture wlan0mon
```
Focuses capture on **one specific AP** — `-c` locks to its channel, `--bssid` filters to just that network, and `-w` saves captured packets to a file — this is where you wait to capture the 4-way handshake.

```bash
sudo aireplay-ng --deauth 5 -a AA:BB:CC:DD:EE:FF wlan0mon
```
Sends **deauthentication frames** (5 of them) to the target AP, forcing connected clients to disconnect and automatically reconnect — which regenerates the 4-way handshake, letting you capture it faster instead of waiting for a natural reconnect.

```bash
aircrack-ng capture-01.cap -w rockyou.txt
```
Runs the actual **offline cracking** step — tests each password in the wordlist against the captured handshake in `capture-01.cap`, directly applying the logic from Section 2.3.

```bash
aircrack-ng capture-01.cap
```
Run without a wordlist first — useful just to **verify a valid handshake was actually captured** in the file before committing to a long cracking run.

```bash
sudo airodump-ng --band abg wlan0mon
```
`--band` scans across multiple frequency bands (a = 5GHz, b/g = 2.4GHz) — useful for making sure you're not missing networks operating on 5GHz-only.

```bash
sudo airmon-ng stop wlan0mon
```
Disables monitor mode, returning the adapter to normal managed mode — good practice to run once your assessment session is complete.

---

## 6. Tool: Wifite

**Wifite** is an **automation wrapper** around Aircrack-ng (and several other wireless tools) — it strings together the entire monitor → scan → target → capture → crack workflow into one guided, largely hands-off process.

### 6.1 How It Works
- Wifite automatically enables monitor mode, scans for nearby networks, and presents a simple list for you to choose a target (or targets) from.
- Once a target is selected, it automatically runs the appropriate attack strategy based on the network's detected security type — for WPA/WPA2, this typically means automating the deauth-and-capture sequence from Aircrack-ng's `aireplay-ng`/`airodump-ng` steps.
- After capturing a handshake, it can automatically attempt to **crack it** using a supplied wordlist, without you needing to manually invoke `aircrack-ng` yourself.
- It's designed to try **multiple attack techniques in sequence** against a target automatically (e.g., falling back to a different method if the first doesn't succeed), rather than requiring you to choose and run each one manually.

### 6.2 What It's Used For
- **Fast, automated wireless assessments** — especially useful when auditing many networks in a lab session, since it removes the manual, multi-step process Aircrack-ng requires.
- A good tool for **beginners** to see the full attack chain work end-to-end quickly, before learning to run each Aircrack-ng component manually for finer control.
- Batch-testing multiple APs in one session without re-typing each Aircrack-ng command by hand.

### 6.3 Important Commands

```bash
sudo wifite
```
The core command — launches Wifite, which automatically enables monitor mode, scans for nearby networks, and presents an interactive target list. This is how you'll start almost every Wifite session.

```bash
sudo wifite --dict rockyou.txt
```
`--dict` specifies the **wordlist** Wifite should use for any automatic cracking attempts against captured handshakes.

```bash
sudo wifite -i wlan0
```
`-i` explicitly specifies which **interface** to use — useful when multiple wireless adapters are connected.

```bash
sudo wifite --kill
```
Kills processes that might interfere with monitor mode (like NetworkManager or `wpa_supplicant`) before starting — Wifite's equivalent of `airmon-ng check kill`.

```bash
sudo wifite -c 6
```
`-c` restricts scanning to a **specific channel**, useful when you already know exactly where your target AP is.

```bash
sudo wifite --wpa
```
Restricts Wifite's scan/attack to **WPA/WPA2 networks only**, skipping other encryption types it might otherwise also try to attack.

```bash
sudo wifite --wps
```
Focuses specifically on networks with **WPS (Wi-Fi Protected Setup)** enabled, using WPS-specific attack techniques — a different, separate weakness from the WPA handshake approach (WPS gets covered in more depth on Day 18 with Reaver/Bully).

```bash
sudo wifite --timeout 300
```
`--timeout` sets a maximum time (in seconds) Wifite will spend attempting to capture a handshake from a given target before moving on — keeps automated sessions from stalling indefinitely on a difficult target.

---

## 7. Quick Cheat Sheet

| Task | Command |
|---|---|
| Enable monitor mode | `sudo airmon-ng start wlan0` |
| Passive full-environment monitoring | `sudo kismet -c wlan0mon` |
| Scan for nearby APs | `sudo airodump-ng wlan0mon` |
| Capture handshake from one AP | `sudo airodump-ng -c <ch> --bssid <BSSID> -w capture wlan0mon` |
| Force reconnect (deauth) to speed up capture | `sudo aireplay-ng --deauth 5 -a <BSSID> wlan0mon` |
| Crack a captured handshake | `aircrack-ng capture-01.cap -w rockyou.txt` |
| Fully automated end-to-end attack | `sudo wifite --dict rockyou.txt` |

**Typical Day 17 Wireless Assessment Workflow (lab, isolated test AP):**
```bash
sudo airmon-ng check kill                                   # 1. Stop interfering processes
sudo airmon-ng start wlan0                                      # 2. Enable monitor mode
sudo kismet -c wlan0mon                                            # 3. Passive situational awareness first
sudo airodump-ng wlan0mon                                             # 4. Identify the specific target AP/channel
sudo airodump-ng -c 6 --bssid AA:BB:CC:DD:EE:FF -w capture wlan0mon      # 5. Focus capture on target
sudo aireplay-ng --deauth 5 -a AA:BB:CC:DD:EE:FF wlan0mon                   # 6. Force a fresh handshake
aircrack-ng capture-01.cap -w rockyou.txt                                     # 7. Offline crack the captured handshake
# --- OR, for the automated equivalent of steps 4-7: ---
sudo wifite --dict rockyou.txt
```

---

### Practice Suggestions for Day 17
1. Enable monitor mode on a supported adapter and run Kismet against your own home network for a few minutes — review the Devices tab and identify your own AP and connected devices.
2. Run `airodump-ng` and identify your lab AP's BSSID, channel, and encryption type from the live scan output.
3. Capture a 4-way handshake from your own isolated test AP using `airodump-ng`, using `aireplay-ng --deauth` to speed up the process, and verify the capture with `aircrack-ng capture.cap` (no wordlist).
4. Crack the captured handshake with `aircrack-ng` and a small test wordlist that includes the known (test) password, to see the full pipeline succeed end-to-end.
5. Repeat the same target with Wifite's fully automated flow and compare how much manual work it saved versus the step-by-step Aircrack-ng approach.
6. Write a short note comparing Kismet (passive monitoring) against Aircrack-ng/Wifite (active capture) — when would you use one over the other in a real engagement?
