# rainfall

CTF project based on ELF patching and disassembling on i386 systems.

##  Description

This project follows the `snow-crash` CTF challenge where we explored binary exploitation, reverse engineering, and disassembling techniques. In this project, we will build upon that knowledge and introduce new concepts related to ELF patching and advanced disassembly techniques.

Again, the project is structured as a series of levels, each focusing on a specific aspect of ELF binaries and disassembly. The objective on each level is to manage to read the `.pass` file with the with the "levelX" user account of the next level (X = number of the next level).

The ".pass" file is located at the home directory of each (level0 excluded) user.

### Objective

The main objective of this project is _not_ to simply get the flag for each level. This can be easily achieved with the help of IA or looking up walkthroughs.

Instead, the real goal is to learn and practice **reverse engineering**, that is analysing unknown binaries to understand their functionality, flow, and being able to reconstruct the source code from the disassembly. Only after we get a solid understanding of the source code can we start thinking about possible vulnerabilities and exploitation techniques.

##  Setup

To set up the environment for this project, you need to start a VM using the provided ISO file. The VM is pre-configured with all the necessary tools and dependencies required for the challenges, all you need to do is confiure the network settings to use a bridged adapter so you can access the VM from your host machine via ssh on port 4242:

```bash
nicolas@pop-os:~$ ssh level0@192.168.1.44 -p4242
	  _____       _       ______    _ _ 
	 |  __ \     (_)     |  ____|  | | |
	 | |__) |__ _ _ _ __ | |__ __ _| | |
	 |  _  /  _` | | '_ \|  __/ _` | | |
	 | | \ \ (_| | | | | | | | (_| | | |
	 |_|  \_\__,_|_|_| |_|_|  \__,_|_|_|

                 Good luck & Have fun

  To start, ssh with level0/level0 on 192.168.1.44:4242
level0@192.168.1.44's password: 
```

The password for the "level0" user is provided in the subject's PDF.

##  Environment

After logging in as level0, we are met with some logs that show various options and checks:

```bash
level0@192.168.1.44's password: 
  GCC stack protector support:            Enabled
  Strict user copy checks:                Disabled
  Restrict /dev/mem access:               Enabled
  Restrict /dev/kmem access:              Enabled
  grsecurity / PaX: No GRKERNSEC
  Kernel Heap Hardening: No KERNHEAP
 System-wide ASLR (kernel.randomize_va_space): Off (Setting: 0)
RELRO           STACK CANARY      NX            PIE             RPATH      RUNPATH      FILE
No RELRO        No canary found   NX enabled    No PIE          No RPATH   No RUNPATH   /home/user/level0/level0
```

Let's break down what each of these hints mean:

- **GCC stack protector support: Enabled**: This indicates that the binary was compiled with stack protection mechanisms, which help prevent stack-based buffer overflows by placing a "canary" value on the stack that is checked before function returns. This does _not_ mean that the kernel actively enforces stack protection, just that the binary has support for it.

- **Strict user copy checks: Disabled**: This means that the kernel does not enforce strict checks on memory copying operations between user space and kernel space. This could potentially allow for certain types of exploits that involve copying data inappropriately, but it is probably not relevant for this level.

- **Restrict /dev/mem access: Enabled** and **Restrict /dev/kmem access: Enabled**: These settings restrict access to physical memory and kernel memory devices, which is a good security practice. This likely won't impact our exploitation strategy for this level.

- **grsecurity / PaX: No GRKERNSEC** and **Kernel Heap Hardening: No KERNHEAP**: These indicate that certain advanced security features provided by the grsecurity patchset are not enabled. This is rather kernel specific and may not directly affect our exploitation of the user-space binary.

- **System-wide ASLR (kernel.randomize_va_space): Off (Setting: 0)**: Address Space Layout Randomisation (ASLR) is disabled system-wide. This is a significant detail because it means that memory addresses will be _deterministic_, ie. predictable, making it easier to exploit buffer overflows and other memory corruption vulnerabilities. In simplified terms, the locations of the segments, libraries, stack, functions, etc. will remain constant across executions of the binary so that we can reliably target them.

The final line provides information about the binary itself:
```RELRO           STACK CANARY      NX            PIE             RPATH      RUNPATH      FILE
No RELRO        No canary found   NX enabled    No PIE          No RPATH   No RUNPATH   /home/user/level0/level0
```

- **No RELRO**: The binary does not have Read-Only Relocations enabled, which means that the Global Offset Table (GOT) is writable. What does that mean for us? It means that we can potentially **overwrite function pointers in the GOT** to redirect execution flow. In other words, if the binary calls a function like `exit()`, we could overwrite address for `exit` to point to our own function, our own shellcode or ROP chain, allowing us to execute arbitrary code when the function is called.

- **No canary found**: Despite the GCC stack protector support being enabled, this binary does not actually implement stack canaries. This is crucial because it means that stack-based buffer overflows are possible without being detected by a canary check.

- **NX enabled**: The binary has Non-Executable (NX) stack enabled, which means that the stack cannot be executed. This prevents traditional stack-based buffer overflow exploits that inject shellcode onto the stack. However, we can still use Return-Oriented Programming (ROP) techniques to exploit the binary.

- **No PIE**: Position Independent Executable (PIE) is not enabled, meaning that the binary is loaded at a fixed address in memory. This, combined with ASLR being disabled, makes it much easier to predict the locations of functions and gadgets within the binary for exploitation.

In summary, the security settings and binary characteristics indicate that while there are some protections in place (like NX), there are also significant weaknesses (like no RELRO, no canary, no PIE, and disabled ASLR) that can be exploited. This sets the stage for a buffer overflow exploit using ROP techniques to gain control of the program's execution flow.

##  Useful commands and tools

Our best allies in this project are mainly `readelf` and `gdb`. Here are some very useful ways to use them:

- `readelf -a <binary>`: Displays detailed information about the ELF binary, including headers, sections, and segments.
- `objdump -s -j .rodata <binary>`: Dumps the contents of the .rodata section, which often contains string literals and other read-only data.
- `gdb -batch <binary> -ex "disas main"`: Disassembles the main function of the binary in a non-interactive way. Replace `main` with any function name to disassemble other functions.

Useful gdb commands in interactive mode:

- `info proc mappings`: Displays all the mapped address spaces in the program, from the stack to the heap and the libraries (libc) used. Very useful to start dumping memory at runtime.
- `break <function_name>`: Sets a breakpoint at the beginning of the specified function. Note: 'break' can be abbreviated as 'b'.
- `b *<address>`: Sets a breakpoint at the specified memory address in hexadecimal. Note: the asterisk (*) indicates that the address is an absolute address.
- `run` or `r`: Starts the execution of the program until it hits a breakpoint or finishes.
- `next` or `n`: Executes the next line of code, stepping over function calls.
- `set $eip = <address>`: Sets the instruction pointer (EIP) to the specified address, allowing you 'cheat' the execution flow and jump to different parts of the code. Useful to skip certain checks or reach hidden functions for example, but may lead to segfaults or undefined behavior if not used carefully.
- `info registers`: Displays the current values of all CPU registers, useful for understanding the program's state during execution and get the values of the registers.
- `x/<format> <address>`: Examines memory at the specified address. The format can be adjusted to display data in different ways (e.g., `x/s` for string, `x/x` for hexadecimal).
