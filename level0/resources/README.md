##  Description

After logging in, we are met with some logs that show various options and checks:

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

These are extremely valuable hints for exploiting the binary `level0` located in `/home/user/level0/level0`. Let's break down what each of these mean:

- **GCC stack protector support: Enabled**: This indicates that the binary was compiled with stack protection mechanisms, which help prevent stack-based buffer overflows by placing a "canary" value on the stack that is checked before function returns. This does _not_ mean that the kernel actively enforces stack protection, just that the binary has support for it.

- **Strict user copy checks: Disabled**: This means that the kernel does not enforce strict checks on memory copying operations between user space and kernel space. This could potentially allow for certain types of exploits that involve copying data inappropriately, but it is probably not relevant for this level.

- **Restrict /dev/mem access: Enabled** and **Restrict /dev/kmem access: Enabled**: These settings restrict access to physical memory and kernel memory devices, which is a good security practice. This likely won't impact our exploitation strategy for this level.

- **grsecurity / PaX: No GRKERNSEC** and **Kernel Heap Hardening: No KERNHEAP**: These indicate that certain advanced security features provided by the grsecurity patchset are not enabled. This is rather kernel specific and may not directly affect our exploitation of the user-space binary.

- **System-wide ASLR (kernel.randomize_va_space): Off (Setting: 0)**: Address Space Layout Randomisation (ASLR) is disabled system-wide. This is a significant detail because it means that memory addresses will be _deterministic_, ie. predictable, making it easier to exploit buffer overflows and other memory corruption vulnerabilities.

The final line provides information about the binary itself:
```RELRO           STACK CANARY      NX            PIE             RPATH      RUNPATH      FILE
No RELRO        No canary found   NX enabled    No PIE          No RPATH   No RUNPATH   /home/user/level0/level0
```

- **No RELRO**: The binary does not have Read-Only Relocations enabled, which means that the Global Offset Table (GOT) is writable. What does that mean for us? It means that we can potentially overwrite function pointers in the GOT to redirect execution flow.

- **No canary found**: Despite the GCC stack protector support being enabled, this binary does not actually implement stack canaries. This is crucial because it means that stack-based buffer overflows are possible without being detected by a canary check.

- **NX enabled**: The binary has Non-Executable (NX) stack enabled, which means that the stack cannot be executed. This prevents traditional stack-based buffer overflow exploits that inject shellcode onto the stack. However, we can still use Return-Oriented Programming (ROP) techniques to exploit the binary.

- **No PIE**: Position Independent Executable (PIE) is not enabled, meaning that the binary is loaded at a fixed address in memory. This, combined with ASLR being disabled, makes it much easier to predict the locations of functions and gadgets within the binary for exploitation.

In summary, the security settings and binary characteristics indicate that while there are some protections in place (like NX), there are also significant weaknesses (like no RELRO, no canary, no PIE, and disabled ASLR) that can be exploited. This sets the stage for a buffer overflow exploit using ROP techniques to gain control of the program's execution flow.

##  Exploitation Strategy

First, we conduct a basic check of the binary:

```bash
level0@RainFall:~$ ls -l
total 732
-rwsr-x---+ 1 level1 users 747441 Mar  6  2016 level0
level0@RainFall:~$ file level0 
level0: setuid ELF 32-bit LSB executable, Intel 80386, version 1 (GNU/Linux), statically linked, for GNU/Linux 2.6.24, BuildID[sha1]=0x85cf4024dbe79c7ccf4f30e7c601a356ce04f412, not stripped
```

So we have a 32-bit binary with the setuid bit set as user `level1`. Executing it without arguments causes a segmentation fault:

```bash
level0@RainFall:~$ ./level0
Segmentation fault (core dumped)
```

Let's try with an argument:

```bash
level0@RainFall:~$ ./level0 1
No !
```

Alright, let's analyse the binary with `gdb` to understand its behavior from the main function:

```bash
level0@RainFall:~$ gdb -batch ./level0 -ex 'disassemble main'
Dump of assembler code for function main:
   0x08048ec0 <+0>:	push   %ebp
   0x08048ec1 <+1>:	mov    %esp,%ebp
   0x08048ec3 <+3>:	and    $0xfffffff0,%esp
   0x08048ec6 <+6>:	sub    $0x20,%esp
   0x08048ec9 <+9>:	mov    0xc(%ebp),%eax
   0x08048ecc <+12>:	add    $0x4,%eax
   0x08048ecf <+15>:	mov    (%eax),%eax
   0x08048ed1 <+17>:	mov    %eax,(%esp)
   0x08048ed4 <+20>:	call   0x8049710 <atoi>
   0x08048ed9 <+25>:	cmp    $0x1a7,%eax
   0x08048ede <+30>:	jne    0x8048f58 <main+152>
   0x08048ee0 <+32>:	movl   $0x80c5348,(%esp)
   0x08048ee7 <+39>:	call   0x8050bf0 <strdup>
   0x08048eec <+44>:	mov    %eax,0x10(%esp)
   0x08048ef0 <+48>:	movl   $0x0,0x14(%esp)
   0x08048ef8 <+56>:	call   0x8054680 <getegid>
   0x08048efd <+61>:	mov    %eax,0x1c(%esp)
   0x08048f01 <+65>:	call   0x8054670 <geteuid>
   0x08048f06 <+70>:	mov    %eax,0x18(%esp)
   0x08048f0a <+74>:	mov    0x1c(%esp),%eax
   0x08048f0e <+78>:	mov    %eax,0x8(%esp)
   0x08048f12 <+82>:	mov    0x1c(%esp),%eax
   0x08048f16 <+86>:	mov    %eax,0x4(%esp)
   0x08048f1a <+90>:	mov    0x1c(%esp),%eax
   0x08048f1e <+94>:	mov    %eax,(%esp)
   0x08048f21 <+97>:	call   0x8054700 <setresgid>
   0x08048f26 <+102>:	mov    0x18(%esp),%eax
   0x08048f2a <+106>:	mov    %eax,0x8(%esp)
   0x08048f2e <+110>:	mov    0x18(%esp),%eax
   0x08048f32 <+114>:	mov    %eax,0x4(%esp)
   0x08048f36 <+118>:	mov    0x18(%esp),%eax
   0x08048f3a <+122>:	mov    %eax,(%esp)
   0x08048f3d <+125>:	call   0x8054690 <setresuid>
   0x08048f42 <+130>:	lea    0x10(%esp),%eax
   0x08048f46 <+134>:	mov    %eax,0x4(%esp)
   0x08048f4a <+138>:	movl   $0x80c5348,(%esp)
   0x08048f51 <+145>:	call   0x8054640 <execv>
   0x08048f56 <+150>:	jmp    0x8048f80 <main+192>
   0x08048f58 <+152>:	mov    0x80ee170,%eax
   0x08048f5d <+157>:	mov    %eax,%edx
   0x08048f5f <+159>:	mov    $0x80c5350,%eax
   0x08048f64 <+164>:	mov    %edx,0xc(%esp)
   0x08048f68 <+168>:	movl   $0x5,0x8(%esp)
   0x08048f70 <+176>:	movl   $0x1,0x4(%esp)
   0x08048f78 <+184>:	mov    %eax,(%esp)
   0x08048f7b <+187>:	call   0x804a230 <fwrite>
   0x08048f80 <+192>:	mov    $0x0,%eax
   0x08048f85 <+197>:	leave  
   0x08048f86 <+198>:	ret    
End of assembler dump.
```

Here we can see that the program expects an argument that, when converted to an integer using `atoi`, should equal `0x1a7` (423 in decimal). So let's try that:

```bash
level0@RainFall:~$ ./level0 423
$ whoami
level1
```

Wait, what? Let's understand what happened there: the `level0` binary has the setuid bit set, meaning that when executed, it runs with the privileges of the `level1` user. When we provided the correct argument (423), the program executed `/bin/sh` (a shell) with `level1` privileges using the `execv` function. This effectively gives us a shell as the `level1` user.

Now that we have a shell as `level1`, we can read the `.pass` file located in the home directory of the `level1` user to get the password for the next level:

```bash
$ cat /home/user/level1/.pass
1fe8a524fa4bec01ca4ea2a869af2a02260d4a7d5fe7e7c24d8617e6dca12d3a
```

There you have it. In spite of all the security mechanisms in place, the real strategy here was very simple: read the disassembly, understand the required input, and give the binary what it wants. All the logs about security features were probably red herrings in this case, but it might not always be so in other levels.