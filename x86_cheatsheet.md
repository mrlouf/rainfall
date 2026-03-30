##  x86 ASM Cheat Sheet

x86 assembly language is a low-level, 32-bit programming language that is closely related to machine code. It provides a way to write instructions that the CPU can execute directly. I had previous experience with assembly language, but with x86-64 architecture, which differs from x86 in several ways.

Here is my attempt at creating a cheat sheet to highlight recurrent patterns in x86 assembly language, and help me reverse engineer the code, ie. re-write C code from assembly.

Note that this is a simplified overview based on my observations in the context of this project, and might not reflect all aspects of x86 assembly in real-world scenarios.

### Stack usage for function calls

Where x86_64 relies on dedicated registers (rdi, rsi, rdx, rcx, etc.) to pass parameters to functions, x86 relies heavily on the stack, and the esp register is used as the stack pointer to load the arguments for function calls.

```assembly
mov X, (%esp)
mov Y, 0x4(%esp)
mov Z, 0x8(%esp)
call foo
```

This translates to this C function call:

```c
foo(X, Y, Z);
```

### Prologue and Epilogue

Functions in x86 assembly typically start with a prologue that sets up the stack frame and end with an epilogue that cleans up the stack frame before returning, something that is not present in x86-64 assembly.

```assembly
push    %ebp
mov     %esp, %ebp
and     $0xfffffff0, %esp
sub     $0x20, %esp
```

This is typical boilerplate for a function that aligns 16 bytes on the stack (0xfffffff0), then reserves 32 bytes (0x20) on the stack for local variables. It can be translated to C to something like:

```c
void function() {
    char local_var[32];
    ...
}
```

At the end of the function, we typically see the epilogue:

```assembly
mov     <value>, %eax
leave
ret
```

This is a typical epilogue that loads a value into the eax register (the return value), then returns from the function. The `leave` instruction is equivalent to:

```assembly
mov %ebp, %esp
pop %ebp
```

In the calling convention for i386, the stack layout for a main function call looks like this:

```assembly
[ebp+12]  → argv (char **)
[ebp+8]   → argc (int)
[ebp+4]   → return address
[ebp]     → saved ebp
```

### CMP and conditional jumps

The `cmp` instruction is used to compare two values, and it sets the CPU flags based on the result of the comparison. This is often followed by conditional jump instructions that alter the flow of execution based on the comparison result.

Let's consider that we have the following snippet from a main function:

```assembly
cmp     $0x0, %eax
je      <main+24>
```

This is a very simple comparison that checks if the value in the eax register is equal to 0. If it is, the execution jumps to <main+24>, otherwise it continues sequentially. This can be translated to C as:

```c
if (eax == 0) {
    // success code
}
```

Even though <main+24> is located later in the assembly code, C almost always will place the success code inside the if block, as shown above, not at the end of the function. This can be a bit confusing at first, but if we keep in mind that assembly code is linear and C code is structured, it becomes somewhat easier to follow.
