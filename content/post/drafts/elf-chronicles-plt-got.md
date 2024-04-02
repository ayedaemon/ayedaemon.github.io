---
title: "Elf Chronicles: PLT/GOT (7/?)"
date: 2023-12-08T14:17:56+05:30
draft: true
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


In previous articles, we have seen different aspects of an ELF file and multiple steps that are involved in production of the ELF executable that can execute on your system.

(Note: Below is a visual image of those steps; For the source code refer to the symbol table article in this series)


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

Once this process is completed we have an ELF executable named `calc`. But we never linked any library which might have `printf` or `scanf` function definition.... but we surely used it in our `main.c` to take input and print the output. So how does that work???


Answer: **Dynamic linking** (which is a very huge topic in itself, so for the sake of this article we'll just try to see the high level view of it)


```
> file calc
calc: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=65b929ceea26ea5e9fb8df1b15f2ab24b5c43ff6, for GNU/Linux 4.4.0, not stripped
```


If you run `file` command on the `calc` executable, it'll show you some interesting information like `dynamically linked` and `interpreter /lib64/ld-linux-x86-64.so.2`.


Before we proceed with anything, let's cover up some prerequisties about libraries and how linux handles them.

Linux supports 2 types of libraries... *static* and *shared*.

Static libraries are bound to a program statically at compile time (during linking phase)... while dynamic libraries (shared libraries) are loaded when the application is loaded and all symbol resolutions and symbol bindings are done at run time.

Dynamic libraries can be handled in one of the 2 ways... either you link your program with the shared library and let linux load the library upon execution (dynamic linking) OR you can program your application in a way that it loads the library from a given path and then calls a particular function within that library (dynamic loading).



Now you can clearly see, in our `calc` binary we are dealing with **dynamic linking** to deal with functions like `printf` and `scanf`. If you have any background with C pogramming, you must already have heard about standard C library (`libc`) atleast once. **libc** have definitions for many standard functions used by many C programs... 2 of such function are `printf` and `scanf` which we need in our `calc` executable.

So in a nutshell, when we execute `calc` executable, linux will resolve the libraries needed for it to run and will load those libraries in process memory space. Once that is done, it'll load the `calc` executable and then resolve all of the dynamic symbols present.

In newer systems, this process of loading and resolving is done in a lazy manner. The libraries will load and resolve whenever there is demand created for it. This approach is called lazy binding. This will helps in faster loading of the `calc`.

Now since the symbol resolution will be done at runtime, the address of the resolved symbol should be stored somewhere so that we don't actually have to resolve symbol everytime it is required.


### GOT (GLobal Offset Table) and PLT (Procedure Linkage Table)

Visualize our situation here, we need address of `printf` function to make a call, but we don't know where in process memory space will the `libc` library will load and hence can't figure out the exact address for `printf` function.

Now how do we handle this??

One of the very naive method will be to load the `libc` in process memory space(dynamic loader will find a convinient place for it and load it), find exact address for `printf` function using `libc`'s base address, then modify the `.text` section to update the placeholder address of `printf` with the exact address.

This looks simple and will work. But with this approach we'll have to load the library separately for each instance of `calc` or any other program that will rely on `libc` like this. It does not make sense to have so many copies of the same library in memory, unless the library is completely read-only and is never modified.

Another approach will be to add a level of redirection to the previous approach.

This newer approach relies on patching the `.got` and/or `.got.plt` section (which contains Global Offset Table). The idea is when the library is loaded, the dynamic linker will examine the relocation, go and find the exact address of `printf`, and patch `.got` and/or `.got.plt` entry as required. And then the `calc` binary will refer to these tables to point to the right place. Everything just works!

**What does PLT do here ??**

Well, PLT adds another level of redirection that uses `.got.plt` section to keep tracks of function jumps. Basically GOT is just a list of addresses from the `libc` and PLT is another list of addresses which are used as placeholders in `.text` section of the `calc`.


### Analysis

It'll make a better sense when you'll look at the disassembly (My fav part)



- Disas main


```
(gdb) disas main
Dump of assembler code for function main:
   0x0000000000001169 <+0>:     push   rbp
   0x000000000000116a <+1>:     mov    rbp,rsp
   0x000000000000116d <+4>:     sub    rsp,0x20
   0x0000000000001171 <+8>:     mov    rax,QWORD PTR fs:0x28
   0x000000000000117a <+17>:    mov    QWORD PTR [rbp-0x8],rax
   0x000000000000117e <+21>:    xor    eax,eax
   0x0000000000001180 <+23>:    lea    rax,[rip+0xe7d]        # 0x2004
   0x0000000000001187 <+30>:    mov    rdi,rax
   0x000000000000118a <+33>:    mov    eax,0x0
   0x000000000000118f <+38>:    call   0x1050 <printf@plt>
   0x0000000000001194 <+43>:    lea    rcx,[rbp-0x10]
   0x0000000000001198 <+47>:    lea    rdx,[rbp-0x15]
   0x000000000000119c <+51>:    lea    rax,[rbp-0x14]
   0x00000000000011a0 <+55>:    mov    rsi,rax
   0x00000000000011a3 <+58>:    lea    rax,[rip+0xe73]        # 0x201d
   0x00000000000011aa <+65>:    mov    rdi,rax
   0x00000000000011ad <+68>:    mov    eax,0x0
   0x00000000000011b2 <+73>:    call   0x1060 <__isoc99_scanf@plt>
   0x00000000000011b7 <+78>:    movzx  eax,BYTE PTR [rbp-0x15]
   0x00000000000011bb <+82>:    movsx  eax,al
   0x00000000000011be <+85>:    cmp    eax,0x2f
   0x00000000000011c1 <+88>:    je     0x128b <main+290>
   0x00000000000011c7 <+94>:    cmp    eax,0x2f
   0x00000000000011ca <+97>:    jg     0x12bf <main+342>
   0x00000000000011d0 <+103>:   cmp    eax,0x2d
   0x00000000000011d3 <+106>:   je     0x1223 <main+186>
   0x00000000000011d5 <+108>:   cmp    eax,0x2d
   0x00000000000011d8 <+111>:   jg     0x12bf <main+342>
   0x00000000000011de <+117>:   cmp    eax,0x2a
   0x00000000000011e1 <+120>:   je     0x1257 <main+238>
   0x00000000000011e3 <+122>:   cmp    eax,0x2b
   0x00000000000011e6 <+125>:   jne    0x12bf <main+342>
   0x00000000000011ec <+131>:   movss  xmm0,DWORD PTR [rbp-0x10]
   0x00000000000011f1 <+136>:   cvtss2sd xmm0,xmm0
   0x00000000000011f5 <+140>:   movss  xmm1,DWORD PTR [rbp-0x14]
   0x00000000000011fa <+145>:   pxor   xmm2,xmm2
   0x00000000000011fe <+149>:   cvtss2sd xmm2,xmm1
   0x0000000000001202 <+153>:   movq   rax,xmm2
   0x0000000000001207 <+158>:   movapd xmm1,xmm0
   0x000000000000120b <+162>:   movq   xmm0,rax
   0x0000000000001210 <+167>:   call   0x1317 <add>
   0x0000000000001215 <+172>:   cvtsd2ss xmm0,xmm0
   0x0000000000001219 <+176>:   movss  DWORD PTR [rbp-0xc],xmm0
   0x000000000000121e <+181>:   jmp    0x12d5 <main+364>
   0x0000000000001223 <+186>:   movss  xmm0,DWORD PTR [rbp-0x10]
   0x0000000000001228 <+191>:   cvtss2sd xmm0,xmm0
   0x000000000000122c <+195>:   movss  xmm1,DWORD PTR [rbp-0x14]
   0x0000000000001231 <+200>:   pxor   xmm3,xmm3
   0x0000000000001235 <+204>:   cvtss2sd xmm3,xmm1
   0x0000000000001239 <+208>:   movq   rax,xmm3
   0x000000000000123e <+213>:   movapd xmm1,xmm0
   0x0000000000001242 <+217>:   movq   xmm0,rax
   0x0000000000001247 <+222>:   call   0x133b <subtract>
   0x000000000000124c <+227>:   cvtsd2ss xmm0,xmm0
   0x0000000000001250 <+231>:   movss  DWORD PTR [rbp-0xc],xmm0
   0x0000000000001255 <+236>:   jmp    0x12d5 <main+364>
   0x0000000000001257 <+238>:   movss  xmm0,DWORD PTR [rbp-0x10]
   0x000000000000125c <+243>:   cvtss2sd xmm0,xmm0
   0x0000000000001260 <+247>:   movss  xmm1,DWORD PTR [rbp-0x14]
   0x0000000000001265 <+252>:   pxor   xmm4,xmm4
   0x0000000000001269 <+256>:   cvtss2sd xmm4,xmm1
   0x000000000000126d <+260>:   movq   rax,xmm4
   0x0000000000001272 <+265>:   movapd xmm1,xmm0
   0x0000000000001276 <+269>:   movq   xmm0,rax
   0x000000000000127b <+274>:   call   0x135f <multiply>
   0x0000000000001280 <+279>:   cvtsd2ss xmm0,xmm0
   0x0000000000001284 <+283>:   movss  DWORD PTR [rbp-0xc],xmm0
   0x0000000000001289 <+288>:   jmp    0x12d5 <main+364>
   0x000000000000128b <+290>:   movss  xmm0,DWORD PTR [rbp-0x10]
   0x0000000000001290 <+295>:   cvtss2sd xmm0,xmm0
   0x0000000000001294 <+299>:   movss  xmm1,DWORD PTR [rbp-0x14]
   0x0000000000001299 <+304>:   pxor   xmm5,xmm5
   0x000000000000129d <+308>:   cvtss2sd xmm5,xmm1
   0x00000000000012a1 <+312>:   movq   rax,xmm5
   0x00000000000012a6 <+317>:   movapd xmm1,xmm0
   0x00000000000012aa <+321>:   movq   xmm0,rax
   0x00000000000012af <+326>:   call   0x1383 <divide>
   0x00000000000012b4 <+331>:   cvtsd2ss xmm0,xmm0
   0x00000000000012b8 <+335>:   movss  DWORD PTR [rbp-0xc],xmm0
   0x00000000000012bd <+340>:   jmp    0x12d5 <main+364>
   0x00000000000012bf <+342>:   lea    rax,[rip+0xd60]        # 0x2026
   0x00000000000012c6 <+349>:   mov    rdi,rax
   0x00000000000012c9 <+352>:   call   0x1030 <puts@plt>
   0x00000000000012ce <+357>:   mov    eax,0x1
   0x00000000000012d3 <+362>:   jmp    0x1301 <main+408>
   0x00000000000012d5 <+364>:   pxor   xmm6,xmm6
   0x00000000000012d9 <+368>:   cvtss2sd xmm6,DWORD PTR [rbp-0xc]
   0x00000000000012de <+373>:   movq   rax,xmm6
   0x00000000000012e3 <+378>:   movq   xmm0,rax
   0x00000000000012e8 <+383>:   lea    rax,[rip+0xd48]        # 0x2037
   0x00000000000012ef <+390>:   mov    rdi,rax
   0x00000000000012f2 <+393>:   mov    eax,0x1
   0x00000000000012f7 <+398>:   call   0x1050 <printf@plt>
   0x00000000000012fc <+403>:   mov    eax,0x0
   0x0000000000001301 <+408>:   mov    rdx,QWORD PTR [rbp-0x8]
   0x0000000000001305 <+412>:   sub    rdx,QWORD PTR fs:0x28
   0x000000000000130e <+421>:   je     0x1315 <main+428>
   0x0000000000001310 <+423>:   call   0x1040 <__stack_chk_fail@plt>
   0x0000000000001315 <+428>:   leave
   0x0000000000001316 <+429>:   ret
End of assembler dump.

```


For now, just focus on `printf` and `scanf`


```
   0x000000000000118f <+38>:    call   0x1050 <printf@plt>
   0x00000000000011b2 <+73>:    call   0x1060 <__isoc99_scanf@plt>
```


You can note that they point to addresses which are very close to each other.


```
(gdb) x/3i 0x1050
   0x1050 <printf@plt>: jmp    QWORD PTR [rip+0x2fba]        # 0x4010 <printf@got.plt>
   0x1056 <printf@plt+6>:       push   0x2
   0x105b <printf@plt+11>:      jmp    0x1020

(gdb) x/3i 0x1060
   0x1060 <__isoc99_scanf@plt>: jmp    QWORD PTR [rip+0x2fb2]        # 0x4018 <__isoc99_scanf@got.plt>
   0x1066 <__isoc99_scanf@plt+6>:       push   0x3
   0x106b <__isoc99_scanf@plt+11>:      jmp    0x1020
```

If you look at the first instruction of both, you can see that they both points to `0x4010` and `0x4018`. These locations are nothing but the GOT entries. You can take a peak at these locations to see where the first instruction in PLT stub is making a jump to.

```
(gdb) x/1x 0x4010
0x4010 <printf@got.plt>:        0x00001056

(gdb) x/1x 0x4018
0x4018 <__isoc99_scanf@got.plt>:        0x00001066
```


Well, since this is the first time we are going to call these functions, the GOT tables don't have any entries so they jump to second instruction of PLT stub -- addresses `0x1056` and `0x1066` respectively.

Now second instruction will push some number to stack, these numbers will help in hinting which GOT entry needs to be updated.

After pushing the number to stack, both of the PLT stubs jump to same address -- `0x1020`


```
(gdb) x/3i 0x1020
   0x1020:      push   QWORD PTR [rip+0x2fca]        # 0x3ff0
   0x1026:      jmp    QWORD PTR [rip+0x2fcc]        # 0x3ff8
   0x102c:      nop    DWORD PTR [rax+0x0]
```

This won't make much sense unless you start the execution of `calc` which will force the loading of the `libc`


Let's take a look at things again now


```
0x000055555555518f <+38>:    call   0x555555555050 <printf@plt>
0x00005555555551b2 <+73>:    call   0x555555555060 <__isoc99_scanf@plt>
```


PLT stub and GOT entries

```
(gdb) x/3i 0x555555555050
   0x555555555050 <printf@plt>: jmp    QWORD PTR [rip+0x2fba]        # 0x555555558010 <printf@got.plt>
   0x555555555056 <printf@plt+6>:       push   0x2
   0x55555555505b <printf@plt+11>:      jmp    0x555555555020

(gdb) x/1x 0x555555558010
0x555555558010 <printf@got.plt>:        0x55555056


(gdb) x/3i 0x555555555060
   0x555555555060 <__isoc99_scanf@plt>: jmp    QWORD PTR [rip+0x2fb2]        # 0x555555558018 <__isoc99_scanf@got.plt>
   0x555555555066 <__isoc99_scanf@plt+6>:       push   0x3
   0x55555555506b <__isoc99_scanf@plt+11>:      jmp    0x555555555020

(gdb) x/1x 0x555555558018
0x555555558018 <__isoc99_scanf@got.plt>:        0x55555066
```


Everything is similar to this point... except the addresses


```
(gdb) x/3i 0x555555555020
=> 0x555555555020:      push   QWORD PTR [rip+0x2fca]        # 0x555555557ff0
   0x555555555026:      jmp    QWORD PTR [rip+0x2fcc]        # 0x555555557ff8
   0x55555555502c:      nop    DWORD PTR [rax+0x0]
```

It pushes something to stack and then makes a jump call to `0x555555557ff8` address which is actually `_dl_runtime_resolve_xsavec` function in our dynamic linker. This function will resolve the address for `printf` and `scanf` and then patch the GOT table for it.


Once that it done, you can check the patched entries

```
(gdb) x/1x 0x555555558010
0x555555558010 <printf@got.plt>:        0xf7e16730

(gdb) x/1x 0x555555558018
0x555555558018 <__isoc99_scanf@got.plt>:        0xf7e16430
```

Now this has actual address for `printf` and `scanf` functions....


Now whenever `calc` wants to use the `printf` again it does not need to actually resolve the address everytime. Also we do not need to patch the `.text` section of `calc` which keeps on pointing to the PLT stub.
















---

### Resources
- https://developer.ibm.com/tutorials/l-dynamic-libraries/
- https://opensource.com/article/22/5/dynamic-linking-modular-libraries-linux
