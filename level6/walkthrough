##	Description

This level introduces a new difficulty in reverse-engineering: function pointers.

The main allocates two pointers, `ptr1` and `ptr2`. The first one is allocated with 64 bytes, while the second one is allocated with only 4 bytes, which is the usual size for a pointer on 32-bit systems.

If we look at all the call instructions from the disassembly, we can see that right before the return, instead of calling the desired function directly, the program calls it from the pointer previously allocated:

```assembly
   0x080484d0 <+84>:	call   *%eax
```

This means that the program is calling a function from the address stored in the register `%eax`. Judging from the behaviour of the program, the function called is `m()`, which is the one that prints "Nope". This means that the address of `m()` is stored in `%eax` at this point. But we want to call `n()`, which calls `system("/bin/cat /home/user/level7/.pass")` and prints the password for the next level.

We can do this by overflowing the buffer allocated for `ptr1`; this is done with the `strcpy(ptr1, ptr2)` instruction. Since `ptr1` is allocated with 64 bytes, we need to write more than 64 bytes to overflow it. The first 64 bytes will fill the buffer plus some 8 bytes to overwrite return addresses and other irrelevant words, then overwrite the value of `ptr2` with the address of `n()`.

The payload we need is this one:

```bash
	level6@RainFall:~$ ./level6 $(python -c 'print "A"*72 + "\x54\x84\x04\x08"')
```
