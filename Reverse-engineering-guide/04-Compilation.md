# 04 — From Code to Machine

>Human-readable code goes through multiple transformation stages before becoming the 1s and 0s your CPU runs. Understanding this pipeline is what makes reverse engineering possible.

---

## The Problem

Computers only understand **machine code** numbers that represent specific CPU operations. But writing in machine code is nearly impossible for humans. So we invented programming languages like C, which humans can read and write and **compilers** to translate them.

Reverse engineering is largely about undoing (or understanding) what the compiler did.

---

## The Compilation Pipeline

```
Source Code (.c)
      │
      │  1. PREPROCESSOR
      ▼
Expanded Source (.i)
      │
      │  2. COMPILER
      ▼
Assembly Code (.s)
      │
      │  3. ASSEMBLER
      ▼
Object File (.o)
      │
      │  4. LINKER
      ▼
Executable Binary (.exe / ELF)
```

Let's walk through each stage.

---

## Stage 1 — The Preprocessor

The preprocessor handles directives starting with `#` in C:
- `#include` → copy-pastes other files in
- `#define` → replaces text macros
- `#ifdef` → conditionally includes code

**Input:**
```c
#include <stdio.h>
#define MAX 100

int main() {
    printf("Max is %d\n", MAX);
}
```

**After preprocessing:** The `#include <stdio.h>` gets replaced by thousands of lines from the stdio header file, and `MAX` gets replaced with `100` everywhere.

---

## Stage 2 — The Compiler

This is the main event. The compiler takes the preprocessed C code and converts it to **assembly language** a human-readable, one-to-one representation of CPU instructions.

**C code:**
```c
int add(int a, int b) {
    return a + b;
}
```

**Compiled to x86-64 assembly:**
```asm
add:
    push    rbp
    mov     rbp, rsp
    mov     DWORD PTR [rbp-4], edi   ; store first argument
    mov     DWORD PTR [rbp-8], esi   ; store second argument
    mov     edx, DWORD PTR [rbp-4]
    mov     eax, DWORD PTR [rbp-8]
    add     eax, edx                  ; do the addition
    pop     rbp
    ret                               ; return the result
```

Assembly is what you'll spend most of your time reading as a reverse engineer.

### Why Assembly?

Assembly maps almost 1:1 to machine code. Each instruction becomes a specific sequence of bytes. The CPU can execute these directly.

---

## Stage 3 — The Assembler

The assembler converts assembly text into **object code** actual binary bytes.

```
Assembly:    mov eax, 1
Object code: B8 01 00 00 00
```

These object files (`.o` on Linux, `.obj` on Windows) aren't runnable yet they may reference functions defined elsewhere.

---

## Stage 4 — The Linker

Programs rarely exist alone. They use external libraries (like `printf` from the C standard library). The linker combines:
- Your object files
- Required library files
- Startup code

Into one final executable.

**Two types of linking:**

| Static Linking | Dynamic Linking |
|---------------|----------------|
| Library code is copied into your binary | Binary just stores a reference to the library |
| Bigger executable | Smaller executable |
| No external dependencies | Requires the `.dll` / `.so` file at runtime |
| Harder for RE (everything is merged) | Easier for RE (you can see which libraries it uses) |

---

## What This Means for Reverse Engineering

When you get a binary to analyze, the source code is *gone*. The compiler has:
- Renamed all variables to memory addresses
- Renamed all functions to memory addresses (sometimes)
- Optimized code (merged, inlined, or reordered operations)
- Removed comments and whitespace entirely

Your job is to work **backwards** from the binary to understand the original logic.

---

## Reading Assembly

You don't need to memorize all instructions. You need to recognize patterns.

### The Most Common Instructions

```asm
; DATA MOVEMENT
mov eax, 5          ; put the value 5 into register eax
mov eax, [rbp-4]    ; load value FROM memory address rbp-4 into eax
mov [rbp-4], eax    ; store eax's value INTO memory at rbp-4

; ARITHMETIC
add eax, ebx        ; eax = eax + ebx
sub eax, 1          ; eax = eax - 1
imul eax, ecx       ; eax = eax * ecx

; COMPARISON & JUMPS
cmp eax, 0          ; compare eax to 0 (sets flags)
je  label           ; jump if equal (jump if last cmp was equal)
jne label           ; jump if not equal
jg  label           ; jump if greater
jl  label           ; jump if less

; FUNCTION CALLS
call printf         ; call the printf function
ret                 ; return from current function

; STACK OPERATIONS
push rax            ; push rax onto the stack
pop  rax            ; pop from the stack into rax
```

### Recognizing Common Patterns

**An if statement:**
```c
// C source
if (x == 0) {
    doSomething();
}
```
```asm
; Assembly
cmp eax, 0       ; compare x to 0
jne skip         ; if NOT equal, skip the block
call doSomething ; the body of the if
skip:            ; execution resumes here
```

**A for loop:**
```c
// C source
for (int i = 0; i < 10; i++) {
    doWork(i);
}
```
```asm
; Assembly
mov ecx, 0           ; i = 0
loop_start:
cmp ecx, 10          ; i < 10?
jge loop_end         ; if not, exit
mov edi, ecx         ; pass i as argument
call doWork
inc ecx              ; i++
jmp loop_start       ; go back to top
loop_end:
```

The more patterns you recognize, the faster you can reverse code.

---

## Executable File Formats

The final binary isn't just raw machine code. It's wrapped in a format the OS understands.

### PE Format (Windows — `.exe`, `.dll`)
```
┌──────────────┐
│  DOS Header  │  ← Legacy header (starts with "MZ")
├──────────────┤
│  PE Header   │  ← Magic bytes "PE\0\0", architecture, entry point
├──────────────┤
│  Sections    │
│  .text       │  ← Executable code lives here
│  .data       │  ← Global variables
│  .rdata      │  ← Read-only data (strings, constants)
│  .bss        │  ← Uninitialized globals
│  .idata      │  ← Import table (which DLLs and functions it uses)
└──────────────┘
```

The **import table** is gold for RE — it tells you exactly which external functions the program uses. A program importing `CreateRemoteThread`, `VirtualAllocEx`, and `WriteProcessMemory` is almost certainly doing process injection. Red flag!

### ELF Format (Linux — no extension or `.so`)
Similar concept, different structure. Used on Linux, Android, and most Unix systems.

---

## Compilers and Optimization

Real-world compilers optimize heavily. This can make RE harder:

```c
// You wrote:
int x = a * 2;

// Compiler generated:
lea eax, [rdi + rdi]   ; much faster than imul, but looks weird
```

Common optimizations you'll encounter:
- **Inlining**: small functions get copy-pasted into the caller (no call instruction)
- **Dead code elimination**: if/else branches that can never execute get removed
- **Loop unrolling**: a loop of 4 iterations might just be 4 sequential copies of the body
- **Strength reduction**: expensive operations replaced with cheaper equivalents (`x*2` → `x<<1`)

---

## ✅ What You Should Know After This Chapter

- [ ] The 4 stages of compilation: preprocessor → compiler → assembler → linker
- [ ] What assembly language is and why it matters
- [ ] The basic assembly instructions: `mov`, `add`, `cmp`, `jmp`, `call`, `ret`
- [ ] How to recognize if/else and loop patterns in assembly
- [ ] The structure of PE (Windows) executables
- [ ] What the import table tells you about a binary

---

← [Previous: Memory Layout →](03-Memory-layout.md)  | [Next: Introduction to Malware →](05-Intro-to-malware.md)
