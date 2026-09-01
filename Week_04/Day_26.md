# Day 26 — Reverse Engineering & Binary Analysis

> **Platform:** Kali Linux
>
> **Focus:** Static Analysis, Dynamic Analysis and Binary Inspection
>
> **Tools:** Ghidra, Radare2, GDB, Objdump, Strings

## 1. Day 26 Overview

**Binary analysis** is the process of examining an executable program's structure and behavior. **Reverse engineering** is the broader practice of understanding how something works — in this context, a compiled program — without having its original source code.

Reverse engineering matters in cybersecurity because analysts frequently encounter binaries with no available source: suspected malware, proprietary software being assessed for vulnerabilities, or firmware components (building directly on Day 25). Understanding how to examine such a binary is a core investigative skill.

Two complementary approaches were covered:

- **Static analysis** — examining a program without executing it.
- **Dynamic analysis** — observing a program's behavior while it runs, in a controlled environment.

Day 26 introduced five tools spanning both approaches: **Strings**, **Objdump**, **Ghidra**, **Radare2**, and **GDB**.

---

## 2. Learning Objectives

By the end of this day, I should be able to:

- Understand what executable binaries are.
- Understand static analysis.
- Understand dynamic analysis.
- Understand binary inspection as a general practice.
- Understand disassembly.
- Understand decompilation.
- Understand assembly instructions at a foundational level.
- Understand ELF binaries on Linux.
- Understand functions and control flow at a conceptual level.
- Understand symbols and sections.
- Understand strings within binaries.
- Understand debugging.
- Understand breakpoints.
- Understand registers and memory at a foundational level.
- Understand Ghidra, Radare2, GDB, Objdump, and Strings individually.
- Understand how these tools complement one another.

This document reflects a foundational understanding of reverse engineering, not advanced expertise.

---

## 3. What Is Reverse Engineering?

Reverse engineering in cybersecurity means analyzing a compiled program to understand its structure, logic, and behavior — typically without access to its original source code.

```text
Source Code
     ↓
Compiler
     ↓
Executable Binary
     ↓
Reverse Engineering
     ↓
Disassembly / Decompilation
     ↓
Program Structure & Logic
```

A compiler transforms human-readable source code into a binary executable containing machine code. Reverse engineering works backward from that binary — using disassembly (converting machine code to assembly) and decompilation (approximating higher-level logic) — to build an understanding of what the program does, since the original source is usually unavailable.

---

## 4. Binary Fundamentals

| Concept | Description |
|---|---|
| **Binary** | An executable file containing compiled machine code and supporting data/structure. |
| **Machine code** | The raw instructions directly executed by the CPU. |
| **Assembly** | A human-readable textual representation of machine instructions, roughly one-to-one with machine code. |
| **Opcode** | The operation an instruction performs (e.g., move data, add, compare). |
| **Registers** | Small, extremely fast storage locations within the CPU used for active computation. |
| **Memory** | Runtime storage (RAM) used by a running process for data beyond what fits in registers. |
| **Instruction Pointer** | A register (e.g., `RIP` on x86-64) tracking the address of the next instruction to execute. |
| **Stack** | A region of memory used for function call management and local variables, growing/shrinking as functions are called/return. |
| **Heap** | A region of memory used for dynamically allocated data whose size or lifetime isn't known at compile time. |

This document keeps CPU architecture discussion at a foundational level appropriate for an introduction to reverse engineering.

---

## 5. ELF Basics

On Linux (including Kali), executables typically use the **ELF (Executable and Linkable Format)**.

Key concepts:

- **ELF header** — identifies the file as ELF and describes its overall structure (architecture, entry point, etc.).
- **Sections** — named regions organizing information within the file, primarily relevant to linking and analysis tools.
- **Segments** — instructions to the loader about what to map into process memory and how, relevant at runtime.
- **Symbols** — named references to functions/variables, useful for readability during analysis.
- **Dynamic linking / shared libraries** — many ELF binaries depend on external libraries loaded at runtime rather than containing all code directly.

**Common sections:**

| Section | Purpose |
|---|---|
| `.text` | Contains executable code. |
| `.data` | Contains initialized global/static variables. |
| `.rodata` | Contains read-only data (e.g., string constants). |
| `.bss` | Reserves space for uninitialized global/static variables (no data stored in the file itself). |

**Sections vs. Segments:** Sections are used primarily for organizing information in an object/executable file (useful to linkers and analysis tools), while segments are used by the loader to map relevant portions of the file into process memory when the program runs. A single segment can encompass multiple sections.

---

## 6. Static vs. Dynamic Analysis

| Feature | Static Analysis | Dynamic Analysis |
|---|---|---|
| Execution required | No | Yes |
| Main objective | Understand structure/logic without running the program | Observe actual runtime behavior |
| Typical evidence | Strings, disassembly, sections, symbols | Registers, memory, system calls, execution flow |
| Tools | Strings, Objdump, Ghidra (static mode), Radare2 (static mode) | GDB, Radare2 (debug mode), Ghidra (with debugger integration) |
| Advantages | Safer — no execution risk; can examine the whole program | Reveals actual behavior, including runtime-only logic |
| Limitations | Can miss behavior dependent on runtime conditions or obfuscation | Only observes the specific execution path taken; requires safe execution environment |
| Typical use | Initial triage, structural understanding, hypothesis generation | Validating hypotheses, observing specific behavior in detail |

### Static Analysis
Examining a program without executing it — reviewing strings, disassembly, function structure, imports/exports, sections, and control-flow structure.

### Dynamic Analysis
Observing a program while it executes in a controlled environment — watching registers, memory, system calls, function calls, runtime behavior, and (where relevant) network activity.

---

## 7. Binary Inspection Workflow

```text
Binary
  ↓
Identify File Type
  ↓
Inspect Metadata
  ↓
Extract Strings
  ↓
Inspect Sections / Symbols
  ↓
Disassemble
  ↓
Identify Functions
  ↓
Analyze Control Flow
  ↓
Debug / Execute in Lab
  ↓
Observe Runtime Behavior
  ↓
Correlate Static + Dynamic Findings
```

| Stage | Explanation |
|---|---|
| Identify File Type | Confirm what kind of binary this is (e.g., ELF, architecture). |
| Inspect Metadata | Review header information (entry point, architecture, etc.). |
| Extract Strings | Get a quick, low-effort overview of embedded text. |
| Inspect Sections / Symbols | Understand the binary's internal organization. |
| Disassemble | Convert machine code to assembly for detailed review. |
| Identify Functions | Locate discrete units of logic within the disassembly. |
| Analyze Control Flow | Understand how execution moves between functions/blocks. |
| Debug / Execute in Lab | Move to dynamic analysis in an isolated environment. |
| Observe Runtime Behavior | Watch actual execution — registers, memory, calls. |
| Correlate Findings | Combine static and dynamic observations into a coherent understanding. |

---

## 8. Strings

### What is `strings`?

`strings` is a Linux utility that extracts printable character sequences from a binary (or any file), providing a fast, low-effort first look at what text is embedded inside.

Why it's useful:

- Surfaces printable sequences without requiring any disassembly or execution.
- Can reveal URLs, file paths, error messages, and configuration information embedded in the binary.
- Considerations include ASCII vs. Unicode encoding — by default, `strings` may miss certain encodings unless told to look for them.

**Basic usage:**

```bash
strings <SAMPLE-BINARY>
```

```bash
strings -a <SAMPLE-BINARY>
```

`-a` (or `--all`) tells `strings` to scan the entire file rather than just sections typically expected to contain printable data, which can matter for certain binary layouts.

`strings` alone does not reveal the complete behavior of a binary — it only shows embedded text, not program logic.

---

## 9. Strings as a Reconnaissance Technique

Strings output can provide quick investigative clues, such as:

- URLs
- Domain names
- IP addresses
- File paths
- Command names
- Error messages
- Debug messages
- Configuration values

**Important caveat:** finding a suspicious string does **not** automatically prove that the binary performs the corresponding action. A string may be:

- Unused (dead code or leftover data)
- Obsolete (from an old version of the logic)
- Embedded in a linked library rather than the program's own logic
- Debug information not relevant to normal execution
- Plain data rather than something referenced by executable logic

Strings should be treated as leads to investigate further (e.g., by finding what references them, per Section 16), not conclusions in themselves.

---

## 10. Objdump

### What is Objdump?

`objdump` is a command-line utility for inspecting object/executable files — including ELF headers, sections, symbols, and disassembly — without needing a full reverse-engineering platform.

**Useful commands:**

```bash
objdump -f <SAMPLE-BINARY>
objdump -h <SAMPLE-BINARY>
objdump -t <SAMPLE-BINARY>
objdump -d <SAMPLE-BINARY>
```

| Flag | Purpose |
|---|---|
| `-f` | Display overall file header summary (format, architecture, entry point). |
| `-h` | Display the section headers (names, sizes, addresses). |
| `-t` | Display the symbol table. |
| `-d` | Disassemble the executable sections. |

Objdump is a lightweight way to get structural and disassembly information quickly, without the overhead of setting up a full project in a platform like Ghidra.

---

## 11. Objdump Disassembly

**Disassembly** converts raw machine code into human-readable assembly instructions.

Common instruction mnemonics an analyst will encounter:

| Instruction | General Meaning |
|---|---|
| `mov` | Move data between registers/memory. |
| `push` | Push a value onto the stack. |
| `pop` | Pop a value off the stack. |
| `call` | Call a function (jump, saving a return address). |
| `cmp` | Compare two values (sets flags for conditional logic). |
| `jmp` | Unconditional jump to another instruction. |
| `ret` | Return from a function call. |

**Illustrative Assembly Example** (not derived from any specific analyzed binary):

```asm
push   rbp
mov    rbp, rsp
mov    eax, 0
cmp    eax, 1
jne    <address>
ret
```

This snippet illustrates the general shape of a function prologue and a conditional comparison — it is not output from any actual binary examined during Day 26.

---

## 12. Ghidra

### What is Ghidra?

**Ghidra** is a software reverse-engineering platform, originally developed by the NSA and released as open source. It's widely used because it provides an integrated environment combining several reverse-engineering capabilities in one tool.

Key capabilities:

- **Project management** — organizing binaries and analysis results.
- **Binary import** — loading an executable for analysis.
- **Analysis engine** — automated processing to identify functions, references, and structure.
- **Disassembler** — converts machine code to assembly.
- **Decompiler** — approximates higher-level pseudocode from the disassembly.
- **Function identification** — automatically locating function boundaries.
- **Cross-references** — tracking what calls/references what.
- **Symbol management** — organizing named references (functions, variables) for readability.
- **Graph views** — visualizing control flow.

Ghidra can make compiled binaries substantially easier to understand by combining disassembly, decompilation, cross-referencing, and navigation in one integrated environment, compared to using several separate command-line tools.

---

## 13. Ghidra Basic Workflow

```text
Create Project
      ↓
Import Binary
      ↓
Analyze Binary
      ↓
Identify Functions
      ↓
Inspect Strings
      ↓
Inspect Imports / Symbols
      ↓
Open Disassembly
      ↓
Use Decompiler
      ↓
Trace References
      ↓
Understand Program Logic
```

| Stage | Explanation |
|---|---|
| Create Project | Set up a Ghidra project to hold imported binaries and analysis. |
| Import Binary | Load the target executable. |
| Analyze Binary | Run Ghidra's automated analysis to identify structure. |
| Identify Functions | Review functions Ghidra has located. |
| Inspect Strings | Review embedded strings within Ghidra's interface. |
| Inspect Imports/Symbols | Review external dependencies and named references. |
| Open Disassembly | View the assembly-level representation of a function. |
| Use Decompiler | View Ghidra's approximate higher-level pseudocode. |
| Trace References | Follow cross-references between functions/strings/data. |
| Understand Program Logic | Synthesize findings into an overall understanding. |

> **Lab Placeholder:** No specific binary has been imported and analyzed in Ghidra as part of Day 26 practical work yet — this workflow describes the intended process (see Section 32).

---

## 14. Ghidra Disassembler

The disassembler view presents:

- **Assembly listing** — instructions in readable mnemonic form.
- **Addresses** — the memory address of each instruction.
- **Instructions and operands** — the operation and the data/registers it acts on.
- **Functions** — grouped sequences of instructions Ghidra has identified as discrete units.
- **Basic blocks** — straight-line instruction sequences with a single entry/exit point, used to build control-flow graphs.
- **Cross-references** — links showing where a given address/function/string is used elsewhere.

Analysts typically move between the disassembly, the decompiled view, the strings list, and the function graph — each offers a different level of abstraction, and combining them builds a fuller picture than any single view alone.

---

## 15. Ghidra Decompiler

The decompiler converts machine code into higher-level pseudocode, attempting to recover approximate program logic — variables, functions, and control structures (if/else, loops) — in a form closer to source code than raw assembly.

> **Important:** Decompiled code is **not the original source code**. It is an approximation generated from compiled machine code. Variable names, comments, original types, and structural idioms may be lost or reconstructed imperfectly (e.g., a `for` loop in the original source might be decompiled as a `while` loop with equivalent logic, and variable names will typically be generic unless manually renamed by the analyst).

---

## 16. Ghidra Functions and Cross-References

Key concepts:

- **Functions** — discrete, named units of logic.
- **Function calls** — one function invoking another.
- **Callers / Callees** — who calls a function, and what a function calls.
- **Cross-references** — the general mechanism linking any address/symbol to where it's used elsewhere in the binary.
- **Strings referenced by functions** — a key technique for finding relevant code starting from an interesting string.

**Conceptual workflow:**

```text
Interesting String
       ↓
Find References
       ↓
Identify Function
       ↓
Analyze Caller
       ↓
Trace Control Flow
       ↓
Understand Behavior
```

This is a common and effective reverse-engineering technique: start from something concrete and readable (a string), use cross-references to find the code that uses it, and work outward from there — often faster than trying to read an entire binary's disassembly from the top.

---

## 17. Radare2

### What is Radare2?

**Radare2** is an open-source, command-line-oriented reverse-engineering framework supporting binary analysis, disassembly, debugging, and more.

Capabilities:

- Binary analysis
- Disassembly
- Debugging
- Function analysis
- Strings extraction
- Section/symbol inspection

Radare2 is powerful but has a steeper learning curve than a GUI tool like Ghidra, since nearly everything is driven through short, sometimes cryptic command sequences rather than menus and visual panels. In exchange, it can be faster and more scriptable for experienced users.

---

## 18. Radare2 Basic Workflow

```bash
r2 <SAMPLE-BINARY>
```

Once inside the Radare2 shell, important commands include:

| Command | Purpose |
|---|---|
| `i` | Display general binary information. |
| `ii` | Display imports. |
| `iz` | Display strings. |
| `afl` | Analyze and list functions. |
| `pdf` | Print disassembly of a function. |
| `s` | Seek — move the current analysis position to a given address/offset. |
| `px` | Hexdump the current location. |

Radare2 command syntax and available options can vary somewhat by version — this list reflects commonly used, foundational commands rather than an exhaustive reference.

---

## 19. Radare2 Analysis

Before interpreting functions, an analyst typically runs an analysis command so Radare2 can identify structure within the binary:

```text
aaa
```

`aaa` triggers Radare2's automatic analysis, attempting to identify functions, references, and other structural elements — analysis depth and accuracy depend heavily on the specific binary (its compiler, optimization level, and whether it's stripped of symbols).

Key elements worth reviewing after analysis:

- Functions
- Imports
- Strings
- Sections
- Entry point
- Cross-references
- Disassembly

Automated analysis is a starting point, not a guaranteed-correct picture — results should be reviewed and cross-checked, particularly for binaries that are obfuscated, stripped, or unusually structured.

---

## 20. GDB

### What is GDB?

The **GNU Debugger (GDB)** is a tool for debugging compiled programs — running them under controlled conditions and observing their execution in detail.

Capabilities:

- Runtime analysis
- Breakpoints (pausing execution at a specific point)
- Register inspection
- Memory inspection
- Stack inspection
- Execution-flow control (stepping through instructions)

GDB is primarily a **dynamic analysis/debugging tool** — its core purpose is observing and controlling a program while it runs, though it also supports some inspection capabilities (like disassembly) that complement static analysis.

---

## 21. GDB Basic Commands

| Command | Purpose |
|---|---|
| `file <SAMPLE-BINARY>` | Load a binary for debugging. |
| `run` | Start execution of the loaded program. |
| `start` | Run the program and automatically break at the entry point (or `main`). |
| `break <location>` | Set a breakpoint at a function or address. |
| `continue` | Resume execution after a breakpoint. |
| `next` | Step to the next line, stepping *over* function calls. |
| `step` | Step to the next line, stepping *into* function calls. |
| `info registers` | Display current register values. |
| `x` | Examine memory at a given address. |
| `disassemble` | Display the disassembly of the current function. |
| `bt` | Show a backtrace (call stack). |
| `quit` | Exit GDB. |

**Example:**

```text
break main
```

This sets a breakpoint at the `main` function — execution will pause immediately upon entering `main`, allowing the analyst to inspect the program's state before it does anything.

No register values or runtime output are fabricated in this document.

---

## 22. GDB Breakpoints

A **breakpoint** tells the debugger to pause execution at a specific point, allowing inspection before the program continues.

- **Function breakpoints** — pause when a named function is entered (e.g., `break main`).
- **Address breakpoints** — pause at a specific memory address.
- **Conditional breakpoints** (conceptually) — pause only when a specified condition is true, useful for skipping past uninteresting iterations of a loop.

```text
Start Program
     ↓
Set Breakpoint
     ↓
Run
     ↓
Execution Pauses
     ↓
Inspect Registers / Memory
     ↓
Step Through Instructions
     ↓
Observe Behavior
```

---

## 23. GDB Registers

Common x86-64 general-purpose and special registers:

| Register | Role |
|---|---|
| `RAX` | General-purpose; often holds return values. |
| `RBX` | General-purpose; often used as a callee-saved register. |
| `RCX` | General-purpose; often used for loop counters/4th argument. |
| `RDX` | General-purpose; often used as 3rd argument/data. |
| `RSI` | General-purpose; often used as 2nd argument/source index. |
| `RDI` | General-purpose; often used as 1st argument/destination index. |
| `RSP` | **Stack pointer** — points to the top of the current stack. |
| `RBP` | **Base/frame pointer** — often used as a stable reference within a function's stack frame. |
| `RIP` | **Instruction pointer** — points to the next instruction to execute. |

This document stays at a foundational level rather than covering every CPU flag or the full extended register set.

---

## 24. GDB Memory Inspection

The `x` (examine) command inspects memory at a given address.

```text
x/10gx <ADDRESS>
```
Examines memory starting at `<ADDRESS>`, showing 10 units in giant-word (8-byte) hexadecimal format.

```text
x/s <ADDRESS>
```
Examines memory at `<ADDRESS>` and interprets it as a null-terminated string.

`<ADDRESS>` is a placeholder — actual memory addresses are specific to a given execution and are never invented here.

Memory inspection helps correlate runtime state (what's actually in memory right now) with static analysis findings (what the disassembly suggested should be there), which is a key way dynamic analysis validates or refines hypotheses formed during static analysis.

---

## 25. Debugging vs. Disassembly

| Activity | Static Disassembly | Dynamic Debugging |
|---|---|---|
| Program execution | Not required | Required |
| Runtime state | Not available | Directly observable |
| Registers | Not applicable (no execution) | Directly inspectable |
| Memory | Only as laid out in the file | Live, actual runtime memory |
| Control flow | Inferred from static structure | Observed as it actually happens |
| Main tools | Objdump, Ghidra (static), Radare2 (static) | GDB, Radare2 (debug mode) |
| Typical purpose | Building structural/logical understanding | Confirming actual behavior under real conditions |

Analysts often use both because static analysis can be misleading (e.g., due to indirect jumps, obfuscation, or conditions not obvious from the disassembly alone), while dynamic analysis, though grounded in real behavior, only shows the specific path actually taken during that execution.

---

## 26. Ghidra vs. Radare2 vs. GDB

| Tool | Main Role | Interface | Static Analysis | Dynamic Analysis | Key Strength |
|---|---|---|---|---|---|
| Ghidra | Integrated reverse engineering | Graphical | Yes — strong | Limited (debugger integration exists but isn't its core focus) | Disassembly + decompiler + cross-referencing in one environment |
| Radare2 | Command-line binary analysis framework | Command-line | Yes — strong | Yes — supports debugging | Fast, scriptable, flexible analysis |
| GDB | Runtime debugger | Command-line | Limited (some inspection) | Yes — primary focus | Precise runtime control (breakpoints, stepping, register/memory inspection) |

GDB is primarily a debugger, while Ghidra and Radare2 provide broader reverse-engineering capabilities spanning both static structure and (for Radare2) dynamic debugging.

---

## 27. Objdump vs. Radare2 vs. Ghidra

| Capability | Objdump | Radare2 | Ghidra |
|---|---|---|---|
| Basic binary inspection | Yes | Yes | Yes |
| Disassembly | Yes | Yes | Yes |
| Function analysis | Limited (via symbol table) | Yes | Yes — strong |
| Decompiler | No | Limited (via plugins in some configurations) | Yes — strong |
| Debugging | No | Yes | Limited |
| GUI | No | No (command-line) | Yes |
| Ease of quick inspection | High — fast, minimal setup | Moderate — some learning curve | Lower — requires project setup |

A lightweight command-line tool like Objdump is often preferable for a quick check (e.g., "what sections does this binary have?"), while a full platform like Ghidra is better suited to deep, sustained analysis of a complex binary where cross-referencing and decompilation add real value.

---

## 28. Static Analysis Workflow

```text
Identify Binary
      ↓
File Type / Architecture
      ↓
Hash / Preserve Sample
      ↓
Strings
      ↓
ELF Headers / Sections
      ↓
Imports / Symbols
      ↓
Disassembly
      ↓
Functions
      ↓
Cross-References
      ↓
Decompiler
      ↓
Program Logic
      ↓
Document Findings
```

| Stage | Explanation |
|---|---|
| Identify Binary / File Type / Architecture | Confirm what's being analyzed before going further. |
| Hash / Preserve Sample | Establish an integrity baseline, especially important for suspicious samples. |
| Strings | Quick initial overview of embedded text. |
| ELF Headers / Sections | Understand overall file structure. |
| Imports / Symbols | Understand external dependencies and named references. |
| Disassembly | Detailed instruction-level review. |
| Functions | Identify discrete logic units. |
| Cross-References | Trace relationships between code/data. |
| Decompiler | Higher-level approximate logic (where available, e.g. Ghidra). |
| Program Logic | Synthesize an overall understanding. |
| Document Findings | Record the analysis process and conclusions. |

---

## 29. Dynamic Analysis Workflow

```text
Prepare Isolated Lab
      ↓
Preserve Sample
      ↓
Start Debugger
      ↓
Set Breakpoints
      ↓
Execute Program
      ↓
Observe Registers
      ↓
Inspect Memory
      ↓
Step Through Instructions
      ↓
Observe Runtime Behavior
      ↓
Compare With Static Analysis
      ↓
Document Findings
```

Dynamic analysis should be performed in an isolated environment (e.g., an isolated VM with no sensitive data and restricted networking) when dealing with unknown or potentially malicious binaries, since actually running the sample means accepting whatever it's designed to do — isolation limits the blast radius if the sample turns out to be harmful.

---

## 30. Static + Dynamic Analysis

Combining both approaches is more effective than either alone:

```text
Static Analysis
     ↓
Find Interesting Function
     ↓
Find Interesting String
     ↓
Trace Reference
     ↓
Set Breakpoint
     ↓
Dynamic Analysis
     ↓
Observe Runtime Behavior
     ↓
Validate Hypothesis
```

Static analysis is good at generating hypotheses — "this function looks like it might do X" — based on structure and readable clues like strings. Dynamic analysis can then validate (or disprove) those hypotheses by observing what actually happens at runtime, closing the loop between "what the code seems to say" and "what the program actually does."

---

## 31. Binary Inspection Checklist

### Identification
- File type
- Architecture
- Endianness
- Hash

### Structure
- ELF header
- Sections
- Segments
- Symbols

### Content
- Strings
- Imports
- Exports
- Libraries

### Code
- Entry point
- Functions
- Control flow
- Interesting references

### Runtime
- Registers
- Stack
- Memory
- System calls / behavior where appropriate

### Documentation
- Evidence
- Observations
- Hypothesis
- Conclusion

---

## 32. Practical Lab

The lab below uses a **simple, intentionally created sample program**, not a real malicious binary. These are safe lab procedures/templates — none are claimed as completed unless actual evidence exists.

### Task 1 — Identify the Binary

```bash
file <SAMPLE-BINARY>
```
The analyst learns the binary's general type (e.g., ELF), architecture, and whether it's dynamically or statically linked — a fast first step before any deeper analysis.



### Task 2 — Extract Strings

```bash
strings <SAMPLE-BINARY>
```
Purpose: quickly surface any embedded readable text as an initial reconnaissance step.



### Task 3 — Inspect ELF Structure

```bash
objdump -h <SAMPLE-BINARY>
```
Purpose: review the binary's section layout before diving into disassembly.



### Task 4 — Disassemble

```bash
objdump -d <SAMPLE-BINARY>
```
Purpose: obtain a full assembly-level view of the executable sections for detailed review.



### Task 5 — Analyze With Ghidra

Import the sample binary into a Ghidra project, run automated analysis, then review: functions identified, embedded strings, disassembly, and the decompiler's pseudocode output.



### Task 6 — Analyze With Radare2

```text
r2 <SAMPLE-BINARY>
i
ii
iz
afl
```
Purpose: get binary information, imports, strings, and a function listing through Radare2's command-line workflow.



### Task 7 — Debug With GDB

```text
gdb <SAMPLE-BINARY>
break main
run
info registers
disassemble
continue
```
Purpose: load the binary, pause at `main`, inspect the initial register state, review the disassembly of the current function, and then resume execution.



**Status note:** All seven tasks above are lab procedures and templates for Day 26. No specific sample binary has yet been run through this full workflow, and no output from any of these tools is claimed as an actual completed result — all remain **Pending / Optional Practice**.

---



## 33 Command Reference

## Strings

```bash
strings <SAMPLE-BINARY>
```
Extracts printable text from the binary.

## Objdump

```bash
objdump -f <SAMPLE-BINARY>
objdump -h <SAMPLE-BINARY>
objdump -t <SAMPLE-BINARY>
objdump -d <SAMPLE-BINARY>
```
File header summary, section headers, symbol table, and disassembly, respectively.

## Radare2

```bash
r2 <SAMPLE-BINARY>
```
Then, inside the Radare2 shell:
```text
i     # binary information
ii    # imports
iz    # strings
afl   # list functions (after analysis)
pdf   # print disassembly of a function
s     # seek to an address/offset
px    # hexdump
```

## GDB

```text
gdb <SAMPLE-BINARY>
file <SAMPLE-BINARY>
run
start
break main
continue
next
step
info registers
disassemble
bt
quit
```
Loads and runs the binary under the debugger, sets a breakpoint at `main`, and provides commands to inspect registers, disassembly, and the call stack while stepping through execution.

---

## 34. Common Beginner Mistakes

- Assuming decompiled code is original source code.
- Assuming strings reveal the complete behavior of a program.
- Trusting automated function detection (Ghidra/Radare2) blindly without review.
- Confusing addresses between static (file-relative) and runtime (loaded/relocated) contexts.
- Not identifying the binary's architecture before analysis.
- Debugging unknown/suspicious binaries directly on the host machine instead of an isolated environment.
- Ignoring symbols/imports, which often provide valuable quick context.
- Assuming a suspicious-looking function is automatically malicious.
- Misinterpreting assembly instructions due to unfamiliarity with the architecture.
- Confusing static analysis findings with confirmed dynamic behavior.
- Modifying a binary before analyzing it (even unintentionally, e.g., via certain tool operations).
- Failing to hash/preserve the original sample before analysis.
- Treating a single indicator (one string, one function) as conclusive evidence of behavior.

---

## 35. Limitations

### Strings
- Only reveals printable sequences.
- May surface irrelevant or unused data.
- Does not explain program logic on its own.

### Objdump
- Primarily command-line inspection with limited high-level abstraction.
- Disassembly output can be difficult to interpret without further tooling.

### Ghidra
- Automated analysis can make mistakes, especially on unusual or obfuscated binaries.
- Decompiled code is an approximation, not guaranteed accurate or complete.
- Obfuscation techniques can significantly complicate analysis.

### Radare2
- Steeper learning curve due to its command-line-driven interface.
- Analysis results depend on configuration and the specific binary's characteristics.

### GDB
- Requires some understanding of the underlying architecture/assembly to be useful.
- Primarily runtime/debugging-focused — not a substitute for static structural analysis.
- Debugging behavior (e.g., timing, certain checks) can sometimes differ from a program's normal, non-debugged execution.

---

## 36. Safe Handling of Unknown Binaries

- Use an **isolated VM** for analysis, especially for unknown or suspicious samples.
- Take a **snapshot** before analysis so the environment can be reverted.
- Avoid storing or exposing **sensitive host data** within the analysis environment.
- Use **restricted networking** where appropriate, to prevent unintended outbound activity.
- Verify **sample integrity** (hashing) before and during analysis.
- Avoid executing unknown binaries directly on the host machine.
- Maintain copies of the **original, unmodified sample**.
- Record hashes as part of standard documentation.

Static analysis can often be performed safely before any execution is attempted, making it a sensible first step for unfamiliar or suspicious binaries.

---

## 37. Defensive / Malware-Analysis Perspective

Binary analysis supports:

- Malware analysis
- Vulnerability research
- Incident response
- Threat intelligence
- Software auditing
- Supply-chain security
- Detection engineering

Analysts may look for indicators such as:

- Suspicious functions
- Embedded URLs
- Hardcoded configuration
- API calls suggestive of certain capabilities (e.g., network or file operations)
- Persistence-related logic
- Network functionality

As with earlier sections, any single indicator requires validation — a suspicious string, function name, or API call is a lead for further investigation, not proof of malicious behavior on its own.

---

## 38. Connection to Previous Days

```text
Day 24
Digital Forensics
      ↓
Disk / Memory / Metadata

Day 25
File Recovery / Firmware Analysis
      ↓
Raw Data / Embedded Structures

Day 26
Binary Analysis
      ↓
Executable Structure
      ↓
Static Analysis
      ↓
Dynamic Analysis
      ↓
Reverse Engineering
```

Day 24 established the foundations of forensic evidence analysis (disk, memory, metadata). Day 25 extended that into recovering raw data and analyzing firmware structure. Day 26 builds directly on both by providing the ability to investigate the executable/binary artifacts that forensic and firmware investigations frequently surface — turning a recovered or extracted binary from an opaque file into something that can be structurally and behaviorally understood.

---

## 39. What I Learned

Day 26 gave me a foundational understanding of what it means to analyze a compiled binary without its source code — the distinction between static analysis (examining structure without running the program) and dynamic analysis (observing actual runtime behavior) became much clearer through working through Strings, Objdump, Ghidra, Radare2, and GDB individually.

I developed a foundational understanding of ELF structure — sections, segments, symbols — and how tools like Objdump and Ghidra expose that structure in different ways, from a quick command-line summary to a full graphical analysis environment. Working through Ghidra's disassembler and decompiler helped me understand that decompiled output is an approximation of program logic, not the original source code, and that variable names and structure are often reconstructed rather than recovered exactly.

I also gained a foundational understanding of debugging with GDB — how breakpoints, registers, and memory inspection let an analyst observe a program's actual behavior rather than just inferring it from static structure, and why combining static and dynamic analysis (using static findings to form hypotheses, then validating them dynamically) is more powerful than relying on either approach alone.

I did not gain advanced malware-analysis expertise, expert-level reverse-engineering skill, or exploit-development ability from this day, and I'm not yet able to fully reverse engineer arbitrary or obfuscated malware — my understanding remains foundational, and the practical lab tasks (Section 32) remain to be completed with an actual sample binary.

---

## 40. Key Takeaways

- Static analysis does not require program execution.
- Dynamic analysis observes actual runtime behavior.
- Binary inspection provides structural information about executables (headers, sections, symbols).
- Strings can provide useful clues but are not proof of behavior.
- Objdump provides lightweight, fast binary inspection and disassembly.
- Ghidra provides an integrated reverse-engineering environment combining disassembly, decompilation, and cross-referencing.
- Radare2 provides powerful, flexible command-line binary analysis.
- GDB is primarily a runtime debugger, focused on execution control and inspection.
- Disassembly represents machine instructions in assembly form.
- Decompiled output is an approximation of program logic, not original source code.
- Static and dynamic analysis complement one another — static generates hypotheses, dynamic validates them.
- Binary architecture must be identified before meaningful analysis can proceed.
- Unknown or suspicious binaries should be handled in isolated environments, not on a normal host machine.
- Findings from any single tool or indicator should be validated and clearly documented, not treated as conclusive on their own.

---

## 41. Further Practice

These are **further practice** items, not completed work.

### Exercise 1
Compile a simple C program and inspect it with Strings, Objdump, Ghidra, and Radare2, comparing what each tool surfaces.

### Exercise 2
Compare the same function across its source code, its raw assembly (via Objdump), and Ghidra's decompiler output, to see how each representation differs.

### Exercise 3
Debug the sample program with GDB, observing breakpoints, register values, the stack, and function calls as execution proceeds.

### Exercise 4
Modify the source program slightly and compare how the resulting binary (disassembly/structure) changes.

### Exercise 5
Create a simple program containing strings, conditional branches, and multiple functions, then practice fully reverse engineering it using the tools from this day.

---

## 42. Day 26 Final Summary

```text
Strings
   ↓
Quick Content Inspection

Objdump
   ↓
Binary Structure + Disassembly

Ghidra
   ↓
Integrated Static Reverse Engineering

Radare2
   ↓
Command-Line Binary Analysis

GDB
   ↓
Runtime Debugging
```

```text
Static Analysis
       +
Dynamic Analysis
       ↓
Binary Understanding
       ↓
Reverse Engineering
```

No single tool from Day 26 provides the complete picture on its own. Strings offers a fast first look, Objdump provides lightweight structural inspection, Ghidra and Radare2 offer deeper static (and, for Radare2, dynamic) analysis capabilities, and GDB provides precise runtime observation and control. Effective binary analysis comes from correlating findings across these tools — moving between static structure and dynamic behavior — rather than relying on any single source of evidence.