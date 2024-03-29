---
title: "Intro to RE: C : part-3"
date: 2023-04-01T21:59:33+05:30
draft: false
showtoc: true
tags: [RE, linux, C programming]
series: ["RE:C"]
description: Blog covers how disassembly of basic operations and functions in C programming looks like.
# math: true
# ShowBreadCrumbs: false
# searchHidden: true
# hideMeta: true
---

In the previous blog, I discussed some of the basic C program's disassembly structures, concentrating on the variables and their memory layouts. This article, a follow-up to the previous one, focuses on basic operations and functions in C programs.


![](https://media.giphy.com/media/3BUYbmXltgQ4zu0Tv5/giphy.gif#center)


In the previous blogs, we have seen what an empty C program looks like

```c
void main() {}
```

Disassembly:

```asm
main:
        push    rbp
        mov     rbp, rsp
        nop
        pop     rbp
        ret
```

## Arithmatic operators

Now if we want to work with operations, we'll have to add 2 local variables to the function. Something like in the below example.

```c
void main() {
    int a=1, b=2;
}
```

Disassembly:

```asm
main:
        push    rbp
        mov     rbp, rsp
        mov     DWORD PTR [rbp-4], 1
        mov     DWORD PTR [rbp-8], 2
        nop
        pop     rbp
        ret
```

### Addition
Now let's perform add operation on the 2 local variables we created and save the result in a new variable.


```c
void main() {
    int a=1, b=2;
    int c = a + b;
}
```

Disassembly:

```asm
main:
        push    rbp
        mov     rbp, rsp
        mov     DWORD PTR [rbp-4], 1
        mov     DWORD PTR [rbp-8], 2
        mov     edx, DWORD PTR [rbp-4]
        mov     eax, DWORD PTR [rbp-8]
        add     eax, edx
        mov     DWORD PTR [rbp-12], eax
        nop
        pop     rbp
        ret
```

We can see a few new instructions in the disassembly code that are responsible for the `int c = a + b` instruction in the source code.

When we look at them separately, it becomes quite natural to understand.

```asm
main:
        push    rbp
        mov     rbp, rsp
        mov     DWORD PTR [rbp-4], 1
        mov     DWORD PTR [rbp-8], 2

        mov     edx, DWORD PTR [rbp-4]      ; Save the first variable value in EDX register
        mov     eax, DWORD PTR [rbp-8]      ; Save the second variable value in EAX register
        add     eax, edx                    ; add EAX and EDX register values, this stores the result in EAX here
        mov     DWORD PTR [rbp-12], eax     ; Move the new value of EAX to third variable

        nop
        pop     rbp
        ret
```


Let's look at other arithmetic operations as well


### Subtraction

```c
void main() {
    int a=1, b=2;
    int c = a - b;
}
```

Disassembly:

```asm
main:
        push    rbp
        mov     rbp, rsp
        mov     DWORD PTR [rbp-4], 1
        mov     DWORD PTR [rbp-8], 2

        mov     eax, DWORD PTR [rbp-4]      ; load first variable in EAX
        sub     eax, DWORD PTR [rbp-8]      ; subtract EAX with second variable, then save the result in EAX
        mov     DWORD PTR [rbp-12], eax     ; save the new value of EAX in the third register

        nop
        pop     rbp
        ret
```


### Multiplication

```c
void main() {
    int a=1, b=2;
    int c = a * b;
}
```

Disassembly:

```asm
main:
        push    rbp
        mov     rbp, rsp
        mov     DWORD PTR [rbp-4], 1
        mov     DWORD PTR [rbp-8], 2
        mov     eax, DWORD PTR [rbp-4]      ; load first variable
        imul    eax, DWORD PTR [rbp-8]      ; multiply it with second
        mov     DWORD PTR [rbp-12], eax     ; save it in the third
        nop
        pop     rbp
        ret
```


### Division and modulo

If you are not aware, the division operation is about calculating the quotient and the modulo operation is about the remainder.

```c
void main() {
    int a = 1, b = 2;
    int c = a / b;
    int d = a % b;
}
```

Disassembly:

```asm
main:
        ; prologue
        push    rbp
        mov     rbp, rsp

        ; first and second variable
        mov     DWORD PTR [rbp-4], 1
        mov     DWORD PTR [rbp-8], 2

        ; division
        mov     eax, DWORD PTR [rbp-4]      ; Load first variable in EAX
        cdq                                 ; Convert double to quad value;
        idiv    DWORD PTR [rbp-8]           ; perform idiv operation with second variable
        mov     DWORD PTR [rbp-12], eax     ; Store new EAX value in third variable

        ; modulo
        mov     eax, DWORD PTR [rbp-4]      ; Load the first value again
        cdq                                 ; Convert double to quad value;
        idiv    DWORD PTR [rbp-8]           ; perform idiv operation with second variable
        mov     DWORD PTR [rbp-16], edx     ; Store the EDX value in fourth variable

        ; epilogue
        nop
        pop     rbp
        ret
```

If you didn't notice, the **division result** was stored in the `EAX` register, while the **modulo result** was stored in the `EDX`...Everything else stays unchanged.


Its OKAY if you are having questions like-

- How?
- why EDX??
- WTF is going on???
- Does it perform both operations even if either one of them is required????

These instructions, however, are not as simple to understand as others. So allow me to attempt to explain what's going on.

To begin, you must comprehend the cdq instruction's magic. This converts a `Doubleword` to a `Quadword` by extending the sign bit of `EAX` into the `EDX` register. For the purposes of this blog, consider the `EAX` and `EDX` to be joined together to form a large quadword register. So, if `EDX` contains `0x12` and `EAX` contains `0x3456789a`, the resulting value is `0x123456789a`. Does that make sense?

So when a `idiv` (or other `div` derivatives) operation is performed, both the quotient and the remainder are calculated. The instruction stores the quotient in `EAX` and the remainder in `EDX` register.

Now that you understand the concept, you can think about removing some of the repeated instructions to make your program smaller and run faster.

```asm
main:
        ; prologue
        push    rbp
        mov     rbp, rsp

        ; first and second variable
        mov     DWORD PTR [rbp-4], 1
        mov     DWORD PTR [rbp-8], 2

        ; division and modulo
        mov     eax, DWORD PTR [rbp-4]      ; Load first variable in EAX
        cdq                                 ; Convert double to quad value;
        idiv    DWORD PTR [rbp-8]           ; perform idiv operation with second variable
        mov     DWORD PTR [rbp-12], eax     ; Store new EAX value in third variable (quotient)
        mov     DWORD PTR [rbp-16], edx     ; Store the EDX value in fourth variable (remainder)

        ; epilogue
        nop
        pop     rbp
        ret
```


Another point worth mentioning is that `div` operations cannot be used without overwriting the contents of the `EAX` and `EDX` registers. If you want to use the values of these registers after the `div` operation, save them somewhere else where they can be read later.


## Increment/Decrement operators

That's all there is to arithmetic operators. Let's move on to the increment and decrement operators...

```c
void main() {
    int A = 5;
    int B = A++;
    int C = ++A;
}
```

Disassembly:-

```asm
main:
        ; prologue
        push    rbp
        mov     rbp, rsp

        ; int A = 5;
        mov     DWORD PTR [rbp-4], 5

        ; int B = A++;
        mov     eax, DWORD PTR [rbp-4]      ; load the value from variable A in EAX
        lea     edx, [rax+1]                ; increment the value and store it in EDX
        mov     DWORD PTR [rbp-4], edx      ; update the incremented value in the variable A
        mov     DWORD PTR [rbp-8], eax      ; Load the old EAX value in variable B;

        ; int C = ++A;
        add     DWORD PTR [rbp-4], 1        ; Increment the value of variable A
        mov     eax, DWORD PTR [rbp-4]      ; Load the updated value of variable A in EAX
        mov     DWORD PTR [rbp-12], eax     ; Store the EAX value in variable C

        ; epilogue
        nop
        pop     rbp
        ret
```

At this level, I believe you can see that this operator is nothing special. I'll leave the decrement operator upto you to test and disect. You can always use [godbolt.org](https://godbolt.org/) [^godbolt_url] for quick testing.


[^godbolt_url]: https://godbolt.org/


## Bitwise operators
We can now proceed to examine the bitwise operators from a low-level perspective.
(PS: They are my personal favourites)


```c
void main() {
    int A = 5, B = 0;
    int C1 = A & B;
    int C2 = A | B;
    int C3 = A ^ B;
    int C4 = ~ B;
}
```

Disassembly:-

```asm
main:
        ; prologue
        push    rbp
        mov     rbp, rsp

        ; int A = 5, B = 0;
        mov     DWORD PTR [rbp-4], 5
        mov     DWORD PTR [rbp-8], 0

        ; int C1 = A & B;
        mov     eax, DWORD PTR [rbp-4]
        and     eax, DWORD PTR [rbp-8]
        mov     DWORD PTR [rbp-12], eax

        ; int C1 = A | B;
        mov     eax, DWORD PTR [rbp-4]
        or      eax, DWORD PTR [rbp-8]
        mov     DWORD PTR [rbp-16], eax

        ; int C1 = A ^ B;
        mov     eax, DWORD PTR [rbp-4]
        xor     eax, DWORD PTR [rbp-8]
        mov     DWORD PTR [rbp-20], eax

        ; int C1 = ~ B;
        mov     eax, DWORD PTR [rbp-8]
        not     eax
        mov     DWORD PTR [rbp-24], eax

        ; epilogue
        nop
        pop     rbp
        ret
```


Simple and neat. Aren't they? Load the variables in registers, perform the operation, store the result.

### shift right/left operators
Then there are shift operators - shift left and shift right.


```c
void main() {
    int A = 1;
    int B = A << 4;
    int C = B >> 4;
}
```

Disassembly:-

```asm
main:
        ; prologue
        push    rbp
        mov     rbp, rsp
        ; int A = 1;
        mov     DWORD PTR [rbp-4], 1

        ; int B = A << 4;
        mov     eax, DWORD PTR [rbp-4]
        sal     eax, 4                  ; Shift arithmetic left
        mov     DWORD PTR [rbp-8], eax

        ; int C = B >> 4;
        mov     eax, DWORD PTR [rbp-8]
        sar     eax, 4                  ; Shift arithmetic right
        mov     DWORD PTR [rbp-12], eax

        ; epilogue
        nop
        pop     rbp
        ret
```


If you look at their binary representation, shift operators are very straightforward. Allow me to create an image for you.

- Initial value in memory

```goat

  Index =>       7 6 5 4 3 2 1 0
                ┌─┬─┬─┬─┬─┬─┬─┬─┐
  Value =>      │0│0│0│0│0│0│0│1│
                └─┴─┴─┴─┴─┴─┴─┴─┘

```

- After shifting left 4 times.

```goat


  Index =>       7 6 5 4 3 2 1 0
                ┌─┬─┬─┬─┬─┬─┬─┬─┐
  Value =>      │0│0│0│1│.│.│.│.│
                └─┴─┴─┴─┴─┴─┴─┴─┘

                   ◄──────────────
                       4 3 2 1

```


Blocks with "." are the freshly shifted block from outside the memory frame. These blocks are packed with zeroes. This makes our resulting value `2^4 = 16`.


```goat

  Index =>       7 6 5 4 3 2 1 0
                ┌─┬─┬─┬─┬─┬─┬─┬─┐
  Value =>      │0│0│0│1│0│0│0│0│
                └─┴─┴─┴─┴─┴─┴─┴─┘
```



If we shift it right 4 times we'll get our initial value.

```goat

Before:

   Index =>       7 6 5 4 3 2 1 0
                 ┌─┬─┬─┬─┬─┬─┬─┬─┐
   Value =>      │0│0│0│1│0│0│0│0│
                 └─┴─┴─┴─┴─┴─┴─┴─┘

                 ─────────────────►
                          1 2 3 4




After:

   Index =>       7 6 5 4 3 2 1 0
                 ┌─┬─┬─┬─┬─┬─┬─┬─┐
   Value =>      │0│0│0│0│0│0│0│1│
                 └─┴─┴─┴─┴─┴─┴─┴─┘


```

Consider this: if we conducted a shift right operation with this, the entire frame would be filled with 0s, and the resultant value would be zero. No matter how many shifts we make.

Another intriguing thing is that you can multiply a number by `2` using the shift left procedure...without actually using the `*` operation.

```
void main() {
    int a = 589;
    int X = a*2;
    int Y = a << 1;
}
```


```
main:
        push    rbp
        mov     rbp, rsp
        mov     DWORD PTR [rbp-4], 589

        ; int X = a*2;
        mov     eax, DWORD PTR [rbp-4]
        add     eax, eax
        mov     DWORD PTR [rbp-8], eax

        ; int Y = a << 1;
        mov     eax, DWORD PTR [rbp-4]
        add     eax, eax
        mov     DWORD PTR [rbp-12], eax

        nop
        pop     rbp
        ret
```

At a lower level, they are identical. Nothing particularly useful, but it's good to know what's happening behind the scenes.

## Branching

Now comes the branching. Every good program employs branching for one reason or another. This is very useful to understand when considering reverse engineering.

### If-else

```c
void main() {
    int a = 1;
    int x;

    if(a==2)
        x = 10;
    else
        x = 5;
}
```

Disassembly:

```asm
main:
        push    rbp
        mov     rbp, rsp
        mov     DWORD PTR [rbp-4], 1
        cmp     DWORD PTR [rbp-4], 2
        jne     .L2
        mov     DWORD PTR [rbp-8], 10
        jmp     .L4
.L2:
        mov     DWORD PTR [rbp-8], 5
.L4:
        nop
        pop     rbp
        ret
```

Let's understand this step by step

| Line  |  Description |
|----------|----------------------------------|
| Line 1     | label for the function starting |
| Line 2:3   | Prologue; Setting up the function frame |
| Line 4     | int a = 1; |
| Line 5     | Comparing this value with a hardcoded value `2` |
| Line 6     | If the result of the comparision is *not equal*, then jump to `L2` flag |
| Line 7     | x = 10; This will run if it didn't jump to `L2` |
| Line 8     | Now jump to `L4` |
| Line 9     | Flag for `L2` |
| Line 10    | x = 5; |
| Line 11:14 | epilogue for the function |


Here is a graph to make it more simpler

```goat{linenos=false}
                ┌────────────────────────────────────────────────────┐
                │ push rbp                                           │
                │                                                    │
                │ mov rbp, rsp                                       │
                │                                                    │
                │ mov dword [rbp-0x4], 1                             │
                │                                                    │
                │ cmp dword [rbp-0x4], 2                             │
                │                                                    │
                │ jne .L2                                            │
                └────────────────────────────────────────────────────┘
                        f        t

                        │        │
                        │        └──────────────┐
              ┌─────────┘                       │
              │                                 │
        ┌────────────────────────────┐    ┌───────────────────────────────────┐
        │ mov dword [rbp-0x8], 10    │    │ mov dword [rbp-0x8], 5            │
        │                            │    │                                   │
        │ jmp .L4                    │    │                                   │
        └────────────────────────────┘    └───────────────────────────────────┘
                v                                 v
                │                                 │
                └────────────────┐                │
                                 │ ┌──────────────┘
                                 │ │
                        ┌───────────────────────────────────┐
                        │ nop                               │
                        │                                   │
                        │ pop rbp                           │
                        │                                   │
                        │ ret                               │
                        └───────────────────────────────────┘
```

### Switch-case
Branching can also be implemented with switch-case directives in C and some other languages. At lower level, they function similarly to if-else.

```c

void main() {
    int a = 1;
    int x;

    switch(a){
        case 1: {
            x = 10;
            break;
        }

        case 2: {
            x = 20;
            break;
        }
    }
}
```

Disassembly:-

```asm
main:
        push    rbp
        mov     rbp, rsp
        mov     DWORD PTR [rbp-4], 1

        ; Compare and jump if equal
        cmp     DWORD PTR [rbp-4], 1
        je      .L2

        ; Compare and jump if equal
        cmp     DWORD PTR [rbp-4], 2
        je      .L3

        ; Default jump to the end
        jmp     .L4

.L2:
        mov     DWORD PTR [rbp-8], 10
        jmp     .L4

.L3:
        mov     DWORD PTR [rbp-8], 20
        nop

.L4:
        nop
        pop     rbp
        ret
```


See...just like if-else statements, switch-case statements also use `cmp` and `jmp` instructions to create branches in the flow.

Graph diagram for the above disassembly will look something like this

```goat

                    ┌────────────────────────────────────────────────────┐
                    │ push rbp                                           │
                    │                                                    │
                    │ mov rbp, rsp                                       │
                    │                                                    │
                    │ mov dword [rbp-0x4], 1                             │
                    │                                                    │
                    │ cmp dword [rbp-0x4], 1                             │
                    │                                                    │
                    │ je .L2                                             │
                    └────────────────────────────────────────────────────┘
                            f t
                            │ │
                            │ └──────────────────────┐
                  ┌─────────┘                        │
                  │                                  │
              ┌──────────────────────────┐       ┌───────────────────────────────────┐
              │ cmp dword [rbp-0x4], 2   │       │  mov dword [rbp-0x8], 10          │
              │                          │       │                                   │
              │ je .L3                   │       │  jmp .L4                          │
              └──────────────────────────┘       │                                   │
                      f t                        └───────────────────────────────────┘
                      │ │                            v
                      │ │                            │
                      │ └──────┐                     │
     ┌────────────────┘        │                     │
     │                         │                     └─────────────┐
     │                         │                                   │
 ┌────────────────────┐    ┌───────────────────────────────────┐   │
 │ jmp .L4            │    │ mov dword [rbp-0x8], 20           │   │
 └────────────────────┘    │                                   │   │
     v                     │ nop                               │   │
     │                     └───────────────────────────────────┘   │
     │                         v                                   │
     │                         │                                   │
     └─────────────────┐       │                                   │
                       │ ┌─────┘                                   │
                       │ │ ┌───────────────────────────────────────┘
                       │ │ │
                 ┌───────────────────────────────────────────────┐
                 │ nop                                           │
                 │                                               │
                 │ pop rbp                                       │
                 │                                               │
                 │ ret                                           │
                 └───────────────────────────────────────────────┘
```



With all that out of the way, let us take a brief look at how function calling works at the low level.

## Functions

I always wished to demonstrate people an infinite loop with recursion. So here you have it.

```c
void main()
{
    main();
}
```

Disassembly:-

```asm
main:
        push    rbp
        mov     rbp, rsp
        mov     eax, 0
        call    main
        nop
        pop     rbp
        ret
```


Each time the `call main` instruction is encountered, the `main()` function is called, and a new function frame is created. Due to the lack of an exit condition, the processor will never be able to read anything beyond the `call main` instruction, and thus the function will never return. Hence, the infinite loop.



Now take a look at how things change when we add arguments to a function.


```c
void main()
{
    main(1,2,3,4,5,6,7,8,9,10);
}
```

Disassembly:-

```asm
main:
        push    rbp
        mov     rbp, rsp
        push    10
        push    9
        push    8
        push    7
        mov     r9d, 6
        mov     r8d, 5
        mov     ecx, 4
        mov     edx, 3
        mov     esi, 2
        mov     edi, 1
        mov     eax, 0
        call    main
        add     rsp, 32
        nop
        leave
        ret
```

If you examine the pattern, you will notice that the arguments are loaded in a particular sequence - from right to left. The first six arguments remain in the registers `edi`, `esi`, `edx`, `ecx`, `r8d` & `r9d` (from left to right). The rest of the arguments are stored on the stack.

This pattern is followed by any function that you wish to invoke from your code.

```c
void main()
{
    printf(1,2,3);
}
```

Disassembly:-

```asm
main:
        push    rbp
        mov     rbp, rsp
        mov     edx, 3
        mov     esi, 2
        mov     edi, 1
        mov     eax, 0
        call    printf
        nop
        pop     rbp
        ret
```


This is obviously not the correct method to invoke a `printf` function. `Printf`'s first argument should be a string (possibly a format string).

```c
void main()
{
    printf("Hello");
}
```

Disassembly:-

```asm
.LC0:
        .string "Hello"
main:
        push    rbp
        mov     rbp, rsp
        mov     edi, OFFSET FLAT:.LC0
        mov     eax, 0
        call    printf
        nop
        pop     rbp
        ret
```

The `Hello` string is kept at `LC0` offset here. So we load the offset's pointer to value and put it in `edi`. Then execute the `printf()` function, which will take as its first argument the value stored in the `edi` register.

If you are wondering what's the point of `mov     eax, 0` just before `printf` call... [read this StackOverflow thread](https://stackoverflow.com/questions/6212665/why-is-eax-zeroed-before-a-call-to-printf) [^eax_0]

[^eax_0]: https://stackoverflow.com/questions/6212665/why-is-eax-zeroed-before-a-call-to-printf

If we add 1 more argument to `printf` call, that should be stored in `esi` register. And the right-most argument will be processed first.

```c
void main()
{
    printf("%d", 10);
}
```

Disassembly:-

```asm
.LC0:
        .string "%d"
main:
        push    rbp
        mov     rbp, rsp
        mov     esi, 10
        mov     edi, OFFSET FLAT:.LC0
        mov     eax, 0
        call    printf
        nop
        pop     rbp
        ret
```

`Printf`, like all other C functions, has a return value that is an `int` type. `Printf` gives the number of characters in the format string that the function has processed.


```c
void main()
{
    int x = printf("%d\n", 10);
    printf("%d\n", x);
}
```

Disassembly:-

```asm
.LC0:
        .string "%d"
main:
        ; Prologue
        push    rbp
        mov     rbp, rsp

        ; Getting memory block for variables
        sub     rsp, 16

        ; first printf
        mov     esi, 10
        mov     edi, OFFSET FLAT:.LC0
        mov     eax, 0
        call    printf

        ; Storing return value (eax) of printf in the local variable
        mov     DWORD PTR [rbp-4], eax

        ; second printf
        mov     eax, DWORD PTR [rbp-4]
        mov     esi, eax
        mov     edi, OFFSET FLAT:.LC0
        mov     eax, 0
        call    printf

        ; epologue
        nop
        leave
        ret
```
Remember how we talked about that the return values from functions are stored in `eax` register. Here also, the return value from `printf` is stored in `eax` which is then stored in some other local variable.

Since both of my strings for `printf` were exactly same, the compiler reused it to call `printf` second time, instead of creating 2 strings with same content.

### Function pointers
Let us spice things up a little more now... and look at function pointers.

```c
void main()
{
    printf("%p\n", main);
    printf("%p\n", *main);
    printf("%p\n", &main);
}
```

Output of this program is not what you might expect if you are not a seasoned C programmer or have never worked with function pointers before.

Output:-

```txt
0x55770bbfa139
0x55770bbfa139
0x55770bbfa139
```

Each of them gave the same output. This is not the case when working with integer pointers. Let's see how this looks at lower level

Disassembly:-

```asm
.LC0:
        .string "%p\n"
main:
        push    rbp
        mov     rbp, rsp

        ; printf("%p\n", main);
        mov     esi, OFFSET FLAT:main
        mov     edi, OFFSET FLAT:.LC0
        mov     eax, 0
        call    printf

        ; printf("%p\n", *main);
        mov     esi, OFFSET FLAT:main
        mov     edi, OFFSET FLAT:.LC0
        mov     eax, 0
        call    printf

        ; printf("%p\n", &main);
        mov     esi, OFFSET FLAT:main
        mov     edi, OFFSET FLAT:.LC0
        mov     eax, 0
        call    printf

        nop
        pop     rbp
        ret
```

They are all precisely the same!! The assembly code for all three lines remains unchanged.

To test whether it behaves the same way with other functions as well, let's add another function to the code.


```c
#include <stdio.h>

int func() {}


void main()
{
    printf("%p\n", func);
    printf("%p\n", *func);
    printf("%p\n", &func);
}
```

Disassembly:-

```asm
func:
        push    rbp
        mov     rbp, rsp
        nop
        pop     rbp
        ret
.LC0:
        .string "%p\n"
main:
        push    rbp
        mov     rbp, rsp

        ; printf("%p\n", func);
        mov     esi, OFFSET FLAT:func
        mov     edi, OFFSET FLAT:.LC0
        mov     eax, 0
        call    printf

        ; printf("%p\n", *func);
        mov     esi, OFFSET FLAT:func
        mov     edi, OFFSET FLAT:.LC0
        mov     eax, 0
        call    printf

        ;printf("%p\n", &func);
        mov     esi, OFFSET FLAT:func
        mov     edi, OFFSET FLAT:.LC0
        mov     eax, 0
        call    printf

        nop
        pop     rbp
        ret
```

Still the same!!

If you have never worked with function pointers before, this is just the beginning of things, we can even call the function `func` using above pointer notations.

```c
#include <stdio.h>

void func() {}


void main()
{
    func();
    (*func)();
    (&func)();
}
```

Disassembly:-

```asm
func:
        push    rbp
        mov     rbp, rsp
        nop
        pop     rbp
        ret
main:
        push    rbp
        mov     rbp, rsp

        ; func();
        mov     eax, 0
        call    func

        ; (*func)();
        mov     eax, 0
        call    func

        ; (&func)();
        mov     eax, 0
        call    func

        nop
        pop     rbp
        ret
```

We can even add arguments to our `func` function call, just like we do with a normal function... and the first argument will be stored in `edi` the second on in `esi` and so on.

```c
int func(int x) {
    printf("%d\n", x);
}


void main()
{
    func(5);
    (func)(6);
    (*func)(7);
    (&func)(8);
}
```

Disassembly:-

```asm
.LC0:
        .string "%d\n"
func:
        push    rbp
        mov     rbp, rsp
        sub     rsp, 16
        mov     DWORD PTR [rbp-4], edi
        mov     eax, DWORD PTR [rbp-4]
        mov     esi, eax
        mov     edi, OFFSET FLAT:.LC0
        mov     eax, 0
        call    printf
        nop
        leave
        ret
main:
        push    rbp
        mov     rbp, rsp

        ; func(5);
        mov     edi, 5
        call    func

        ; func(6);
        mov     edi, 6
        call    func

        ; func(7);
        mov     edi, 7
        call    func

        ; func(8);
        mov     edi, 8
        call    func

        nop
        pop     rbp
        ret
```


That's it for today. In the next article, I'll try to use all our knowledge we have gathered till now to reverse engineer a very simple calculator program.

![](https://media.giphy.com/media/3ZZVfvtxywq9dIGZM0/giphy.gif#center)
