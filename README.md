🔍 Introduction to Reverse Engineering

> A beginner-friendly guide for anyone stepping into cybersecurity and wanting to understand how things work *under the hood*.

No prior experience needed. We start from zero.

---

## 📚 Table of Contents

| # | Chapter | What You'll Learn |
|---|---------|-------------------|
| 01 | [What is Reverse Engineering?](Reverse-engineering-guide/01-What-is-RE.md)| Definition, real-world uses, and why it matters |
| 02 | [How Systems Work](./02-how-systems-work/README.md) | Your PC, the OS, the kernel, and how they talk to each other |
| 03 | [Memory Layout](./06-memory-layout/README.md) | Stack, heap, registers, and how programs live in memory |
| 04 | [From Code to Machine](./03-compilation/README.md) | How C code becomes 1s and 0s your CPU understands |
| 05 | [Introduction to Malware](./04-intro-to-malware/README.md) | What malware is, types, and how it behaves |
| 06 | [Static & Dynamic Analysis](./05-analysis/README.md) | Hands-on analysis with Ghidra, GDB, and more |


---

## 🎯 Who Is This For?

- You're new to cybersecurity and curious about how things *really* work
- You've heard terms like "disassembly", "malware analysis", or "binary exploitation" and want to understand them
- You want a structured, no-BS starting point before diving into CTFs or malware analysis

---

## 🧰 Tools We Use

| Tool | What It's For | Free? |
|------|--------------|-------|
| [Ghidra](https://ghidra-sre.org/) | Disassembler / decompiler (NSA open-source) | ✅ |
| [GDB](https://www.sourceware.org/gdb/) | Linux debugger | ✅ |
| [x64dbg](https://x64dbg.com/) | Windows debugger | ✅ |
| [Detect It Easy (DIE)](https://github.com/horsicq/Detect-It-Easy) | File type & packer detection | ✅ |
| [PEStudio](https://www.winitor.com/) | Static PE analysis | ✅ |
| [strings](https://linux.die.net/man/1/strings) | Extract readable text from binaries | ✅ |
| [Wireshark](https://www.wireshark.org/) | Network traffic analysis | ✅ |

---

## 🗺️ Learning Path

```
[ 01 What is RE ] → [ 02 How Systems Work ] → [ 03 Memory Layout ]
                                                       ↓
[ 06 Compilation ] ← [ 05 Malware intro ] ← [ 04 Analysis ]
```

Follow the chapters in order each one builds on the last.

---

## ⚠️ Ethics & Legal Note

Reverse engineering is a powerful skill. With great power comes great responsibility.

- **Always** work on software you own or have explicit permission to analyze
- **Never** use these skills to access systems without authorization
- Many countries have laws around reverse engineering know yours
- Use isolated VMs when analyzing malware **never** run unknown binaries on your main machine

---

## 📖 Further Reading & Resources

- [Malware Unicorn Workshops](https://malwareunicorn.org/) — free malware analysis workshops
- [OpenSecurityTraining2](https://ost2.fyi/) — deep-dive RE courses
- [pwn.college](https://pwn.college/) — hands-on binary exploitation
- [crackmes.one](https://crackmes.one/) — practice reverse engineering challenges
- [VirusTotal](https://www.virustotal.com/) — scan and analyze suspicious files online
- *The IDA Pro Book* by Chris Eagle
- *Practical Malware Analysis* by Sikorski & Honig