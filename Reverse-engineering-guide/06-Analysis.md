# 06 — Static & Dynamic Analysis

> Static analysis = examining the binary without running it. Dynamic analysis = running it and watching what it does. Real-world RE combines both.

---

## Two Paths, One Goal

When you receive an unknown binary, you have two complementary approaches:

```
Unknown Binary
       │
       ├─── STATIC ANALYSIS ──→ What COULD this do?
       │         (don't run it)
       │
       └─── DYNAMIC ANALYSIS ──→ What DOES this do?
                 (run it safely)
```

You'll usually start with static (safer, no risk of infection), then confirm findings with dynamic.

---

# PART 1  Static Analysis

## Level 0: The Very First Things to Check

Before opening any tool, you can learn a lot from basic metadata.

### File Type Identification

Never trust the file extension. A file named `invoice.pdf` might actually be `invoice.pdf.exe`.

**Tool: `file` (Linux)**
```bash
$ file suspicious_file
suspicious_file: PE32+ executable (GUI) x86-64, for MS Windows
```

**Tool: Detect It Easy (DIE)**
A GUI tool that identifies file types, compilers used, and **packers**.

> If DIE says the file is packed (e.g., "UPX", "MPRESS"), the real code is hidden. You'll need to unpack it first.

### Hashing
Generate a hash of the file and search it online:
```bash
$ md5sum malware.exe
$ sha256sum malware.exe
```
Paste the SHA256 into [VirusTotal](https://www.virustotal.com) if it's known malware, you get instant analysis. If it's brand new (0 detections), be *more* suspicious, not less.

### Entropy
Entropy measures how random the data is. Packed/encrypted sections have very high entropy (~8.0).
```bash
# Using 'ent' tool on Linux:
$ ent malware.exe
Entropy = 7.93 bits per byte   ← very high → probably packed
```

---

## Level 1: Strings Analysis

One of the fastest ways to understand a binary. Run `strings` to extract all printable text:

```bash
$ strings malware.exe | less
```

You're looking for:
- URLs or IP addresses → `http://192.168.1.1/update`, `https://evil-c2.xyz/beacon`
- File paths → `C:\Users\%USERNAME%\AppData\Roaming\`
- Registry keys → `SOFTWARE\Microsoft\Windows\CurrentVersion\Run`
- Error messages → often left in by the developer, reveal internal logic
- Suspicious Windows API names → `CreateRemoteThread`, `VirtualAllocEx`
- Encoded strings → Base64 blobs, XOR-encoded data

**Example real-world findings:**
```
http://185.220.101.12/gate.php      ← C2 server
Mozilla/5.0 (Windows NT 10.0)       ← fake User-Agent for HTTP requests  
cmd.exe /c net user backdoor P@ss   ← command to create a hidden account
HKCU\Software\Microsoft\Windows\...← registry persistence
```

---

## Level 2: PE Structure Analysis

**Tool: PEStudio** (Windows) or **pefile** (Python library)

### Import Table — The Most Valuable Static Artifact

The import table lists every external function the binary calls. Group them by what they suggest:

```
 Network activity:
   WSAStartup, connect, send, recv, HttpSendRequest

 File system:
   CreateFile, WriteFile, DeleteFile, CopyFile

 Process injection:
   VirtualAllocEx, WriteProcessMemory, CreateRemoteThread, NtUnmapViewOfSection

 Credential theft:
   CryptAcquireContext, LsaOpenPolicy, SamOpenDatabase

 Anti-analysis:
   IsDebuggerPresent, CheckRemoteDebuggerPresent, GetTickCount
```

### Sections
```
.text   → code (should be executable)
.data   → initialized global variables
.rdata  → read-only data (strings, constants)
.rsrc   → resources (icons, embedded files, configs)
```
A section that is both writable AND executable is a red flag — normal code sections are execute-only.

---

## Level 3: Disassembly with Ghidra

[Ghidra](https://ghidra-sre.org/) is a free, open-source reverse engineering suite from the NSA. It's incredibly powerful.

### Getting Started with Ghidra

1. **Install**: Download from ghidra-sre.org, requires Java 17+
2. **Create project**: File → New Project → Non-Shared Project
3. **Import file**: Drag your binary in, or File → Import File
4. **Auto-analyze**: When prompted, click "Yes" to auto-analyze. Wait for it.
5. **CodeBrowser**: Double-click the file to open it

### The Ghidra Interface

```
┌─────────────────────────────────────────────────────┐
│  Program Trees  │  Listing (Disassembly)  │  Decompiler  │
│  - Shows all    │  - Assembly view        │  - Pseudo-C  │
│    sections     │  - Click to navigate    │  - Much more │
│                 │  - Add comments!        │    readable  │
├─────────────────────────────────────────────────────┤
│  Symbol Tree    │  References / Cross-references    │
│  - Functions    │  - Who calls this function?       │
│  - Labels       │  - Where is this string used?     │
└─────────────────────────────────────────────────────┘
```

### Your First Ghidra Workflow

```
1. Open the binary in Ghidra
2. Check the Symbol Tree → Functions
   └── Look for functions with suspicious names
       └── If names are stripped, look for suspiciously-named or unnamed ones

3. Find interesting strings:
   Window → Defined Strings
   └── Double-click a URL or suspicious string
   └── Right-click → References → Show References
       └── Navigate to the function that uses it

4. Read the decompiler output:
   └── Rename variables (press L) to track data flow
   └── Add comments (press ; or End key) to document your findings

5. Look at cross-references:
   └── Right-click any function → References → Show References to
   └── Understand who calls what, and when
```

---

# PART 2  Dynamic Analysis

##  First: Your Safe Lab Setup

>  **Never skip this step.** Run malware only in an isolated VM.

**Recommended free setup:**
1. [VirtualBox](https://www.virtualbox.org/) (free) + Windows 10 VM
   - Microsoft offers free Windows 10 VMs at [developer.microsoft.com](https://developer.microsoft.com/en-us/windows/downloads/virtual-machines/)
2. **Network**: Set VM adapter to "Host-only" — no internet access
3. **Snapshot**: Take a clean snapshot before running anything
4. **Restore**: After analysis, restore the snapshot

---

## Level 1: Behavior Monitoring (No Debugger Needed)

Before even opening a debugger, just run the file and watch what happens.

### Sysinternals Suite (Windows) — Free, Essential

**Process Monitor (Procmon)**
Logs *everything* a process does: file reads/writes, registry changes, network connections.

```
Filter by:
  Process Name → contains → malware.exe

Watch for:
  Registry: HKCU\Software\...\Run  (persistence)
  File: C:\Users\...\AppData\      (dropping files)
  Network: 192.168.x.x:443         (connecting out)
```

**Process Hacker / Process Explorer**
Advanced Task Manager. Shows:
- All running processes with their full paths
- Loaded DLLs (libraries) per process
- Network connections per process
- Injected code (highlighted in different color)

**Autoruns**
Shows *everything* that runs on startup. After running malware, check Autoruns for new entries.

---

## Level 2: Debugging

Debugging = running a program under your control, step by step.

### Core Debugger Concepts

**Breakpoint**: Tells the debugger "pause execution when you reach this address"
```
Software breakpoint: replaces one byte of code with 0xCC (INT3 instruction)
Hardware breakpoint: uses CPU debug registers (stealthier — malware won't notice)
```

**Stepping**:
- `Step Into` (F7/s): execute one instruction, enter function calls
- `Step Over` (F8/n): execute one instruction, skip over function calls
- `Continue` (F9/c): run until next breakpoint

**Inspecting State**:
- Registers window: current values of rax, rbx, rsp, rip...
- Memory window: dump any address to see its raw bytes
- Stack window: what's currently on the stack

---

## GDB — Linux Debugger

GDB (GNU Debugger) is the standard debugger on Linux.

### Basic GDB Workflow

```bash
$ gdb ./binary

(gdb) info functions          # list all functions
(gdb) disas main              # disassemble the main function
(gdb) break main              # set breakpoint at main
(gdb) run                     # start execution
(gdb) next                    # step over (source level)
(gdb) nexti                   # step over (instruction level)
(gdb) stepi                   # step into (instruction level)
(gdb) info registers          # show all register values
(gdb) x/10wx $rsp             # examine 10 words at the stack pointer
(gdb) x/s 0x404010            # show string at address 0x404010
(gdb) p $rax                  # print value of rax
(gdb) continue                # continue to next breakpoint
```

### pwndbg — GDB Enhancement Plugin

Install [pwndbg](https://github.com/pwndbg/pwndbg) for a much nicer experience:
```bash
$ git clone https://github.com/pwndbg/pwndbg
$ cd pwndbg
$ ./setup.sh
```
It gives you a beautiful context view showing registers, stack, and disassembly all at once on every step.

---

## x64dbg — Windows Debugger

[x64dbg](https://x64dbg.com/) is the go-to free debugger for Windows. Very user-friendly.

### Basic x64dbg Workflow

1. **Open**: File → Open → select your binary
2. **Auto-pause**: x64dbg pauses at the entry point automatically
3. **Find main**: look in the CPU tab, navigate to `main` or the first interesting function
4. **Breakpoints**: 
   - `F2` on any instruction to set/remove a breakpoint
   - Right-click → Breakpoint → Set breakpoint on API for API breakpoints (e.g., break when `CreateFile` is called)
5. **Step**: `F8` (step over), `F7` (step into), `F9` (run)
6. **Watch imports**: Click on a call to `URLDownloadToFile` → follow it to see what URL

### Setting API Breakpoints

The most powerful technique for behavior analysis:
```
Right-click → Breakpoint → Set Breakpoint on all References to...

Or use the Command window:
bp CreateFileW      ← break every time CreateFile is called
bp RegSetValueExW   ← break every time a registry key is written
bp WSASend          ← break every time data is sent over network
```

---

## Network Analysis

**Wireshark** — capture all network traffic from your VM:
1. Start capture before running the malware
2. Filter by process or protocol:
   ```
   http                          # HTTP traffic
   dns                           # DNS lookups
   ip.addr == 192.168.1.105      # traffic to/from specific IP
   tcp.port == 4444              # common reverse shell port
   ```
3. Follow TCP/HTTP streams to see full request/response

**FakeNet-NG** — simulates internet in your VM:
- Responds to DNS lookups with a fake IP
- Runs a fake HTTP/HTTPS server
- Lets you capture C2 communication even without real internet

---

## Putting It Together — An Analysis Report

After analysis, document your findings:

```markdown
## Sample: suspicious.exe
**SHA256**: abc123...
**Type**: PE32+ executable, x64
**Packer**: None detected

### Static Findings
- Imports CreateRemoteThread, VirtualAllocEx → process injection
- String found: "http://185.220.101.12/c2.php"
- String found: "HKCU\Software\Microsoft\Windows\CurrentVersion\Run"

### Dynamic Findings
- Drops file: C:\Users\%USER%\AppData\Roaming\svchost32.exe
- Adds registry key: Run → "WindowsUpdate" → path to dropped file
- Connects to: 185.220.101.12:443 via HTTPS
- Sends HTTP POST with base64-encoded victim data

### Classification
Trojan dropper with C2 communication and registry persistence.
```

---

## ✅ What You Should Know After This Chapter

- [ ] Static analysis workflow: strings → imports → disassembly (Ghidra)
- [ ] What the import table tells you about malware behavior
- [ ] How to set up a safe dynamic analysis environment
- [ ] Behavior monitoring with Procmon and Process Hacker
- [ ] Basic GDB usage for Linux binaries
- [ ] Basic x64dbg usage for Windows binaries
- [ ] Network analysis with Wireshark

---

## 🎉 You've Completed the Introduction!

You now have a solid foundation in reverse engineering. Here's where to go next:

| Goal | Resource |
|------|----------|
| Practice RE challenges | [crackmes.one](https://crackmes.one), [reversing.kr](http://reversing.kr) |
| CTF competitions | [picoCTF](https://picoctf.org/), [pwn.college](https://pwn.college/) |
| Malware analysis | [Malware Unicorn workshops](https://malwareunicorn.org/) |
| Binary exploitation | [pwn.college](https://pwn.college/), [exploit.education](https://exploit.education/) |
| Deep-dive RE course | [OpenSecurityTraining2](https://ost2.fyi/) |
| Read malware reports | [Mandiant](https://www.mandiant.com/resources/blog), [SentinelOne Labs](https://www.sentinelone.com/labs/) |

---

← [Previous: Introduction to Malware](04-Intro-to-malware.md)