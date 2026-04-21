# 02 — How Systems Work

>Your computer is layers of software, each trusting the one below it. Understanding these layers is essential for RE.

---

## The Big Picture

Think of your computer like an onion layers on layers, each doing a different job.

```
┌─────────────────────────────────────────┐
│         YOUR APPLICATIONS               │  ← Browser, games, Ghidra...
├─────────────────────────────────────────┤
│         OPERATING SYSTEM (OS)           │  ← Windows, Linux, macOS
├─────────────────────────────────────────┤
│              KERNEL                     │  ← The core of the OS
├─────────────────────────────────────────┤
│           HARDWARE DRIVERS              │  ← Talk to physical devices
├─────────────────────────────────────────┤
│      HARDWARE (CPU, RAM, Disk...)       │  ← The physical machine
└─────────────────────────────────────────┘
```

Each layer only talks to the layer directly next to it. Your browser doesn't talk directly to the CPU it goes through the OS, which goes through the kernel, which manages the hardware.


---

## The Hardware Layer

### CPU (Central Processing Unit)
The brain of your computer. It executes instructions billions per second.

The CPU only understands **machine code**: a sequence of numbers (binary) that represent specific operations. Everything you write in any programming language eventually becomes these numbers.

**Registers** are tiny, ultra-fast storage locations *inside* the CPU. Think of them like sticky notes on your desk very fast to access, but very few of them.

Common registers you'll see in reverse engineering:

| Register (x86-64) | Nickname | Common Use |
|-------------------|----------|------------|
| `rax` | Accumulator | Return values from functions |
| `rbx` | Base | General purpose |
| `rsp` | Stack Pointer | Points to top of the stack |
| `rbp` | Base Pointer | Points to bottom of current stack frame |
| `rip` | Instruction Pointer | Points to the *next* instruction to execute |

> The `rip` register is crucial in RE it always tells you "where are we in the program right now?"

### RAM (Memory)
Temporary storage while programs are running. When you open an app, it gets loaded from your disk into RAM. It's much faster than disk but disappears when power is off.

Everything in memory has an **address** a number that tells the CPU exactly where something is stored.

### Disk
Long-term storage. Executables live here until they're launched.

![The memory hirarchy](pictures/memory.png)

---

## The Kernel, The Most Powerful Software

The kernel is the core of any operating system. It's the most trusted, most privileged piece of software on your machine.

### What Does the Kernel Do?

- **Manages processes** starts, stops, and schedules running programs
- **Manages memory** decides which program gets which chunk of RAM
- **Manages hardware** controls access to disk, network, USB, etc.
- **Enforces security** programs can't just access any memory they want

### Kernel Space vs User Space

This is one of the most important concepts in RE:

```
┌─────────────────────────────────────────┐
│            USER SPACE                   │
│   Your apps live here. Limited power.   │
│   A crash here = just that app dies.    │
├─────────────────────────────────────────┤
│           KERNEL SPACE                  │
│   The OS core. Full power.              │
│   A crash here = entire system dies     │
│   (Blue Screen of Death / kernel panic) │
└─────────────────────────────────────────┘
```

Normal programs run in **user space**. They have limited privileges they can't directly touch hardware or other programs' memory.

When a program needs to do something powerful (like read a file or send network traffic), it makes a **system call (syscall)** a special request to the kernel saying "hey, I need you to do this for me."

![user and kernel spaces](pictures/user-vs-kernel-space.png)

### Why This Matters in RE

Malware often tries to:
- **Escape user space** into kernel space (called a "kernel exploit" or "privilege escalation")
- **Hook system calls** to intercept what other programs do
- **Load a driver** (kernel-level code) to hide itself from antivirus

---

## The Operating System

The OS sits between you and the kernel, making the computer usable.

### Key OS Concepts for RE

**Processes**
Every running program is a "process". Each process has its own isolated chunk of memory your browser can't read your game's memory (in theory).

```
Process: firefox.exe
  ├── Memory: 0x00400000 - 0x7fffffffffff (its own private address space)
  ├── Threads: 12 running threads
  └── Open files, network connections, handles...
```

**Threads**
A single process can do multiple things at once using threads. Think of a thread as one "worker" inside the process. Multiple threads share the same memory (not the stack).

When a thread (the unit of execution) running inside a process generates a virtual address, the Memory Management Unit (MMU) uses the process's page table to translate it into a physical address. 
This means every memory access by any thread within a process is translated using the same page table, which defines that process's private virtual address space  

**File System**  

How the OS organizes files on disk. In RE, you'll often look at:  

- Where a malware drops files (`%APPDATA%`, `/tmp/`)
- Config files and logs it reads/writes
- Registry keys it modifies (Windows)

**Windows Registry**  

The Windows Registry is the operating system's central configuration database, acting like a massive, hierarchical filing system for all system and application settings.  

Think of it as Windows' DNA. Instead of scattered configuration files, everything—hardware drivers, user preferences, installed software, and system policies—is stored in one organized, tree-like structure. The top-level folders are called hives (like HKEY_LOCAL_MACHINE and HKEY_CURRENT_USER), which contain subkeys (like folders) and values (like files with actual data).  

When you install a program or change a setting (like your desktop background), Windows writes that information into the Registry. On the next boot, it reads this "blueprint" to know exactly how to configure itself, which programs to start, and what drivers to load.  

Malware exploits the Run keys because they are part of this automatic startup blueprint. By adding its own entry to HKEY_CURRENT_USER\...\Run, the malware tricks Windows into loading it every time the user logs in, just like a legitimate application would.  

![windows registry keys](pictures/windows-reg-keys.png)


In contrast, Linux uses a decentralized approach with plain text configuration files, primarily stored in the /etc directory for system-wide settings and in hidden files (like ~/.config/) in a user's home directory for individual preferences.  This means Linux settings are human-readable and can be edited with any text editor, offering flexibility.


---

## System Calls How Programs Talk to the OS

When a program wants to read a file, it doesn't do it directly. It calls a **system call**.

Here's the flow:

```
Your Program
    │
    │  calls ReadFile() / fopen() / read()
    ▼
Standard Library (e.g., glibc / Windows API)
    │
    │  makes syscall (e.g., syscall #0 = read on Linux)
    ▼
Kernel
    │
    │  actually reads the disk
    ▼
Hardware (Disk)
```

![Types of system calls](pictures/system-calls.png)

**Why care about syscalls in RE?**
By monitoring which syscalls a program makes, you can understand exactly what it's doing without even looking at its code. Tools like `strace` (Linux) and API Monitor (Windows) do exactly this.

---

## Rings of Privilege

![Kernel Rings Diagram](pictures/kernel-rings.png)

CPUs have different levels of privilege called "rings":

```
Ring 0  →  Kernel (full hardware access)
Ring 1  →  (rarely used)
Ring 2  →  (rarely used)
Ring 3  →  User applications (restricted)
```

Most modern OSes only use Ring 0 (kernel) and Ring 3 (user). Trying to execute a privileged instruction from Ring 3 causes a fault the kernel catches it and usually kills the offending process.

Malware that achieves Ring 0 execution (rootkits) is the most dangerous kind it can hide from everything running in Ring 3, including antivirus.

---

## Putting It All Together

When you double-click a `.exe`:

1. The OS reads the file from disk
2. The kernel creates a new **process** and allocates memory for it
3. The OS loads the executable's code and data into that memory
4. The CPU starts executing the program's instructions
5. Whenever the program needs something from the OS (file, network, etc.) it makes a **syscall**
6. When the program exits, the kernel frees all its resources

Every single step of this is something a reverse engineer can observe and analyze.

---

## ✅ What You Should Know After This Chapter

- [ ] The layered architecture: hardware → kernel → OS → applications
- [ ] What a CPU, registers, and RAM are
- [ ] The difference between kernel space and user space
- [ ] What a system call (syscall) is and why it matters
- [ ] What processes and threads are
- [ ] Why the Windows Registry matters for malware persistence

---

← [Previous: What is RE?](01-What-is-RE.md) | [Next: Memory Layout →](03-Memory-layout.md) 