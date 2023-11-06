---
title: "Intro to Re: C : part-1"
date: 2022-09-21T01:10:18+05:30
draft: false
# showtoc: false
tags: [RE, linux, C programming]
series: [Reverse Engineering]
description: Basics of assembly and its relation with higher level constucts
# math: true
# ShowBreadCrumbs: false
# searchHidden: true
# hideMeta: true
---

## Steps to generate a binary
When we write a program using a language like C, it is not C source code which really gets executed. This C code passes through many steps and finally a binary file is generated out of it. This binary file is what gets executed on any computer. 

There are many steps through which a C code is converted into a binary file:-
1. Pre-processing
2. Compilation
3. Assemble
4. Linking

Let's follow these steps one by one to understand what they do to the C code and how a binary is generated via this. To get started, we need a C program that we would want to convert into a binary.


```C
//file: hello_world.c

#include <stdio.h>

// main function
int main() {
    printf("Hello World");  // Print "Hello World"
    return 5;               // Return with 5 return value
}
```

If you are even a bit familiar with C programs, you would understand that the above program will create a `main()` function, call `printf()` function to print `Hello World` string and finally return with a `5` return value.

#### Pre-processing

Let's see what we get after **pre-processing** this C program. With gcc this can be done via below command

```bash
gcc -E hello_world.c -o hello_world.i
```

This takes the provided C program and does many things to it, few of these are mentioned below.

- Removes the comments.
- Replaces the `#include` statements with the actual file content. For example, `#include<stdio.h>` is replaced with `stdio.h` file contents.


#### Compilation

After pre-processing, the generated file is used to generate assembly instructions. These instructions are microprocessor (or CPU) specific. Microprocessor is a computer component that handles all kinds of conditional logic, arithmatic calculations and other logical activities.

You might have heard about few types of micro processor families like intel x86 and ARM.. but there are [many more](https://en.wikipedia.org/wiki/List_of_microprocessor). Unfortunately, each family has their own instruction sets to perform different tasks. 

We can convert our **pre-processed** code file to equivalent **assembly** code using `gcc`.

```bash
gcc -S hello_world.i -o hello_world.s
```

This will provide us with a `hello_world.s` file.

```bash
file hello_world.s

# hello_world.s: assembler source, ASCII text
```

This step produces an assembly language source code for my micro-processor family (intel x86-64). If you want to generate assembly code for other families (also called, architectures), you might want to look into [**cross-compilers**](https://en.wikipedia.org/wiki/Cross_compiler)

This assembly code is what get's executed by the processor. If we look into this file, we will be able to see the assembly code for the C program we wrote.

```bash
cat hello_world.s
```

Output:

```asm
        .file   "hello_world.c"
        .text
        .section        .rodata
.LC0:
        .string "Hello World"
        .text
        .globl  main
        .type   main, @function
main:
.LFB0:
        .cfi_startproc
        pushq   %rbp
        .cfi_def_cfa_offset 16
        .cfi_offset 6, -16
        movq    %rsp, %rbp
        .cfi_def_cfa_register 6
        leaq    .LC0(%rip), %rax
        movq    %rax, %rdi
        movl    $0, %eax
        call    printf@PLT
        movl    $5, %eax
        popq    %rbp
        .cfi_def_cfa 7, 8
        ret
        .cfi_endproc
.LFE0:
        .size   main, .-main
        .ident  "GCC: (GNU) 12.1.1 20220730"
        .section        .note.GNU-stack,"",@progbits
```

This is what the intermediate code in assembly language looks like. We'll talk more about this in later sections. For now, let's move forward and see how a binary file is created from this assembly code.

## Assembling into binary

We can convert the assembly code to binary file using `gcc` via the below command:-

```bash
gcc -c hello_world.s -o hello_world.o
```

This will generate the binary object file, which can be analyzed with tools like `objdump` and `hexdump`. This file is the binary file for the source code we wrote, but we need more than that for the program to actually execute on terminal with `./hello_world.o`.


#### Linking the binary

This last step will take your obect file (`.o` files from last step) and produce either a library or an executable file. It replaces the references to undefined symbols with the correct addresses. There are many more things going in here other than this, so to keep it short and simple - this step will create your executable from your object file by linking it with other required files like standard libraries. Ultimately, this will generate the executable or library which we can use or distribute it.

Using gcc, we can achieve this with:-

```bash
## -v option is just to print the verbose output.
gcc -v hello_world.o -o hello_world.out
```


After all these steps, we have our binary file and all other intermediate files.


```bash
file *

# hello_world.c:   C source, ASCII text

# hello_world.i:   C source, ASCII text

# hello_world.o:   ELF 64-bit LSB relocatable, x86-64, version 1 (SYSV), not stripped

# hello_world.out: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=c6293fa95c56250957a3babeecd8d18fc463e7cf, for GNU/Linux 4.4.0, with debug_info, not stripped

# hello_world.s:   assembler source, ASCII text
```

To summarize everything, the C program we write goes through various steps to generate an executable binary file. And this binary file is what executes on our machine....or in another words, this binary tells our processors about what job we want them to do.

![](https://media.giphy.com/media/oYtVHSxngR3lC/giphy.gif#center)


## Reverse Engineering??

Reverse Engineering is the process of figuring out how something works. This process can be applied to approximately anything even computer programs.

If we are provided with a compiled, assembled and linked binary file, we can apply the process of reverse engineering to understand what that program does. This skill can be useful for many tasks like generating keygens, patches, understanding low-level vulnerabilities, profiling applications, malware analysis, etc..

Since assembly language is the closest human-readable language for any binary, we need to have a good understanding of that to perform good reverse engineering. Obviously this varies with compilers used and architecture for which the binary was compiled... So we need might need to learn about how different compilers generate assembly code for different micro-processor families.

Most of the times we are provided with the binary file for programs instead of the source code, therefore, we only have assembly representation to work from. However there are tools like ghidra that takes the compiled assembly code and give us a view of what the C code for that binary might look like. This is called [decompilation](https://en.wikipedia.org/wiki/Decompiler).


#### binary <--> assembly (assembly <--> disassembly)

At this point, we know about how a C program gets converted to a binary file after the complete compilation process. Let us understand more about the relation between a binary file and it's assembly code.

We have already seen that the intermediate assembly code can be converted to binary file via assemblers (in our case, gcc assembler feature). The inverse can be done via [**disassemblers**](https://en.wikipedia.org/wiki/Disassembler).

A disassembler takes the binary file as input and read it's contents and maps the binary values to it's respective assembly instructions.

```goat
binary values --> byte pairs --> assembly code
```

There are many disassemblers we can use like:-
- [objdump](https://www.man7.org/linux/man-pages/man1/objdump.1.html)
- [ghidra](https://ghidra-sre.org/)
- [radare2](https://rada.re/n/radare2.html)
- [gdb](https://sourceware.org/gdb/)
- [hopper](https://www.hopperapp.com/)
- and [many more](https://en.wikipedia.org/wiki/Disassembler#Examples_of_disassemblers)


Let's try to disassemble our `hello_world.out` binary and see what it provides us.

```bash
objdump --disassemble  hello_world.out | wc -l

# 122
```

Objdump provides a lot of output for the disassembly of a simple "Hello Wold" program. This is because we linked the `hello_world.o` file to obtain a `hello_world.out` file. This new file has a lot of things that helps this to run on the machine with `./hello_world.out` command.

If we check the unlined version of our binary, we'll get a lot less output.

```bash
objdump --disassemble  hello_world.o | wc -l

# 16


## Looking at the disassembled binary
objdump --disassemble  hello_world.o


# 0000000000000000 <main>:
#    0:   55                      push   %rbp
#    1:   48 89 e5                mov    %rsp,%rbp
#    4:   48 8d 05 00 00 00 00    lea    0x0(%rip),%rax        # b <main+0xb>
#    b:   48 89 c7                mov    %rax,%rdi
#    e:   b8 00 00 00 00          mov    $0x0,%eax
#   13:   e8 00 00 00 00          call   18 <main+0x18>
#   18:   b8 05 00 00 00          mov    $0x5,%eax
#   1d:   5d                      pop    %rbp
#   1e:   c3                      ret
```

The same results can be obtained from our linked binary, if we only ask `objdump` to disassemble `main()` function.

```bash
objdump --disassemble=main hello_world.out


# 0000000000001139 <main>:
#     1139:       55                      push   %rbp
#     113a:       48 89 e5                mov    %rsp,%rbp
#     113d:       48 8d 05 c0 0e 00 00    lea    0xec0(%rip),%rax        # 2004 <_IO_stdin_used+0x4>
#     1144:       48 89 c7                mov    %rax,%rdi
#     1147:       b8 00 00 00 00          mov    $0x0,%eax
#     114c:       e8 df fe ff ff          call   1030 <printf@plt>
#     1151:       b8 05 00 00 00          mov    $0x5,%eax
#     1156:       5d                      pop    %rbp
#     1157:       c3                      ret
```

Ofcourse, few things are different like the numbers in first column (these numbers are offset values, we don't need to understand them right now).... But if we focus only on the *second column* and *third column* which are [**hexa-decimal values or opcodes**](https://en.wikipedia.org/wiki/Opcode) and **assembly code/representation** respectively.

The main difference between a `.o` and `.out` file is that the `.o` file is not yet linked to any platoform dependent libraries. Here is a [good stackoverflow thread](https://stackoverflow.com/questions/58245469/o-vs-out-in-c) about the same topic.

The assembly code has a syntax structure, which is a good point to start understanding how they work. Each line in the above output contains 1 assembly instruction ... and every instruction is composed of either 1, 2 or 3 keywords. These can be represented in below syntax.

```
## Syntax by Intel

| Keyword1     | Keyword2   | Keyword3   |
|--------------|------------|------------|
| OPCODE       | destination| source     |


    or 


## Syntax by AT&T

| Keyword1     | Keyword2  | Keyword3    |
|--------------|-----------|-------------|
| OPCODE       | source    | destination |

```

There are few other differences between these 2 syntaxes..but this does not change anything in logic or working of the binary. You can think of them as different kinds of representations for the same thing. Read more from [here: AT&T Syntax versus Intel Syntax](https://web.mit.edu/rhel-doc/3/rhel-as-en-3/i386-syntax.html) and [here: StackOverflow - NASM (Intel) versus AT&T Syntax: what are the advantages?](https://stackoverflow.com/questions/8549427/nasm-intel-versus-att-syntax-what-are-the-advantages#8550917). I prefer to use **Intel** syntax and will be using that throughout this article.


By default, `objdump` provides *AT&T* syntax but you can explicitely ask it to provide the *intel* syntax using `--disassembler-options=intel` flag.


```bash
objdump --disassemble=main --disassembler-options=intel hello_world.out
```
output

```objdump
0000000000001139 <main>:
    1139:       55                      push   rbp
    113a:       48 89 e5                mov    rbp,rsp
    113d:       48 8d 05 c0 0e 00 00    lea    rax,[rip+0xec0]        # 2004 <_IO_stdin_used+0x4>
    1144:       48 89 c7                mov    rdi,rax
    1147:       b8 00 00 00 00          mov    eax,0x0
    114c:       e8 df fe ff ff          call   1030 <printf@plt>
    1151:       b8 05 00 00 00          mov    eax,0x5
    1156:       5d                      pop    rbp
    1157:       c3                      ret

```

The purpose of higher level languages like C is that we don't have to deal with all this assembly code for all things. To effectively use assembly there are a lot of things we need to understand and think continously according to them. With higher level languages, we can write the code more easily and then pass that code to compiler, and that will generate a assembly code and binary file for us to use...


#### Heap, Stack and Registers

Every C process ([not program](https://www.geeksforgeeks.org/difference-between-program-and-process/)) use many things to work, 4 of them are - Heap, Stack, Registers and Instructions. We have just understood about the instructions in previous section, now let us start with others.

**Heap** is one of the memory alocation strategy used for dynamic storage allocations, ie, allocating memory at run time. The actual working of the heap is a bit complex and is out of scope for this article. For now, keep in mind that any calls to `malloc`, `calloc` or any other similar kind of function will allocate memory in heap. All objects which have dynamic storage duration are suitable for heap.

**Registers** are small storage areas in the processors, which are used by instructions to store multiple values. They can store anything upto their size limits. Different architectures have different size of registers ranging from 8 bit to 64 bit registers. There are even 128, 256, and 512-bit registers. [Here is a stackoverflow thread](https://stackoverflow.com/questions/52932539/what-are-the-128-bit-to-512-bit-registers-used-for) for the same.


Back in early days (1972), Intel added few 8-bit general purpose registers to their microprocessors...general purpose registers are used for general purposes like storing return values, temporary calculation results, etc.. 

```goat

  ┌─────────┐
  │    A    │
  ├─────────┤
  │    B    │
  ├─────────┤
  │    C    │
  ├─────────┤
  │    D    │
  ├─────────┤
```

Later these 8-bit registers were updated with 16-bit registers... This was logically partitioned into two 8-bit registers, maybe to back-support the old systems/softwares that used needed 8 bit registers to work, Anyways, the new registers looked something like this...


```goat

      ◄──8-bit─► ◄─8-bit──►               ◄───── 16-bit──────►

      ┌─────────┬─────────┐               ┌───────────────────┐
      │    AH   │    AL   │               │        AX         │
      ├─────────┼─────────┤               ├───────────────────┤
      │    BH   │    BL   │               │        BX         │
      ├─────────┼─────────┤      OR       ├───────────────────┤
      │    CH   │    CL   │               │        CX         │
      ├─────────┼─────────┤               ├───────────────────┤
      │    DH   │    DL   │               │        DX         │
      └─────────┴─────────┘               └───────────────────┘

```

These 16-bit registers were partitioned into 2 sections, higher address (H) and lower address (L).

Following that, few years later, 16-bit registers were extended to support 32 bit softwares, adding an `E` prefix. And after that adding a `R` prefix for 64 bit registers.

Diagram below shows how these registers are mapped to support previous designs/architectures.

```goat

                                                                       ◄─8-bit──►

                                                                      ┌─────────┐
                                                                      │    A    │
                                                                      └─────────┘


                                                             ◄───── 16-bit(AX)──►

                                                             ┌─────────┬─────────┐
                                                             │    AH   │    AL   │
                                                             └─────────┴─────────┘

                                         ◄────────────── 32 bit (EAX)───────────►

                                         ┌───────────────────────────────────────┐
                                         │                   EAX                 │
                                         └───────────────────────────────────────┘


 ◄──────────────────────────────────  64 - bit (RAX)────────────────────────────►

 ┌───────────────────────────────────────┬───────────────────────────────────────┐
 │  (This 32-bit section has no name)    │                   EAX                 │
 └───────────────────────────────────────┴───────────────────────────────────────┘

```

Nowadays, 64-bit and 32-bit systems are very common and can be easily found. Knowing this is very important to reverse engineer arithmatic and logic calculations from any disassembly code.

Apart from these general register, there are 3 special purpose registers - `ebp`, `esp` and `eip`. These registers are generally used to point to different memory locations of the stack. There are some times when we need to store the values of these registers on to the stack (Another kind of memory area used by processes). We'll see some of those cases as we go further.

**Stack** is a data structure in memory which operates with 2 operations - `push` and `pop`. *Push* adds an element to the top of the stack and `pop` removes the top element of the stack.

Each element on the stack has an assigned stack address which can be used to refer any location on the stack. The stack is upside down - means that the stack grows towards the lower memory addresses.

```goat


        High Address

                                                         │
                                ┌─────────────┐          │
                            ▲   │             │          │
                            │   │             │          │
                            │   │  Func 1     │          │
                  Stack Frame 1 │             │          │
                            │   │             │          │
                            │   │             │          │
                            ▼   ├─────────────┤          │
                         ▲      │             │          │    Stack
                         │      │             │          │
                         │      │   Func 2    │          │    Grows
                 Stack Frame 2  │             │          │
                         │      │             │          │    Towards
                         │      │             │          │
                         ▼      ├─────────────┤          │    Low Address
                            ▲   │             │          │
                            │   │             │          │
                            │   │   Func 3    │          │
                  Stack Frame 3 │             │          │
                            │   │             │          │
                            │   │             │          │
                            ▼   ├─────────────┤          │
                                │             │          │
                                │             │          │
                                │             │          │
                                │             │          ▼

         Low Address

```

Whenever a function is called it creates its stack frame, and all the local variables for that function will be stored in that function's stack frame. This introduces the need to track 2 values for the current stack frame.

1. What is the base of the stack frame? Where did the stack frame start? 
2. What is the top most location of the stack frame? How much the stack has grown?

These are tracked by 2 special purpose registers - `rbp` (64-bit base pointer register) and `rsp` (64-bit stack pointer register) respectively.


```goat

   High Address


                              ┌─────────────┐
                          ▲   │             │
                          │   │             │
                          │   │  Func 1     │
                Stack Frame 1 │             │
                          │   │             │
                          │   │             │
                          ▼   ├─────────────┤
                       ▲      │             │
                       │      │             │
                       │      │   Func 2    │
               Stack Frame 2  │             │
                       │      │             │
                       │      │             │
                       ▼      ├─────────────┤
                          ▲   │             │
                          │   │             │
                          │   │   Func 3    │
                Stack Frame 3 │             │
                          │   │             │
                          │   │             │
                          ▼   ├─────────────┤  ◄─────  rbp
                        ▲     │             │
                        │     │             │
                        │     │   Func 4    │
                Stack Frame 4 │             │
                        │                      ◄─────  rsp
                        │
                        ▼
   Low Address

```

Let's take another example to understand how this would work for a regular C program.

```c
#include <stdio.h>

int func2(int a, int b, int c, int d, int e, int f, int g, int h) {
    int z = 0;
    int sum = a + b + c + d + e + f + g + h;
    char ch = 'A';
    return sum;
}

void func1() {
    int x  = func2(1, 2, 3, 4, 5, 6, 7, 8);
}

int main(){
    func1();
}
```

This is roughly what the execution flow of the above program will look like... 


1. `main()` calls `func1()` function without any arguments.
2. `func1()` function allocates some memory for `int x` and then calls `func2()` functions with a single integer type argument.
3. `func2()` function creates some local variables - both with hardcoded value and using passed arguments.
4. And then returns a variable back to `func1()`.
5. Finally the control is passed back to `main()`.

If you visualise the stack just before the `main()` function is loaded on stack.. the stack will be something like this:-

```goat

## Before main started executing

                       ┌───────────────────┐   ◄───────  rbp
                       │ * * * *           │
    Stack frame for    │ * * * *           │
    previous function  │                   │   ◄───────  rsp  
                       │                   │

```

After this, the `main()` function loads its variables on the stack.. this will cause the return address to be loaded on to the stack, which will be used once the `main()` function has completed it's execution... This is the address which tells where to go back once the `main()` function returns. Visually the stack will look something like this.


```goat
                     ┌───────────────────┐
                     │ * * * *           │
  Stack frame for    │ * * * *           │
  previous function  │                   │
                     │ return address    │  ◄───────  rbp  ◄───────  rsp 
                     │                   │
```

Notice that the `rbp` and `rsp` are also moved... `rsp` will move where the stack's top is... and `rbp` will point to the base of the current stack frame. This is what the loading of a function looks like. This is called [**Prologue**](https://en.wikipedia.org/wiki/Function_prologue_and_epilogue#Prologue)

Now the `rip` (64-bit instruction pointer) will point to `main()` function's code block and will execute those instructions one by one. This will push more values to stack as required... not all things are needed to be added to stack. We only push those values to stack which we need to save for later use and then pop when it is no longer required.

Looking at the disassembly of `main()` function will help us to understand more what will be added to stack.

```asm
0000000000001153 <main>:
    1153:       55                      push   rbp
    1154:       48 89 e5                mov    rbp,rsp
    1157:       b8 00 00 00 00          mov    eax,0x0
    115c:       e8 da ff ff ff          call   113b <func1>
    1161:       b8 00 00 00 00          mov    eax,0x0
    1166:       5d                      pop    rbp
    1167:       c3                      ret
```

Here, the above 2 lines make what we call **Prologue**. This pushes previous `rbp` to stack (stores returning point) and then updates the `rbp` with current `rsp` (stack pointer) value.

then it moves `0x00` to `eax` register... Purpose of this is to reset `eax` register as `0` so that when the `func1` returns, we are sure that it is not a garbage value. *Remember:- `eax` is another general purpose register that is used by called functions to save the return value. This then can be read by the caller function to know what that function returned.*

After reseting `eax`, it `call`s `func1()` function... this will transfer the control to `func1()`. 

```asm

000000000000116a <func1>:
    116a:       55                      push   rbp
    116b:       48 89 e5                mov    rbp,rsp
    116e:       48 83 ec 10             sub    rsp,0x10
    1172:       6a 08                   push   0x8
    1174:       6a 07                   push   0x7
    1176:       41 b9 06 00 00 00       mov    r9d,0x6
    117c:       41 b8 05 00 00 00       mov    r8d,0x5
    1182:       b9 04 00 00 00          mov    ecx,0x4
    1187:       ba 03 00 00 00          mov    edx,0x3
    118c:       be 02 00 00 00          mov    esi,0x2
    1191:       bf 01 00 00 00          mov    edi,0x1
    1196:       e8 7e ff ff ff          call   1119 <func2>
    119b:       48 83 c4 10             add    rsp,0x10
    119f:       89 45 fc                mov    DWORD PTR [rbp-0x4],eax
    11a2:       90                      nop
    11a3:       c9                      leave
    11a4:       c3                      ret
```

`func1()` will then store the previous (main function's) `rbp` value to stack.. and update the new base pointer with stack pointer's value. At this point our stack will look something like below:-


```goat

    ┌────────────────────────────────┐
    │                                │
    │  * * * *                       │
    │                                │
    │  * * * *                       │
    │                                │
    │  return address of prev func   │
    │                                │
    │  return address of main func   │ ◄───────  rbp   ◄───────  rsp
    │                                │
    │                                │
```

Now after the prologue instructions, the instruction pointer (`rip`) is at instruction `113f` --> `sub    rsp,0x10`. This subtracts `0x10` from `rsp` that will increase the gap betweem `rbp` and `rsp` resulting in some memory space in the stack frame. This does not overwrite the stack values, but this will create a sense of cleaning the messed up stack.


```goat


    high Addr       ┌────────────────────────────────┐
                    │                                │
                    │  * * * *                       │
                    │                                │
                    │  * * * *                       │
                    │                                │
                    │  return address of prev func   │
                    │                                │
                    │  return address of main func   │ ◄───────  rbp  ..
                    │                                │                 .
                    │                                │                 .  0x10
                    │                                │                 .
                    │                                │ ◄───────  rsp  ..
                    │                                │
                    │                                │
      Low Addr      │                                │
                    │                                │
                    │                                │
                    │                                │
                    │                                │
                    │                                │


```


This allocated space is now used to create local variables for the function `func1()`... For our case, that will be `int x`. Integers use only 4 bytes of space for themselves, but `gcc` by default allocates memory in 16-bytes chunk. That is the reason for `rsp` to move `0x10` (16) bytes downwards. We can change this behaviour by explicitely passign  `-mpreferred-stack-boundary=n` flag to `gcc`. Read more about this [here: Stack allocation, padding, and alignment](https://stackoverflow.com/questions/1061818/stack-allocation-padding-and-alignment)


After the memory is allocated on stack for local variables, it is time to call func2 with all the arguments. According to the calling convention defined for x86 assembly instructions, whenever a function is called, it's arguments are first loaded into predefined registers in specific positional order. But due to limitations of general purpose registers, only first 6 arguments are loaded to registers and the rest are pushed to stack.

For this case, there are in total 8 arguments... out of which 6 will be stored in the general purpose registers and the rest 2 will be pushed to stack.


```goat


high Addr       ┌────────────────────────────────┐
                │                                │
                │  * * * *                       │
                │                                │
                │  * * * *                       │
                │                                │
                │  return address of prev func   │
                │                                │
                │  return address of main func   │ ◄───────  rbp  ..
                │                                │                 .
                │                                │                 .  0x10
                │                                │                 .
                │                                │..................
                │  0x8                           │
                │                                │
Low Addr        │  0x7                           │ ◄───────  rsp
                │                                │
                │                                │
                │                                │
                │                                │
                │                                │


```


When more values are pushed to stack, the stack pointer moves more towards lower addresses and the stack frame grows. Here these 2 values are added to stack and then the function is called.... this will cause the return address to be pushed to stack too (So that control can come back to this location.)


```goat


 high Addr       ┌────────────────────────────────┐
                 │                                │
                 │  * * * *                       │
                 │                                │
                 │  * * * *                       │
                 │                                │
                 │  return address of prev func   │
                 │                                │
                 │  return address of main func   │ ◄───────  rbp  ..
                 │                                │                 .
                 │                                │                 .  0x10
                 │                                │                 .
                 │                                │..................
                 │  0x8                           │
                 │                                │
   Low Addr      │  0x7                           │
                 │                                │
                 │  return address of func1 func  │ ◄───────  rsp
                 │                                │
                 │                                │
                 │                                │

```


After this, the instruction pointer starts pointing towards the instructions of the `func2` function.


```asm
0000000000001119 <func2>:
    1119:       55                      push   rbp
    111a:       48 89 e5                mov    rbp,rsp
    111d:       89 7d ec                mov    DWORD PTR [rbp-0x14],edi
    1120:       89 75 e8                mov    DWORD PTR [rbp-0x18],esi
    1123:       89 55 e4                mov    DWORD PTR [rbp-0x1c],edx
    1126:       89 4d e0                mov    DWORD PTR [rbp-0x20],ecx
    1129:       44 89 45 dc             mov    DWORD PTR [rbp-0x24],r8d
    112d:       44 89 4d d8             mov    DWORD PTR [rbp-0x28],r9d
    1131:       c7 45 f8 00 00 00 00    mov    DWORD PTR [rbp-0x8],0x0
    1138:       8b 55 ec                mov    edx,DWORD PTR [rbp-0x14]
    113b:       8b 45 e8                mov    eax,DWORD PTR [rbp-0x18]
    113e:       01 c2                   add    edx,eax
    1140:       8b 45 e4                mov    eax,DWORD PTR [rbp-0x1c]
    1143:       01 c2                   add    edx,eax
    1145:       8b 45 e0                mov    eax,DWORD PTR [rbp-0x20]
    1148:       01 c2                   add    edx,eax
    114a:       8b 45 dc                mov    eax,DWORD PTR [rbp-0x24]
    114d:       01 c2                   add    edx,eax
    114f:       8b 45 d8                mov    eax,DWORD PTR [rbp-0x28]
    1152:       01 c2                   add    edx,eax
    1154:       8b 45 10                mov    eax,DWORD PTR [rbp+0x10]
    1157:       01 c2                   add    edx,eax
    1159:       8b 45 18                mov    eax,DWORD PTR [rbp+0x18]
    115c:       01 d0                   add    eax,edx
    115e:       89 45 fc                mov    DWORD PTR [rbp-0x4],eax
    1161:       c6 45 f7 41             mov    BYTE PTR [rbp-0x9],0x41
    1165:       8b 45 fc                mov    eax,DWORD PTR [rbp-0x4]
    1168:       5d                      pop    rbp
    1169:       c3                      ret
```


This is the biggest function we have seen so far... We already know the *prologue* that covers the first 2 instructions of this assembly code. Let's read and try to understand further instructions. Before starting that, let's see how our stack looks like after this function's prologue.

```goat

  high Addr       ┌────────────────────────────────┐
                  │                                │
                  │  * * * *                       │
                  │                                │
                  │  * * * *                       │
                  │                                │
                  │  return address of prev func   │
                  │                                │
                  │  return address of main func   │....
                  │                                │   .
                  │                                │   .  0x10
                  │                                │   .
                  │                                │....
                  │  0x8                           │
                  │                                │
    Low Addr      │  0x7                           │
                  │                                │
                  │  return address of func1 func  │ ◄───────  rsp    ◄───────  rbp
                  │                                │
                  │                                │
                  │                                │

```


It is a good idea to visualize how the stack will look after this function has been loaded and all the required memory is allocated to it.

```goat

    HIGH ADDR               ┌────────────────────────────────┐
                            │                                │
                            │  * * * *                       │
                            │                                │
                            │  * * * *                       │
                            │                                │
                            │  return address of prev func   │
                            │                                │
                            │  return address of main func   │....
                            │                                │   .
                            │                                │   .  0x10
                            │                                │   .
                            │                                │....
                            │  0x8                           │
                            │                                │
                            │  0x7                           │
                            │                                │
                            │  return address of func1 func  │ ◄───────  rbp
                            │                                │
                rbp - 0x04  │                                │
                            │                                │
                rbp - 0x08  │                                │
                            │                                │
                rbp - 0x09  │                                │
                            │                                │
                            │                                │
                            │                                │
                rbp - 0x14  │                                │.....
                            │                                │    .
                            │                                │    .
                            │                                │    .
                            │                                │    .  Local variables for passed arguments
                            │                                │    .
                            │                                │    .
                rbp - 0x28  │                                │    .
                            │                                │.....  
                            │                                │
                            │                                │

LOW ADDR

```

Now we can understand easily which variables are stored where on the stack. The previously passed positional arguments are loaded to stack after the local variables.

```asm

111d:       89 7d ec                mov    DWORD PTR [rbp-0x14],edi         ; 0x1
1120:       89 75 e8                mov    DWORD PTR [rbp-0x18],esi         ; 0x2
1123:       89 55 e4                mov    DWORD PTR [rbp-0x1c],edx         ; 0x3
1126:       89 4d e0                mov    DWORD PTR [rbp-0x20],ecx         ; 0x4
1129:       44 89 45 dc             mov    DWORD PTR [rbp-0x24],r8d         ; 0x5
112d:       44 89 4d d8             mov    DWORD PTR [rbp-0x28],r9d         ; 0x6
```

These locations are in continous order with 4-bytes of memory space for each integer. This is the optimization done by `gcc` because it knows what will be the size of the passed arguments.


```asm
1131:       c7 45 f8 00 00 00 00    mov    DWORD PTR [rbp-0x8],0x0
```

Then, `0x0` is stored to `rbp-0x8`. This will be one of our local variables. Then it adds all the passed arguments and save that to another local variable.

```asm
    1138:       8b 55 ec                mov    edx,DWORD PTR [rbp-0x14]
    113b:       8b 45 e8                mov    eax,DWORD PTR [rbp-0x18]
    113e:       01 c2                   add    edx,eax
; var_edx = 0x1 + 0x2

    1140:       8b 45 e4                mov    eax,DWORD PTR [rbp-0x1c]
    1143:       01 c2                   add    edx,eax
; var_edx = var_edx + 0x3

    1145:       8b 45 e0                mov    eax,DWORD PTR [rbp-0x20]
    1148:       01 c2                   add    edx,eax
; var_edx = var_edx + 0x4

    114a:       8b 45 dc                mov    eax,DWORD PTR [rbp-0x24]
    114d:       01 c2                   add    edx,eax
; var_edx = var_edx + 0x5

    114f:       8b 45 d8                mov    eax,DWORD PTR [rbp-0x28]
    1152:       01 c2                   add    edx,eax
; var_edx = var_edx + 0x6

    1154:       8b 45 10                mov    eax,DWORD PTR [rbp+0x10]
    1157:       01 c2                   add    edx,eax
; var_edx = var_edx + 0x7

    1159:       8b 45 18                mov    eax,DWORD PTR [rbp+0x18]
    115c:       01 d0                   add    eax,edx
; var_eax = var_edx + 0x8

    115e:       89 45 fc                mov    DWORD PTR [rbp-0x4],eax
; int sum = var_eax
```

Once all the arguments are added it stores that to another local variable. Another point to note here is that the first 6 arguments were stored to stack from general purpose registers and the rest 2 arguments (that were already on stack) were directly referenced from the stack... These values were added to stack before the previous function return value so they are referenced by `rbp+0x10` and `rbp+18`.

Visually this part of the stack will look something like this...

```goat

             │                                │   .
             │                                │....
rbp + 0x18   │  0x8                           │
             │                                │
rbp + 0x10   │  0x7                           │
             │                                │
rbp + 0x08   │  return address of func1 func  │ 
             │                                │ ◄───────  rbp
rbp - 0x04   │                                │
```


Now comes the third local variable, `char ch='A'`... This is the next instruction in our disassembly

```asm
  1161:       c6 45 f7 41             mov    BYTE PTR [rbp-0x9],0x41   ; 0x41 = 65 = A
```


Now we can take a look again to the stack and understand what memory locations are for what purposes...

```goat


             high Addr               ┌────────────────────────────────┐
                                     │                                │
                                     │  * * * *                       │
                                     │                                │
                                     │  * * * *                       │
                                     │                                │
                                     │  return address of prev func   │
                                     │                                │
                                     │  return address of main func   │....
                                     │                                │   .
                                     │                                │   .  0x10
                                     │                                │   .
                                     │                                │....
                                     │  0x8                           │
                                     │                                │
                                     │  0x7                           │
                                     │                                │
                                     │  return address of func1 func  │
                                     │                                │ ◄───────  rbp
      (First local var) rbp - 0x04   │   36                           │
                                     │                                │
     (Second local var) rbp - 0x08   │   0                            │
                                     │                                │
      (Third local var) rbp - 0x09   │   A                            │
                                     │                                │
                                     │                                │
                                     │                                │
                        rbp - 0x14   │  0x1                           │.....
                                     │                                │    .
                                     │  0x2                           │    .
                                     │                                │    .
                                     │  0x3                           │    .  Local variables for passed arguments
                                     │                                │    .
                                     │  0x4                           │    .
                                     │                                │    .
                                     │  0x5                           │    .
                                     │                                │    .
                        rbp - 0x28   │  0x6                           │.....   

           Low Addr



```

Then we set `eax` to the value we want to return back to the caller function `func1()`.

```asm
    1165:       8b 45 fc                mov    eax,DWORD PTR [rbp-0x4]
```

And finally, we pop the `rbp` from stack and move to the instruction in the previous function from where we left the execution. This is called [**Epilogue**](https://en.wikipedia.org/wiki/Function_prologue_and_epilogue#Epilogue)

```asm
    1168:       5d                      pop    rbp
    1169:       c3                      ret
```

After this, the stack will look something like as shown below:

```goat

 high Addr       ┌────────────────────────────────┐
                 │                                │
                 │  * * * *                       │
                 │                                │
                 │  * * * *                       │
                 │                                │
                 │  return address of prev func   │
                 │                                │
                 │  return address of main func   │ ◄───────  rbp  ..
                 │                                │                 .
                 │                                │                 .  0x10
                 │                                │                 .
                 │                                │..................
                 │  0x8                           │
                 │                                │
   Low Addr      │  0x7                           │  ◄───────  rsp
                 │                                │
                 │                                │ 

```


And in disassembly code, Instruction pointer will be pointing to the next instruction after `func2()` function call.


```asm

000000000000116a <func1>:
    116a:       55                      push   rbp
    116b:       48 89 e5                mov    rbp,rsp
    116e:       48 83 ec 10             sub    rsp,0x10
    1172:       6a 08                   push   0x8
    1174:       6a 07                   push   0x7
    1176:       41 b9 06 00 00 00       mov    r9d,0x6
    117c:       41 b8 05 00 00 00       mov    r8d,0x5
    1182:       b9 04 00 00 00          mov    ecx,0x4
    1187:       ba 03 00 00 00          mov    edx,0x3
    118c:       be 02 00 00 00          mov    esi,0x2
    1191:       bf 01 00 00 00          mov    edi,0x1
    1196:       e8 7e ff ff ff          call   1119 <func2>
    119b:       48 83 c4 10             add    rsp,0x10                         <--- instruction pointer
    119f:       89 45 fc                mov    DWORD PTR [rbp-0x4],eax
    11a2:       90                      nop
    11a3:       c9                      leave
    11a4:       c3                      ret

```

This will then increase the stack pointer by `0x10` and save return value to a local variable at `rbp-0x4` location...and then exit. This is *epilogue* for `func1()` function. And after `leave` and `ret` instructions the stack will look like as shown below:

```goat

     high Addr               ┌────────────────────────────────┐    ◄───────  rbp
                             │                                │
                             │  * * * *                       │
                             │                                │
                             │  * * * *                       │
                             │                                │
                             │  return address of prev func   │    
                             │                                │    ◄───────  rsp
     Low Addr


```


And instruction pointer will be pointing to the next instruction in `main()` function.

```asm
00000000000011a5 <main>:
    11a5:       55                      push   rbp
    11a6:       48 89 e5                mov    rbp,rsp
    11a9:       b8 00 00 00 00          mov    eax,0x0
    11ae:       e8 b7 ff ff ff          call   116a <func1>
    11b3:       b8 00 00 00 00          mov    eax,0x0           <--- instruction pointer
    11b8:       5d                      pop    rbp
    11b9:       c3                      ret

```

This instruction will simply set the return value for this function as `0x0` and then will pop `rbp` from stack. This will collapse the `main()` function's stack frame and in the next instruction it'll return the control to whatever function called the `main()` function.

![](https://media.giphy.com/media/giVvDFs5deYrKCKpD4/giphy.gif#center)

## Some most common instructions

Most of the times, you'll see similar instructions being executed like `push`, `pop`, `ret`, `mov`, `add`, `sub`, etc.. Let's understand some of those which we will be seeing most of the times.


1. `push` : This simply adds the value to the top of the stack and then decrements the stack pointer. (Decrementing stack pointer means growing the stack, since stack is upside down and grows towards lower addresses)

2. `pop` : Opposite to **push** instruction, this removes the top of the stack and increments the stack pointer.

3. `mov` : This is used to move some values from one location to another. This instruction is quite versatile and can move the values from/to register, stack memory locations, etc... For example, 

 - `mov rax, 0x10` will move `0x10` constant value to `rax` register. 

 - `mov rdi, rax` will move the `rax` register value to `rdi` register.

 - `mov rdi, [rax]` will move the value pointer by `rax` to `rdi` register. You can think of this as dereference pointer in C.

 - `mov [rbp-0x8], rax` will move the value from `rax` register to memory location poined by `rbp-0x08`.


4. `add` and `sub` : adds and substracts one value from another value. For example, 

 - `add rax, 0x10` will add `0x10` to value at `rax` register. And the result will be stored in `rax` register.
 - `sub rax, 0x10` wil subtract `0x10` from value at `rax` register. And the result will be stored in `rax` register.

5. `lea` : this Loads Effective Address (lea) to any destination. This is used to copy the address of the memory location to a register/stack. For example, `lea eax, rbp-0x8` will load the address of `rbp-0x8` into `eax` register.

6. `cmp` : This (compare instruction) is equivalent to **sub** instruction, just instead of saving the result into the first argument, it updates a flag. If the value is less then 0 then the flag is set to 1 else 0. For example, `cmp 1, 3` will result in `-2` and this will set the flag to be 1.

7. `jmp` : Compare instructions are usually followed by jump instructions. This will check the above set flag and jump to the address specified in the argument accordingly. There are many types of jump instructions like jump equal (`je`), jump not equal (`je`), jump greater (`jg`), jump less (`jl`), etc.. This instruction actually manipulates the instruction pointer to make the jump. If the condation matches then it'll take the jump by setting the instruction pointer to the memory location from the argument, else it'll change the instruction pointer to point to the next statement.

8. `call` : This instruction calls a function. This is equivalent to `push eip` (save the next instruction on stack) followed by `jmp func`.

9. `leave/ret` : This is called at the end of every function. This destroys the current stack frame by incrementing the stack pointer or by moving the stack pointer(`rsp`) to the same location pointer by base pointer (`rbp`) and then poping the base pointer. This will make the previous return address the top of the stack which will be used by `ret` to return to that address by setting `eip` to that address and eventually pop the return address from the stack.


## Conclusion

You're all set! This all could be a lot to take in all at once. But atleast you now have a rudimentary grasp of how a C code is compiled and how everything functions at the low level. I hope you now have enough information and self-assurance to begin your adventures into reverse engineering. 

Have fun!! ✌️
