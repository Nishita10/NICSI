# Day 22 — Exploitation Tools: Metasploit & SearchSploit

## 1. Day 22 Overview

| | |
|---|---|
| **Day** | 22 |
| **Topic** | Exploitation Concepts, Metasploit Framework, SearchSploit |
| **Tools** | Metasploit Framework, SearchSploit |
| **Environment** | Kali Linux (isolated lab) |

Day 22 focused on understanding how vulnerabilities are turned into practical exploitation activity in a controlled lab, using two of the most common tools in a penetration tester's toolkit: **Metasploit Framework** for exploit execution and management, and **SearchSploit** for offline exploit-database lookups. The goal was conceptual understanding and safe, authorized hands-on practice — not building new exploits.

---

## 2. Learning Objectives

By the end of this day, I should be able to:

- Explain the difference between a vulnerability, an exploit, and a payload.
- Describe the general vulnerability-to-exploit workflow used in penetration testing.
- Explain Metasploit's architecture and module categories.
- Use `msfconsole` and its core commands (`search`, `use`, `info`, `show options`, `set`, `run`/`exploit`).
- Explain what module options like `RHOSTS`, `RPORT`, `LHOST`, `LPORT` represent.
- Explain payload concepts (reverse shell, bind shell, staged vs. stageless).
- Use SearchSploit to look up publicly documented exploits and understand its relationship to Exploit-DB.
- Compare Metasploit and SearchSploit and know when each is appropriate.
- Describe, at a conceptual level, the basics of exploit development and common memory-safety mitigations.
- Explain the ethical and legal boundaries around exploitation.

---

## 3. Exploitation Fundamentals

| Term | Meaning |
|---|---|
| **Vulnerability** | A weakness in software, configuration, or design that could be abused. |
| **Exploit** | Code or a technique that takes advantage of a specific vulnerability. |
| **Exploitation** | The act of using an exploit against a target. |
| **Payload** | The code that runs on the target after successful exploitation (e.g., a shell). |
| **Target** | The system or service being tested. |
| **Listener** | A process on the attacker's machine waiting for an incoming connection from a payload. |
| **Shell** | An interactive command interface obtained on the target. |
| **RCE** | Remote Code Execution — running arbitrary code on a target over a network. |
| **LPE** | Local Privilege Escalation — gaining higher privileges from an existing foothold. |
| **Initial Access** | The first foothold gained on a target system or network. |

**Conceptual workflow:**

```text
Vulnerability
   ↓
Identify suitable exploit
   ↓
Select target/module
   ↓
Configure required options
   ↓
Select payload
   ↓
Execute in authorized environment
   ↓
Observe result
   ↓
Post-exploitation / cleanup
```

A vulnerability does not automatically mean a usable exploit exists, and having an exploit does not guarantee it will work against a specific target. Each stage above requires verification, not assumption.

---

## 4. Vulnerability-to-Exploit Workflow

A structured, professional workflow avoids wasted effort and reduces the risk of damaging a target:

1. Identify the service/software and its version.
2. Identify a candidate vulnerability affecting that version.
3. Verify the vulnerability actually applies to the target (not just the software name).
4. Search for a suitable exploit (Metasploit, SearchSploit, Exploit-DB, vendor advisories).
5. Evaluate the exploit's reliability and prerequisites (e.g., authentication, specific patch level).
6. Select the most appropriate exploit.
7. Configure target-specific parameters (IP, port, target OS/architecture).
8. Select an appropriate payload, if the exploit requires one.
9. Execute **only** against a system I am explicitly authorized to test.
10. Verify whether the result actually indicates success (not just "it ran").
11. Document findings clearly, including failures.
12. Clean up: close sessions, remove any dropped files, restore lab state.

Blindly running exploits without following this process is bad practice because it can crash services, corrupt data, trigger unintended side effects, or produce false confidence in a result that didn't actually succeed.

---

## 5. Metasploit Framework

**Metasploit Framework** is an open-source platform for developing, testing, and executing exploit code against a target in a controlled manner. Penetration testers use it because it standardizes exploitation into a consistent module structure, handles payload generation and delivery, and provides session/post-exploitation management — instead of every tester writing raw exploit code from scratch each time.

**`msfconsole`** is the primary interactive command-line interface to the framework.

**Module categories:**

| Module Type | Purpose |
|---|---|
| **Exploit** | Code that leverages a specific vulnerability to gain code execution or a similar outcome. |
| **Auxiliary** | Non-exploitation modules — scanning, fuzzing, enumeration, denial-of-service testing, etc. |
| **Payload** | The code delivered/executed on the target after successful exploitation. |
| **Encoder** | Transforms payload bytes to avoid bad characters or basic signature matching in a lab context. |
| **NOP** | "No operation" generators used to pad payloads for certain memory-corruption exploits. |
| **Post** | Modules run after a session is established (e.g., gathering system info, pivoting). |

---

## 6. Metasploit Architecture

```text
Target
   ↓
Vulnerability
   ↓
Exploit Module
   ↓
Payload
   ↓
Session
   ↓
Post-Exploitation
```

- **Exploit** — the mechanism that triggers the vulnerability.
- **Payload** — what runs once the exploit succeeds.
- **Handler** — the listener component that manages the connection from a payload (for example, `exploit/multi/handler` when working with a payload generated outside of a direct exploit module).
- **Session** — an active, interactive connection to the target created after successful exploitation, tracked by Metasploit so it can be interacted with, backgrounded, or handed to post-exploitation modules.

These four concepts are distinct: an exploit gets you in, a payload defines what runs once you're in, a handler catches the resulting connection, and a session is the ongoing interactive access.

---

## 7. Starting Metasploit

```bash
msfconsole
```

| Command | Purpose | Example | Explanation |
|---|---|---|---|
| `help` | Lists available commands | `help` | Shows console-wide command reference. |
| `version` | Shows framework version | `version` | Useful for confirming module/database compatibility. |
| `search` | Searches for modules | `search type:exploit smb` | Finds modules matching keywords/filters. |
| `info` | Shows module details | `info exploit/…` | Displays description, options, targets, references. |
| `use` | Loads a module | `use exploit/…` | Makes the module the active context. |
| `show options` | Lists a module's configurable options | `show options` | Shows required/optional parameters and current values. |
| `show payloads` | Lists compatible payloads | `show payloads` | Shows payloads that work with the loaded exploit. |
| `set` | Sets an option value | `set RHOSTS 10.0.0.5` | Configures a required or optional parameter. |
| `unset` | Clears an option value | `unset RHOSTS` | Removes a previously set value. |
| `back` | Exits current module context | `back` | Returns to the main `msf6 >` prompt. |
| `exit` | Closes msfconsole | `exit` | Ends the session. |

---

## 8. Searching for Exploits in Metasploit

```text
search <keyword>
```

A search can be built around:

- Software name (e.g., `search wordpress`)
- Product name
- Version string
- Vulnerability identifier (e.g., `search type:exploit apache`)
- CVE number (e.g., `search cve:2021-xxxxx`)

**Interpreting search result fields:**

| Field | Meaning |
|---|---|
| **Name** | The module's path/identifier. |
| **Disclosure Date** | When the vulnerability was first publicly disclosed. |
| **Rank** | Metasploit's estimate of reliability/stability (e.g., excellent, good, normal, low). |
| **Check** | Whether the module supports a safe "check" mode to test applicability without exploiting. |
| **Description** | A short summary of what the module targets. |

No specific search results are reproduced here, since actual output depends on the live Metasploit database at the time of the lab session.

---

## 9. Selecting and Inspecting a Module

```text
search
   ↓
use
   ↓
info
   ↓
show options
   ↓
show payloads
```

`info` and `show options` should always be reviewed **before** execution because they reveal:

- What the module actually does and what conditions it assumes.
- Which options are required vs. optional.
- Which targets/platforms are supported.
- Any known caveats (e.g., "may crash the service").

**Illustrative Example — Not My Actual Result**

```text
msf6 > search type:exploit ftp
msf6 > use exploit/example/ftp_example
msf6 exploit(example/ftp_example) > info
msf6 exploit(example/ftp_example) > show options
```

This sequence is shown only to illustrate command order, not as evidence of an exploit I ran.

---

## 10. Understanding Metasploit Options

| Option | Meaning |
|---|---|
| **RHOSTS** | Remote target address(es) — the system(s) being targeted. |
| **RPORT** | Remote target service port. |
| **LHOST** | Local/listener address — where the payload will connect back to, or bind from. |
| **LPORT** | Local/listener port. |
| **TARGET** | The specific OS/platform variant the exploit should assume (some exploits support multiple). |
| **PAYLOAD** | The specific payload module to deliver on success. |

Not every module uses every option — auxiliary scanners, for example, may only need `RHOSTS`/`RPORT` and have no payload at all. The exact required options are always shown by `show options` for the currently loaded module.

---

## 11. Payload Concepts

A **payload** is the code that executes on the target once exploitation succeeds — commonly used to establish some form of shell access.

| Payload Type | Description |
|---|---|
| **Reverse shell** | The target initiates a connection back to the attacker's listener. |
| **Bind shell** | The target opens a listening port that the attacker connects to. |
| **Meterpreter** | An advanced, extensible Metasploit-specific payload with built-in post-exploitation features. |
| **Staged** | A small initial stager is sent first, which then pulls down the rest of the payload. |
| **Stageless** | The full payload is sent in one piece. |

```text
Reverse:  Target  ---(connects out)--->  Attacker Listener
Bind:     Attacker ---(connects in)--->  Target Listening Port
```

Reverse connections are often preferred in lab/real-world scenarios because outbound connections are less likely to be blocked than unsolicited inbound ones. This section is conceptual only — no payload construction or delivery instructions are included here.

---

## 12. Exploit Execution Workflow

```text
search
use
info
show options
set required options
show payloads
set payload
run / exploit
```

- **Why configuration must be verified:** incorrect `RHOSTS`/`RPORT`/`LHOST` values will cause failures or, worse, target the wrong system.
- **Why target authorization matters:** running exploit code against a system without permission is unethical and, in most jurisdictions, illegal, regardless of intent.
- **What happens conceptually after execution:** if the exploit succeeds, a session is created; if it fails, Metasploit typically reports an error or timeout — success is never guaranteed.
- **What a session represents:** an active, trackable connection to the target that can be interacted with or handed off to post-exploitation modules.

| Command | Purpose |
|---|---|
| `run` / `exploit` | Executes the configured module. |
| `sessions` | Lists and manages active sessions. |

No successful exploitation output is claimed or fabricated in this document.

---

## 13. SearchSploit

**SearchSploit** is a command-line search tool for the **Exploit-DB** archive, allowing offline searching of a locally mirrored copy of publicly documented exploits, shellcode, and papers.

- It is a **search and reference tool**, not an execution framework.
- It is directly tied to Exploit-DB's dataset, which is periodically synced locally.
- It's useful for quickly checking whether a public exploit exists for a given software/version without needing internet access during a lab or engagement.
- Results typically include exploit title, Exploit-DB ID, and a local file path to the exploit code/documentation.

```bash
searchsploit <keyword>
searchsploit -w <keyword>      # show the Exploit-DB web URL for a result
searchsploit -x <path>         # examine/view an exploit's contents
```

**Key difference from Metasploit:** SearchSploit only finds and displays references to exploits — it does not run them.

---

## 14. SearchSploit Practical Workflow

```text
Identify software/version
        ↓
Search with SearchSploit
        ↓
Review matching exploits
        ↓
Inspect exploit metadata/code
        ↓
Determine applicability
        ↓
Cross-reference vulnerability information
        ↓
Choose appropriate lab exercise
```

Version matching is critical — an exploit written for one specific version (or even build/patch level) of a product frequently fails, or behaves unpredictably, against a different version. Finding a matching exploit entry does **not** by itself confirm the target is vulnerable; it only indicates a documented issue exists for that software, which must still be verified against the actual target.

---

## 15. SearchSploit vs Metasploit

| Feature | Metasploit | SearchSploit |
|---|---|---|
| Primary purpose | Exploitation framework — configure and run exploits | Offline search tool for the Exploit-DB archive |
| Exploit database/search | Has its own curated, actively maintained module database | Mirrors the Exploit-DB archive locally |
| Exploit execution | Yes — modules can be configured and run directly | No — only locates and displays exploit code/info |
| Payload support | Yes — integrated payload generation and delivery | No — exploits are standalone scripts/code, payloads (if any) are manual |
| Exploit code inspection | Possible, but framework-oriented | Core use case — designed for reading raw exploit source |
| Automation | High — options, sessions, post modules are all scriptable | Low — primarily a lookup/reference tool |
| Typical use | Executing a tested, integrated exploit against an authorized target | Discovering whether a public exploit exists for a given software/version |

---

## 16. Exploit Development Basics

This section covers only the **fundamental concepts**, not advanced exploit-writing techniques.

| Concept | Description |
|---|---|
| **Vulnerability research** | Investigating software to find weaknesses, often via code review, fuzzing, or reverse engineering. |
| **Root cause** | The underlying programming/design flaw that makes a vulnerability possible. |
| **Attack surface** | The set of inputs/interfaces an attacker could interact with. |
| **Input handling** | How an application processes untrusted data — a common source of flaws. |
| **Memory corruption** | A class of bugs where a program writes/reads memory outside intended bounds. |
| **Buffer overflow** | A specific memory-corruption case where data exceeds an allocated buffer's size. |
| **Control-flow hijacking** | Redirecting a program's execution path, often as the goal of a memory-corruption exploit. |
| **Proof of concept (PoC)** | Minimal code demonstrating that a vulnerability is real and triggerable. |
| **Reliability** | How consistently an exploit works across environments/conditions. |
| **Exploit constraints** | Environmental factors (OS version, architecture, memory layout) that affect whether an exploit works. |
| **Mitigations** | Defensive mechanisms that make exploitation harder. |

**Common mitigations:**

- **ASLR** — Address Space Layout Randomization; randomizes memory locations.
- **DEP/NX** — Marks memory regions non-executable to prevent injected code from running.
- **Stack canaries** — Values placed on the stack to detect buffer overflows before they corrupt control data.
- **PIE** — Position-Independent Executables; extends ASLR to the executable itself.
- **Control Flow Integrity (CFI)** — Restricts a program's execution to a set of valid, expected control-flow paths.

Modern exploit development typically requires understanding both the specific vulnerability *and* which of these mitigations are active on the target, since a technique that works with mitigations disabled often needs to be adapted — or may not be feasible at all — when they're enabled.

---



## 17. Practical Command Reference

### Metasploit

```text
msfconsole        # start the framework
search             # find modules
use                # load a module
info               # view module details
show options       # view configurable options
show payloads      # view compatible payloads
set                # set an option value
unset               # clear an option value
back                # exit current module context
run / exploit       # execute the module
sessions            # list/manage active sessions
```

### SearchSploit

```bash
searchsploit <keyword>   # search the local Exploit-DB mirror
```

---

## 18. Common Mistakes

- Running exploits without verifying authorization.
- Selecting an exploit solely because the software name matches, ignoring the version.
- Ignoring software version differences entirely.
- Not reading `info` before running a module.
- Not checking required options with `show options`.
- Confusing `RHOSTS` (target) with `LHOST` (attacker/listener).
- Assuming an exploit guarantees success.
- Confusing a vulnerability (a weakness) with an exploit (code that abuses it).
- Confusing an exploit with a payload.
- Blindly copying and running exploit code without understanding it.
- Failing to document results, including failed attempts.

---

## 19. Ethics and Legal Considerations

- Exploitation should **only** be performed against systems for which explicit, documented authorization exists.
- Publicly available exploit code should be handled carefully — reading and understanding it is different from running it against a live system.
- Intentionally vulnerable lab applications and isolated virtual environments are appropriate, safe places to practice.
- Unauthorized exploitation can cause real damage (service outages, data loss, legal consequences) and is illegal in most jurisdictions regardless of intent.
- Where a real vulnerability is discovered outside a lab context, responsible disclosure to the affected vendor/organization is the appropriate path, not independent exploitation.

---

## 20. What I Learned

Working through Day 22, I gained an understanding of how a vulnerability moves from being a theoretical weakness to something a penetration tester can practically act on, and why that process involves several distinct, verifiable steps rather than a single action. I learned how Metasploit organizes exploitation into modules, options, and payloads, and how `msfconsole` ties these pieces together through a consistent workflow (`search` → `use` → `info` → `show options` → `set` → `run`). I also came to understand SearchSploit's role as a reference and discovery tool built on the Exploit-DB archive, and why it complements rather than replaces Metasploit — one helps you find documented exploits, the other helps you configure and run supported ones.

I also gained a clearer sense of why version verification and careful review of module requirements matter: a name match alone says very little about whether an exploit will actually work. Separating the concepts of vulnerability, exploit, and payload — and seeing where a session and post-exploitation fit into that chain — helped me understand exploitation as one stage in a larger penetration testing process, not the whole process itself.

---

## 21. Key Takeaways

- Vulnerability ≠ exploit.
- Exploit ≠ payload.
- Version matching matters — a name match is not enough.
- Metasploit provides a modular, standardized exploitation framework.
- SearchSploit helps locate publicly available exploit references via Exploit-DB.
- Exploit success depends heavily on target-specific conditions.
- `info` and `show options` should always be reviewed before execution.
- Exploitation requires explicit authorization — no exceptions.
- Exploit development requires understanding both root cause and active mitigations.
- Sessions represent ongoing access and should be tracked and closed properly.
- Documentation, including failures, is a core part of security testing.

---

## 22. Suggested Further Practice

- Practice discovering Metasploit modules for different software categories using `search`.
- Practice identifying a module's required options using `info` and `show options` without running it.
- Search known, historical vulnerabilities using SearchSploit to see how results are structured.
- Compare a SearchSploit result against an equivalent Metasploit module for the same vulnerability.
- Study an intentionally vulnerable application (e.g., a dedicated training VM) in an isolated lab network.
- Read the source of a documented exploit to understand its structure and logic.
- Analyze why a given exploit might succeed against one target configuration and fail against another.

*(Practice should remain confined to isolated lab environments and systems I am explicitly authorized to test — never public or third-party systems.)*

---

## 23. Final Summary

Day 22 covered the conceptual and practical foundation of exploitation using Metasploit Framework and SearchSploit. The focus was on understanding the vulnerability-to-exploit-to-payload pipeline, Metasploit's module-based architecture, and SearchSploit's role as an offline Exploit-DB reference tool, alongside the ethical and procedural discipline required to use these tools responsibly. This groundwork supports more advanced exploitation and post-exploitation topics in later stages of the course, always within authorized, isolated lab environments.