##  Description

This level is pretty straightforward. The program reads the content of the file `/home/user/end/.pass` and stores it in a stream, then a buffer.
The program then takes the first argument, converts it to an integer and uses it as an index to put a null byte in the buffer.
Finally, it prints the content of the buffer, if the buffer and the av[1] are a match, the program will execute a shell, otherwise it will print the string after ths null byte that was inserted.

An overflow is not possible here, since the program uses safe functions to read the file and store it in the buffer, so we need to find a way to make the buffer content match the argument we pass.

Since the buffer is 132 bytes long, we can try to bruteforce the right argument with a loop like so:

```bash
for (( c=0; c<=132; c++ )); do ./bonus3 $c 2>/dev/null; done
```

But this does not work, the program will try to print the content of the buffer everytime.
It also does not work if we try to pass a negative number.

The real solution is so obvious that it might be hard to find: we just need to pass an empty string as a parameter.
This will cause the program to null-terminate the buffer at the first byte, effectively turning it into an empty string equaling the parameter we passed, which in turn allows us to pass the strcmp check and triggers the shell.

```bash
bonus3@RainFall:~$ ./bonus3 ""
$ whoami
end
$ cat /home/user/end/.pass
3321b6f81659f9a71c76616f606e4b50189cecfea611393d5d649f75e157353c
$ cat /home/user/end/end
Congratulations graduate!
```
