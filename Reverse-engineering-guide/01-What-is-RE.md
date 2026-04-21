# 01 — What is Reverse Engineering?

>Reverse engineering is the process of taking something apart to understand how it works without having the original instructions.

---

##  The Simple Explanation

Imagine you find a mysterious machine with no manual. You press some buttons, observe what happens, open it up, look at the parts and eventually figure out exactly how it works.

That's reverse engineering.

In software, the "machine" is a program, and we're analyzing it to understand what it does, how it does it, and sometimes *why* it does suspicious things.

---

##  Forward Engineering vs Reverse Engineering

| Forward Engineering | Reverse Engineering |
|--------------------|---------------------|
| Start with an idea → build software | Start with software → figure out the idea |
| You have the source code | You often only have the compiled binary |
| You know the "recipe" | You're trying to rediscover the recipe |

**Example**: 
- A developer writes code in Python or C → that's forward engineering
- A security researcher takes a compiled `.exe` and figures out what it does → that's reverse engineering

---

##  Real-World Uses

Reverse engineering isn't just for hackers. Here's where it legitimately shows up every day:

###  Malware Analysis
Security researchers take malicious software apart to understand:
- What does it steal?
- How does it spread?
- What servers does it talk to?
- How can we detect and stop it?

###  Vulnerability Research
Finding security bugs in software *before* attackers do. Companies like Google's Project Zero reverse engineer software to find flaws and responsibly disclose them.

###  Interoperability
Sometimes you need two programs to talk to each other, but one is closed-source. Reverse engineering helps understand the protocol so you can build compatible software. (This is how projects like [Samba](https://www.samba.org/) reverse-engineered Windows file sharing.)

###  Game Modding & Preservation
Gamers reverse engineer old games to:
- Create mods
- Preserve games no longer sold
- Enable multiplayer on abandoned servers

###  Hardware & Embedded Systems
Engineers reverse engineer chips, firmware, and devices to:
- Audit security cameras, routers, and IoT devices
- Fix hardware with no documentation
- Understand proprietary protocols

###  Legal Use in Cybersecurity
- **Penetration testers** use RE to find vulnerabilities in systems they're hired to test
- **Incident responders** use it to understand what attackers left behind
- **CTF players** use it as a sport/skill-building exercise

---

##  Key Concepts You'll Encounter

### Binary / Executable
When code is compiled, the human-readable source code is turned into a **binary** a file made of machine instructions that only the CPU understands. You'll work with these a lot.

### Disassembly
Taking a binary and converting its machine code back into **assembly language** a low-level, human-readable representation of CPU instructions. Tools like Ghidra and IDA Pro do this.

```
Machine code:    55 48 89 E5 89 7D FC
Disassembly:     push rbp
                 mov  rbp, rsp
                 mov  DWORD PTR [rbp-0x4], edi
```

### Decompilation
Going one step further than disassembly trying to reconstruct something closer to the original high-level code (like C). It's never perfect, but it's much more readable.

### Debugging
Running a program step-by-step, pausing it at certain points (called **breakpoints**), and inspecting what's in memory. This is dynamic analysis.

---

##  The Mindset

Good reverse engineers are:
- **Patient** : you rarely understand everything at once
- **Curious** : always asking "but *why* does it do that?"
- **Systematic** : taking notes, forming hypotheses, testing them
- **Creative** : sometimes the answer requires thinking sideways

> "Reverse engineering is 20% tools and 80% thinking." every RE guy, ever

---

## ✅ What You Should Know After This Chapter

- [ ] Reverse engineering = understanding software/hardware without original source
- [ ] It has many legitimate uses beyond "hacking"
- [ ] Key terms: binary, disassembly, decompilation, debugging
- [ ] The main tools we'll use throughout this guide

---

**Next Chapter →** [How Systems Work](02-How-systems-work.md)
