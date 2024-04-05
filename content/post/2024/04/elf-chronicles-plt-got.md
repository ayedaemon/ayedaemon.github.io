---
title: "Elf Chronicles: PLT/GOT (7/?)"
date: 2024-04-03T20:17:56+05:30
draft: false
# showtoc: false
tags:
    - "C"
    - "ELF"
    - "RE"
series:
    - "ELF Chronicles"
description: "Exploring general concepts of dynamic linking with PLT and GOT tables"
# math: true
# ShowBreadCrumbs: false
# searchHidden: true
# hideMeta: true
---

### Intro

In earlier articles, we talked about various parts of an ELF file and the many steps needed to create an executable ELF file that can run on your computer.

(Note: The steps are shown visually below; For the source code, check out the symbol table article in this series.)


```


     ┌────────────────────┐                        ┌─────────────────┐         ┌─────────────────┐
     │                    │                        │                 │         │                 │
     │   libarithmatic.c  │                        │ libarithmatic.h ├───────► │     main.c      │
     │                    │                        │                 │         │                 │
     └─────────┬──────────┘                        └─────────────────┘         └────────┬────────┘
               │                                                                        │
               │                                                                        │
               │ /* Compile + assemble */                                               │ /* Compile + assemble */
               │                                                                        │
               │                                                                        │
               ▼                                                                        ▼
    ┌─────────────────────┐                                                   ┌────────────────────┐
    │                     │                                                   │                    │
    │   libarithmatic.o   │                                                   │       main.o       │
    │                     │                                                   │                    │
    └─────────┬───────────┘                                                   └──────────┬─────────┘
              │                                                                          │
              │                                                                          │
              │                                                                          │
              │                                                                          │
              │                          /* Linking Magic */                             │
              └───────────────────────────────────┬──────────────────────────────────────┘
                                                  │
                                                  │
                                                  │
                                                  │
                                                  │
                                                  │
                                                  ▼
                                           ┌────────────────┐
                                           │                │
                                           │     calc       │
                                           │                │
                                           └────────────────┘
```

After completing this process, we have an ELF executable called `calc`. However, we didn't directly include any library that contains definitions for functions like `printf` or `scanf`, which we used in our `main.c` file to input and output data. So, how does that work?

Answer: **Dynamic linking** (which is a complex topic, so for this article, we'll just cover the basics).


If you use the `file` command on the `calc` executable, it will display interesting information such as `dynamically linked` and `interpreter /lib64/ld-linux-x86-64.so.2`.

```
> file calc
calc: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=65b929ceea26ea5e9fb8df1b15f2ab24b5c43ff6, for GNU/Linux 4.4.0, not stripped
```


Before we move forward, let's discuss some basics about libraries and how Linux manages them. Linux supports two types of libraries: *static* and *shared*.

*Static* libraries are connected to a program directly during the compile time (linking phase), while dynamic libraries (also known as shared libraries) are loaded when the application is launched, and all symbol resolutions and bindings are done at runtime.

*Dynamic* or *shared* libraries can be handled in two ways: Either you link your program with the shared library and let Linux load the library when the program runs (dynamic linking) [^baeldung_dynamic_linking], or you can design your application so that it loads the library from a specified path and then calls a particular function within that library (dynamic loading). [^baeldung_dynamic_loading]

[^baeldung_dynamic_linking]: https://www.baeldung.com/cs/dynamic-linking-vs-dynamic-loading#2-dynamic-linking

[^baeldung_dynamic_loading]: https://www.baeldung.com/cs/dynamic-linking-vs-dynamic-loading#loading


Now, looking at our `calc` binary, it's evident that we're using **dynamic linking** to handle functions like `printf` and `scanf` (Since we are not loading any other library in out code). If you have any background in C programming, you've likely heard of the standard C library (`libc`) at least once. `libc` contains definitions for many standard functions used by many C programs, including `printf` and `scanf`, which we need in our `calc` executable.

So, when we run the `calc` executable, Linux will figure out which libraries it needs to run and load them into the process memory space. Once that's done, it'll load the `calc` executable and resolve all the dynamic symbols it contains.

In newer systems, this loading and resolving process is done lazily. This means that libraries will only load and resolve when there's demand for a specific symbol. This approach is called lazy binding, and it helps speed up the loading of `calc` itself.

Since symbol resolution happens at runtime, the address of the resolved symbol needs to be stored somewhere so that we don't have to resolve it every time it's needed.

### GOT (GLobal Offset Table) and PLT (Procedure Linkage Table)

Let's visualize our situation: We need the address of the `printf` function to make a call, but we don't know where in the process memory space the `libc` library will load, so we can't determine the exact address for `printf`.

How can we call `printf` then?

One naive method would be to load `libc` into the process memory space, find the exact address for `printf` using `libc`'s base address, and then modify the `.text` section of `calc` to update the placeholder address of `printf` with the exact address. This seems straightforward and will work. However, with this approach, we'll have to load the library separately for each instance of `calc` or any other program that relies on `libc`. This isn't efficient because it would mean having many copies of the same library in memory, unless the library is completely read-only and never modified.

Another approach is to add a level of redirection to the this method. In this newer approach, we patch the `.got` and/or `.got.plt` section (which contains the Global Offset Table) of `calc`. The idea is that when the library is loaded, the dynamic linker examines the relocation, finds the exact address of `printf`, and patches the `.got` and/or `.got.plt` entry as required. Then, the `calc` binary refers to these tables to point to the right place. This way, everything works seamlessly!

**What does PLT do here ??**

The PLT (Procedure Linkage Table) adds another level of redirection that utilizes the `.got.plt` section to keep track of function jumps. Essentially, the Global Offset Table (GOT) is a list of addresses from the `libc`, while the PLT is another list of addresses used as placeholders in the `.text` section of the `calc` binary.


By utilizing this combination of the PLT and the `.got.plt` section, there's no need to directly patch the `.text` section of the `calc` binary. This approach offers security benefits as it avoids modifying the executable code, which could potentially introduce vulnerabilities or trigger security mechanisms designed to detect such modifications.

*Security benifits ++*


### Analysis

It will become clearer when we examine the disassembly (which is my favorite part).



As usual, we'll **disas**semble **main** function first. We don't have to check everything here, just focus on `printf` and `scanf` call instructions.

```
0x000055555555518f <+38>:    call   0x555555555050 <printf@plt>
0x00005555555551b2 <+73>:    call   0x555555555060 <__isoc99_scanf@plt>
```



Interesting thing to note here is that they point to addresses which are just *0x555555555060 - 0x555555555050 = `16`* bytes away from each other. I'm sure none of these functions can be defined in just 16 bytes.

This is the PLT stub, the area which is referred by `.text` section for all kinds of dynamic linked library calls.

```
(gdb) x/3i 0x555555555050
   0x555555555050 <printf@plt>:         jmp    QWORD PTR [rip+0x2fba]        # 0x555555558010 <printf@got.plt>
   0x555555555056 <printf@plt+6>:       push   0x2
   0x55555555505b <printf@plt+11>:      jmp    0x555555555020

(gdb) x/3i 0x555555555060
   0x555555555060 <__isoc99_scanf@plt>:         jmp    QWORD PTR [rip+0x2fb2]        # 0x555555558018 <__isoc99_scanf@got.plt>
   0x555555555066 <__isoc99_scanf@plt+6>:       push   0x3
   0x55555555506b <__isoc99_scanf@plt+11>:      jmp    0x555555555020

```

If you examine the first instructions in both, you'll notice they both point to memory locations `0x555555558010` and `0x555555558018`, which are the Global Offset Table (GOT) entries. These entries hold addresses of actual functions from the dynamic libraries. You can inspect these locations to find where the first instruction in the PLT stub is directing the jump to.

```
(gdb) x/1x 0x555555558010
0x555555558010 <printf@got.plt>:        0x55555056

(gdb) x/1x 0x555555558018
0x555555558018 <__isoc99_scanf@got.plt>:        0x55555066
```

Alright, since we're using these functions for the first time, the steps of finding the function's address and storing it in the Global Offset Table haven't been completed yet (which is part of the lazy binding logic). So, the program jumps to the next step instead (`0x55555056` and `0x55555066` respectively).


In next instruction from PLT stub, a certain number onto the stack... and then both of the PLT stubs jump to same address -- `0x555555555020`. This is the address which should trigger the dynamic symbol resolution process.


```
(gdb) x/3i 0x555555555020
=> 0x555555555020:      push   QWORD PTR [rip+0x2fca]        # 0x555555557ff0
   0x555555555026:      jmp    QWORD PTR [rip+0x2fcc]        # 0x555555557ff8
   0x55555555502c:      nop    DWORD PTR [rax+0x0]

```


It pushes something (`0x555555557ff0`) to stack and then jumps to `0x555555557ff8` address which is actually `_dl_runtime_resolve_xsavec` function in our dynamic linker (`/lib64/ld-linux-x86-64.so.2`). This function will resolve the address for `printf` and `scanf` and then patch the GOT table for it.


Once that is done, you can check the patched entries in the GOT tables

```
(gdb) x/1x 0x555555558010
0x555555558010 <printf@got.plt>:        0xf7e16730

(gdb) x/1x 0x555555558018
0x555555558018 <__isoc99_scanf@got.plt>:        0xf7e16430
```

Now, these are the real addresses for the `printf` and `scanf` functions within the `calc` program's memory space. With this completed, whenever `calc` needs to use `printf` or `scanf`, their addresses will already be stored in the GOT table, which can be accessed by the corresponding PLT stubs.



### Conclusion

In conclusion, dynamic linking plays a crucial role in how programs interact with shared libraries in Linux systems. By utilizing mechanisms like the Procedure Linkage Table (PLT) and the Global Offset Table (GOT), programs can efficiently access functions from shared libraries at runtime. This process involves lazy binding, where function addresses are resolved and stored in the GOT only when they are first called, optimizing performance and memory usage. Through this approach, programs like `calc` can seamlessly utilize functions like `printf` and `scanf` without the need for manual intervention or redundant loading of shared libraries. Overall, dynamic linking provides a flexible and efficient way for programs to access external functionality, enhancing the functionality and usability of software on Linux platforms.




---

### Resources
- https://developer.ibm.com/tutorials/l-dynamic-libraries/
- https://opensource.com/article/22/5/dynamic-linking-modular-libraries-linux
