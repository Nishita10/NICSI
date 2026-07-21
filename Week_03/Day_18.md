# Day 18 — Wireless Security Assessment


**Tools**
● Bettercap ● Reaver ● Bully ● Airgeddon ● hcxdumptool

**Topics**
● Wireless Security Concepts ● Rogue AP Detection ● Wireless Assessment (Lab)

A complete reference covering broader wireless security concepts, rogue AP detection, and structured lab-based wireless assessment — using Bettercap, Reaver, Bully, Airgeddon, and hcxdumptool. (Topics are explained first, followed by the tools — each explained by how it works and what it's used for, then its most important commands.)

> ⚠️ **Ethical Note:** Every tool in this document is more intrusive than Day 17's — WPS attacks can lock routers out, deauth attacks disrupt real users' connectivity, and MITM techniques intercept live traffic. These must only be run in an isolated lab against equipment you own, or with explicit written authorization. Several techniques here (deauthentication, rogue APs) can also violate radio-spectrum regulations even on "your own" network if it interferes with neighboring networks — always test in a shielded/controlled lab environment where possible.

---

## Table of Contents
1. [Wireless Security Concepts](#1-wireless-security-concepts)
2. [Rogue AP Detection](#2-rogue-ap-detection)
3. [Wireless Assessment (Lab)](#3-wireless-assessment-lab)
4. [Tool: hcxdumptool](#4-tool-hcxdumptool)
5. [Tool: Reaver](#5-tool-reaver)
6. [Tool: Bully](#6-tool-bully)
7. [Tool: Bettercap](#7-tool-bettercap)
8. [Tool: Airgeddon](#8-tool-airgeddon)
9. [Quick Cheat Sheet](#9-quick-cheat-sheet)

---

## 1. Wireless Security Concepts

Day 17 focused on one specific weak point — the WPA/WPA2 4-way handshake. Real wireless environments have several **other** distinct weaknesses worth understanding, each exploited by a different tool in this document.

### 1.1 WPS (Wi-Fi Protected Setup) — A Separate, Serious Weakness
WPS is a convenience feature letting users connect to a router using an **8-digit PIN** instead of typing the full Wi-Fi password — often enabled by default on consumer routers, and largely independent of how strong the actual WPA2 password is.

**Why it's a serious weakness:** the 8-digit PIN is validated by the router in **two separate halves** (the first 4 digits, then the last 3 — the 8th digit is just a checksum). This dramatically reduces the real search space compared to a true 8-digit brute-force, since an attacker only needs to correctly guess two much smaller chunks rather than the full combination at once. A vulnerable, unprotected implementation can potentially be brute-forced in hours, entirely independent of password strength — this is exactly what Reaver and Bully (Sections 5–6) target.

### 1.2 Man-in-the-Middle (MITM) on Wireless Networks
Once an attacker is **on** a wireless network (whether through a cracked password, a WPS PIN, or by tricking a user onto a fake network), a whole separate category of attack becomes possible: intercepting traffic **between** other devices and their destination.
- **ARP spoofing** — tricking devices on the local network into sending their traffic through the attacker's machine instead of the real gateway.
- **DNS spoofing** — returning fake DNS answers to redirect a victim to an attacker-controlled site.
- **Traffic sniffing/manipulation** — reading or altering unencrypted traffic passing through the intercepted connection.

This is the domain Bettercap (Section 7) operates in — it's less about "getting onto" the network and more about what becomes possible **once you're already on it**.

### 1.3 Why It Matters
- Password strength (Day 17) is only one layer — a strong WPA2 password provides **no protection at all** if WPS is left enabled and vulnerable, illustrating why a full assessment checks multiple, independent attack surfaces rather than just one.
- Understanding MITM concepts explains *why* being on an open/compromised Wi-Fi network is risky even beyond "someone might see my Wi-Fi password" — it's a foothold for intercepting everything else you do on that network.

---

## 2. Rogue AP Detection

A **rogue access point** is any Wi-Fi access point operating in or near an organization's environment that isn't authorized or sanctioned — ranging from an employee's innocently-plugged-in personal router, to a deliberately malicious **evil twin** designed to trick users into connecting.

### 2.1 Types of Rogue APs

| Type | Description |
|---|---|
| **Unauthorized/unmanaged AP** | An employee plugs in their own router/hotspot for convenience, unknowingly creating an unmanaged backdoor into the network |
| **Evil Twin** | An attacker sets up an AP broadcasting the **same SSID** as a legitimate network, hoping victims connect to the fake one instead |
| **Honeypot AP** | A deliberately open, tempting-looking network (e.g., "Free_Airport_WiFi") set up purely to capture traffic/credentials from anyone who connects |

### 2.2 How Evil Twins Actually Trick Victims
Since a device typically auto-connects to any network matching a previously-saved SSID, an evil twin exploits the fact that **most devices only check the network name (SSID), not the BSSID**, when deciding whether to auto-reconnect. A device can be tricked into joining an evil twin without any user interaction at all, especially if the attacker also sends deauth frames against the *real* AP to push clients toward the fake one — a technique often bundled into all-in-one tools like Airgeddon (Section 8).

### 2.3 Detection Approaches
- **Duplicate SSID/mismatched BSSID detection** — flagging when the same SSID appears broadcast from an unexpected or unknown BSSID/MAC vendor.
- **Signal strength anomalies** — a legitimate AP's signal characteristics staying consistent over time; a new, unexplained signal claiming the same identity is suspicious.
- **Continuous monitoring** — this is exactly what Day 17's Kismet is well-suited for: passively cataloging every AP/BSSID pair over time and alerting on anything new or duplicated.
- **Wired-side detection** — checking for unexpected devices/MAC addresses appearing on the wired network, which can indicate someone plugged in an unauthorized AP.

### 2.4 Why It Matters
- Rogue APs — especially evil twins — bypass **every** other wireless security control, since the victim isn't attacking the real, properly-secured network at all; they're handing their traffic directly to an attacker who set up something that merely *looks* legitimate.
- This is one of the most practically dangerous techniques in this document precisely because it requires no cracking, no WPS vulnerability, and no password guessing at all.

---

## 3. Wireless Assessment (Lab)

This topic is about the **structured methodology** for running a wireless security assessment safely and thoroughly — bringing together Day 17's tools and this day's tools into one coherent, professional process, rather than randomly trying techniques.

### 3.1 A Structured Assessment Flow
```
1. Passive Reconnaissance    → Kismet (Day 17): map every AP/client in range, note anything unexpected
2. Rogue AP Check            → Compare observed SSIDs/BSSIDs against the organization's known-authorized list
3. WPA/WPA2 Handshake Testing  → Aircrack-ng/Wifite/hcxdumptool (Day 17 + this Day): capture and test password strength
4. WPS Testing                  → Reaver/Bully: check if WPS is enabled and vulnerable
5. Post-Connection Testing         → Bettercap: once on the network (with permission), assess what an attacker could do next
6. Reporting                          → Document every finding, its risk, and remediation, exactly like Day 8-9's assessments
```

### 3.2 Lab Safety & Scoping Practices
- **Use a dedicated, isolated test AP** whenever possible — physically separate from any production or shared network, ideally in a space (or RF-shielded environment) that minimizes interference with real neighboring networks.
- **Get explicit, written scope** before any assessment — which SSIDs/BSSIDs are in scope, which techniques are authorized (e.g., is deauthentication/WPS testing approved, or off-limits due to business impact?).
- **Time-box intrusive tests** — deauth attacks and WPS brute-forcing can disrupt legitimate users; agree on a testing window with the network owner.
- **Log everything** — every command run, every capture file, every timestamp — both for your own report and to prove exactly what was (and wasn't) done, in case anything is questioned later.

### 3.3 Why It Matters
- Wireless assessments touch shared physical spectrum and real user connectivity in a way most of the earlier tools in this course don't — a structured, scoped methodology is what separates a professional assessment from something that looks like an actual attack.
- This structured flow is also just good practice generally: passive before active, broad before deep — the same principle from Day 3's recon sequencing, now applied to the wireless domain.

---

## 4. Tool: hcxdumptool

**hcxdumptool** is a modern wireless capture tool built specifically to gather WPA/WPA2 key material efficiently — including via the **PMKID** technique, a faster, often client-independent alternative to waiting for a traditional 4-way handshake.

### 4.1 How It Works
- Like Aircrack-ng's `airodump-ng`, hcxdumptool puts the adapter into monitor mode and captures wireless frames — but it's purpose-built to specifically extract the data needed for **hashcat**-based cracking (Day 15), rather than the older `.cap` format Aircrack-ng uses.
- Its standout feature is **PMKID capture**: many routers include a PMKID (Pairwise Master Key Identifier) in the very first message of the handshake process, sent as soon as *any* client (including the attacker's own device) merely associates with the AP — meaning a full 4-way handshake, and even a connected legitimate client, isn't strictly required to capture crackable material.
- Captured data is saved in a `.pcapng` file, then converted (using a companion tool, `hcxpcapngtool`) into a hash format hashcat can crack directly (mode `22000`, the modern unified WPA-PBKDF2 format).

### 4.2 What It's Used For
- **Faster, often stealthier handshake/PMKID capture** than the deauth-based Aircrack-ng approach — since PMKID capture doesn't necessarily require forcibly disconnecting a real client (no deauth frames needed in the ideal case).
- Feeding directly into **Hashcat** (Day 15) for GPU-accelerated cracking, taking advantage of Hashcat's speed advantage over Aircrack-ng's own built-in (CPU-only) cracking.

### 4.3 Important Commands

```bash
sudo airmon-ng start wlan0
```
Enables monitor mode — the same prerequisite step as Day 17's tools.

```bash
sudo hcxdumptool -i wlan0mon -o capture.pcapng
```
The core command — `-i` sets the monitor-mode interface, `-o` sets the output capture file, and hcxdumptool begins passively/actively gathering handshake and PMKID material from all visible networks.

```bash
sudo hcxdumptool -i wlan0mon -o capture.pcapng --enable_status=1
```
`--enable_status` shows **live status output** during the capture (e.g., how many handshakes/PMKIDs found so far) — useful for knowing when you've captured enough to stop.

```bash
sudo hcxdumptool -i wlan0mon -o capture.pcapng --filterlist=targets.txt --filtermode=2
```
Restricts capture to **specific target BSSIDs** listed in a file — keeps the capture scoped to authorized targets only, rather than every network in range.

```bash
hcxpcapngtool -o hashes.22000 capture.pcapng
```
Converts the raw `.pcapng` capture into hashcat's **mode 22000** hash format — the required conversion step before cracking.

```bash
hashcat -m 22000 -a 0 hashes.22000 rockyou.txt
```
Feeds the converted hash file directly into **Hashcat** (Day 15) for a GPU-accelerated dictionary attack — the payoff step that makes hcxdumptool's speed advantage worthwhile.

---

## 5. Tool: Reaver

**Reaver** is a dedicated **WPS PIN brute-forcing** tool — it directly implements the WPS vulnerability described in Section 1.1, systematically testing PIN halves against a target router.

### 5.1 How It Works
- Reaver targets a specific AP's WPS implementation, exploiting the fact that the 8-digit PIN is validated in two smaller, independently-checkable halves (Section 1.1).
- It sends a sequence of WPS authentication requests, narrowing down the correct PIN halves through the router's own responses — since a full brute-force of the *effective* search space (rather than the full 8 digits) is often computationally feasible.
- Because it depends on repeatedly querying the live router, it's inherently an **online, network-based attack** (unlike the offline handshake-cracking in Section 4) — meaning it's directly affected by the target's response speed and any WPS lockout protections the router may have.

### 5.2 What It's Used For
- Testing whether a target AP's WPS implementation is vulnerable to PIN brute-forcing — a completely separate check from WPA2 password strength.
- Recovering the actual WPA2 password **indirectly** — once the WPS PIN is found, the router will hand over the actual Wi-Fi password through the WPS protocol itself, bypassing the need to crack the password at all.

### 5.3 Important Commands

```bash
sudo airmon-ng start wlan0
```
Enables monitor mode, the shared prerequisite.

```bash
wash -i wlan0mon
```
`wash` (bundled with Reaver) **scans for WPS-enabled APs** specifically — the essential first step, since Reaver is only useful against targets that actually have WPS enabled.

```bash
sudo reaver -i wlan0mon -b AA:BB:CC:DD:EE:FF -vv
```
The core command — `-i` sets the interface, `-b` targets a specific **BSSID**, and `-vv` gives verbose output showing each PIN attempt and the router's response.

```bash
sudo reaver -i wlan0mon -b AA:BB:CC:DD:EE:FF -c 6 -vv
```
`-c` specifies the target's **channel** explicitly, which can improve reliability versus letting Reaver auto-detect it.

```bash
sudo reaver -i wlan0mon -b AA:BB:CC:DD:EE:FF -d 5 -vv
```
`-d` sets a **delay** (in seconds) between PIN attempts — slower, but reduces the chance of triggering the router's WPS lockout protection.

```bash
sudo reaver -i wlan0mon -b AA:BB:CC:DD:EE:FF -p 12345670
```
`-p` starts the attack with a **specific known/first PIN**, useful for resuming a previous session or testing a specific suspected default PIN.

```bash
sudo reaver -i wlan0mon -b AA:BB:CC:DD:EE:FF --session=session1
```
`--session` saves attack progress under a named session, allowing you to **stop and resume** a long-running attack later without starting over.

---

## 6. Tool: Bully

**Bully** is an alternative WPS brute-forcing tool, functionally similar to Reaver but implemented as an **independent rewrite** — it targets the same WPS vulnerability but is generally regarded as **more stable and resilient** against certain router-side countermeasures/edge cases than Reaver.

### 6.1 How It Works
- Like Reaver, Bully exploits the two-halves WPS PIN validation weakness described in Section 1.1.
- Its internal implementation handles some of the low-level protocol timing and state-tracking differently from Reaver, which in practice means it sometimes succeeds against routers/firmware where Reaver stalls or fails — making it a valuable **second option** rather than a strict replacement.

### 6.2 What It's Used For
- The **go-to fallback** when Reaver fails to make progress against a WPS-enabled target — a common, expected part of a real WPS assessment workflow, since router firmware behavior varies significantly.
- Same end goal as Reaver: recovering the WPS PIN, and through it, the actual WPA2 password.

### 6.3 Important Commands

```bash
sudo bully wlan0mon -b AA:BB:CC:DD:EE:FF -v 3
```
The core command — targets a specific **BSSID** (`-b`) with verbosity level 3 (`-v 3`) for detailed progress output, directly comparable to Reaver's `-vv`.

```bash
sudo bully wlan0mon -b AA:BB:CC:DD:EE:FF -c 6
```
`-c` specifies the **channel**, same purpose as Reaver's `-c`.

```bash
sudo bully wlan0mon -b AA:BB:CC:DD:EE:FF -d
```
`-d` enables **detection of the "pixie dust" vulnerability conditions** in some builds — a related, much faster WPS attack technique that exploits weak randomness in some routers' key generation (worth knowing exists as an even faster alternative on vulnerable hardware).

```bash
sudo bully wlan0mon -b AA:BB:CC:DD:EE:FF -B
```
`-B` enables **Bruteforce mode** explicitly, forcing the traditional online PIN-guessing approach rather than any faster shortcut technique.

```bash
sudo bully wlan0mon -b AA:BB:CC:DD:EE:FF -F
```
`-F` forces the tool to continue even if it detects the router may have **WPS locked** — useful for testing whether a suspected lockout is actually being enforced.

```bash
sudo bully wlan0mon -b AA:BB:CC:DD:EE:FF -o results.txt
```
`-o` saves progress/results to a file — helpful for long-running sessions and later reporting.

---

## 7. Tool: Bettercap

**Bettercap** is a powerful, modular network attack and monitoring framework — while it supports wireless-specific reconnaissance, its real strength (and its role in this document) is **post-connection MITM testing**, directly implementing the concept from Section 1.2.

### 7.1 How It Works
- Bettercap runs as an interactive framework with a large set of loadable **modules**, each handling a specific capability — ARP spoofing, DNS spoofing, HTTP/HTTPS traffic sniffing/proxying, and more.
- Once positioned on a network (having joined it via any method — a cracked password, a WPS PIN via Reaver/Bully, or simple authorized access), Bettercap can be pointed at specific victim IPs to begin intercepting their traffic path through **ARP spoofing** — telling both the victim and the gateway that the attacker's machine is the other party.
- Its **caplets** (scripted command sequences) let you chain multiple modules together into a repeatable, automated attack/test sequence.
- It also has wireless-specific modules (`wifi.recon`, `wifi.deauth`) letting it perform some of the same scanning/deauth functions as Aircrack-ng, all from within one unified framework.

### 7.2 What It's Used For
- **Post-connection assessment** — demonstrating what becomes possible once an attacker has network access, which is the natural conclusion of a wireless assessment: "we got on the network — now what could we actually do?"
- Testing whether traffic on the network is properly encrypted end-to-end, by attempting to intercept and inspect it via MITM positioning.
- A single, unified tool covering both some wireless recon functions and the full post-connection MITM testing phase.

### 7.3 Important Commands

```bash
sudo bettercap -iface wlan0
```
Launches Bettercap's interactive session on the specified interface — the entry point to everything else below.

```bash
net.probe on
```
(Inside Bettercap) Actively probes the local network to **discover connected hosts** — Bettercap's equivalent of a quick network host-discovery sweep.

```bash
net.show
```
Displays the list of **discovered hosts** on the network so far, along with IP/MAC addresses — the "who's here" view before choosing a target.

```bash
set arp.spoof.targets 192.168.1.50
arp.spoof on
```
Sets a specific **victim IP** and starts **ARP spoofing** against it — positioning the attacker's machine in the middle of that victim's traffic, directly implementing Section 1.2's MITM concept.

```bash
net.sniff on
```
Starts **sniffing** traffic passing through the now-intercepted connection — capturing what the victim is sending/receiving.

```bash
wifi.recon on
```
Enables Bettercap's **wireless reconnaissance module** — scans for nearby APs/clients, similar in spirit to Kismet/`airodump-ng` from Day 17.

```bash
wifi.deauth AA:BB:CC:DD:EE:FF
```
Sends **deauthentication frames** targeting a specific AP/client, the same technique as Aircrack-ng's `aireplay-ng --deauth` but accessible from within Bettercap's unified interface.

```bash
help arp.spoof
```
Shows detailed **help/options** for any specific module — useful for exploring the many available modules beyond the core ones covered here.

---

## 8. Tool: Airgeddon

**Airgeddon** is an **all-in-one wireless auditing wrapper** — a guided, menu-driven Bash script that ties together Aircrack-ng, Reaver/Bully, Bettercap, and several other tools into one unified interface, so you don't have to remember or manually chain every individual command yourself.

### 8.1 How It Works
- Airgeddon presents a series of **interactive text menus** — choose your interface, enable monitor mode, pick a scan type, select a target, and choose an attack — with the script handling the underlying tool invocations (many of the exact commands from Sections 4–7 above) behind the scenes.
- It supports a very wide range of attack categories from one place: WPA/WPA2 handshake capture, WPS PIN attacks (via Reaver/Bully), **Evil Twin attacks** (directly implementing Section 2's rogue-AP concept, including an optional captive portal to try to capture the real password from a tricked victim), and DoS/deauth testing.
- Because it's a wrapper, understanding the *individual* tools first (as this whole document does) makes Airgeddon far more useful — you'll actually understand what each menu option is really doing underneath, rather than clicking through blindly.

### 8.2 What It's Used For
- **Fast, guided assessments** covering multiple attack categories (WPA handshake, WPS, evil twin) in a single session without manually re-typing each underlying tool's syntax.
- **Demonstrating Evil Twin/rogue AP attacks** specifically — Airgeddon is one of the most accessible ways to actually set up and observe this technique in a lab, directly illustrating Section 2's concepts hands-on.
- A strong choice for structured lab exercises (Section 3) where you want to move through several assessment phases in one guided session.

### 8.3 Important Commands

```bash
sudo airgeddon
```
Launches the tool — the **only command you actually type**; everything else happens through Airgeddon's interactive menu system.

```
# Menu flow (illustrative, exact numbering varies by version):
1. Select your wireless interface from the detected list
2. Choose "Put interface in monitor mode"
3. Select a main menu category:
   - WEP/WPA/WPA2 attacks (handshake capture, offline cracking)
   - WPS attacks (wraps Reaver/Bully)
   - Evil Twin attacks (rogue AP + optional captive portal)
   - DoS attacks (deauth flooding)
4. Follow the guided target-selection screen (scans and lists nearby APs)
5. Confirm the specific attack technique and any required parameters
6. Airgeddon runs the underlying tool(s) and displays live progress
```
This menu-driven flow is Airgeddon's actual "command interface" — each numbered choice ultimately maps to one of the specific commands already covered in Sections 4–7 of this document, run automatically on your behalf.

---

## 9. Quick Cheat Sheet

| Task | Command |
|---|---|
| Capture WPA handshake/PMKID for Hashcat | `sudo hcxdumptool -i wlan0mon -o capture.pcapng` |
| Convert capture for Hashcat | `hcxpcapngtool -o hashes.22000 capture.pcapng` |
| Scan for WPS-enabled APs | `wash -i wlan0mon` |
| WPS PIN attack (primary) | `sudo reaver -i wlan0mon -b <BSSID> -vv` |
| WPS PIN attack (fallback) | `sudo bully wlan0mon -b <BSSID> -v 3` |
| Post-connection MITM (ARP spoof) | `set arp.spoof.targets <IP>` then `arp.spoof on` (in Bettercap) |
| Guided, all-in-one wireless assessment | `sudo airgeddon` |

**Typical Day 18 Assessment Workflow (isolated lab environment):**
```bash
sudo airmon-ng start wlan0                                     # 1. Monitor mode (shared prerequisite)
wash -i wlan0mon                                                   # 2. Check if WPS is enabled on the target
sudo reaver -i wlan0mon -b AA:BB:CC:DD:EE:FF -vv                       # 3a. Try WPS attack (Reaver)
sudo bully wlan0mon -b AA:BB:CC:DD:EE:FF -v 3                             # 3b. Fallback WPS attack (Bully) if Reaver stalls
sudo hcxdumptool -i wlan0mon -o capture.pcapng                               # 4. Capture handshake/PMKID if WPS isn't viable
hcxpcapngtool -o hashes.22000 capture.pcapng && hashcat -m 22000 -a 0 hashes.22000 rockyou.txt   # 5. Offline crack
sudo bettercap -iface wlan0                                                     # 6. Post-connection MITM assessment
sudo airgeddon                                                                     # 7. Or: run the whole flow guided, end-to-end
```

---

### Practice Suggestions for Day 18
1. Run `wash -i wlan0mon` against your isolated lab AP with WPS enabled, and confirm it's detected before attempting Reaver.
2. Run Reaver against the lab AP with a deliberate `-d` delay, and time how long a full PIN attack realistically takes.
3. If Reaver stalls, try the same target with Bully and compare behavior/output.
4. Capture a handshake/PMKID with hcxdumptool, convert it with `hcxpcapngtool`, and crack it with Hashcat — compare the speed against Aircrack-ng's own cracking from Day 17.
5. In a fully isolated lab, use Bettercap to ARP-spoof a test victim device you control, and use `net.sniff` to observe unencrypted traffic — then repeat while the victim only visits HTTPS sites and note the difference.
6. Run Airgeddon's Evil Twin menu option in a controlled lab against your own test AP, and document exactly what a connecting victim would see — tie this back to Section 2's rogue AP concepts.
7. Write a short "Rogue AP Detection" checklist (Section 2.3) as if advising an organization on what to monitor for, based on everything demonstrated this session.
