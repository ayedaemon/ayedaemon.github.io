---
title: "Intro to Re: C : part-4"
date: 2023-05-01T02:34:50+05:30
draft: False
showtoc: true
tags: [RE, linux, C programming]
series: [Reverse Engineering]
description: Some things about process and stack memory
# math: true
# ShowBreadCrumbs: false
# searchHidden: true
# hideMeta: true
---


When an operating system (OS) runs a program, the program is first loaded into main memory. Memory is utilized for both program's machine instructions and program's data...this includes parameters, dynamic variables, (un)initialized variables, and so on.

Most computers today use paged memory allocations, which allow the amount of memory assigned to a program to increase/decrease as the needs of the application change. Memory is allocated to the program and reclaimed by the operating system in fixed-size chunks known as `pages`. When a program is loaded into a paged-memory computer, the operating system initially allocates a small number of pages to the program and then allocates additional memory as needed.

On a linux machine you can check the memory layout of a running program using `cat /proc/<proc_id>/map`.

```
> cat /proc/self/maps

55f5db535000-55f5db537000 r--p 00000000 08:02 917947                     /usr/bin/cat
55f5db537000-55f5db53b000 r-xp 00002000 08:02 917947                     /usr/bin/cat
55f5db53b000-55f5db53d000 r--p 00006000 08:02 917947                     /usr/bin/cat
55f5db53d000-55f5db53e000 r--p 00007000 08:02 917947                     /usr/bin/cat
55f5db53e000-55f5db53f000 rw-p 00008000 08:02 917947                     /usr/bin/cat
55f5dd440000-55f5dd461000 rw-p 00000000 00:00 0                          [heap]
7f0db2800000-7f0db2aea000 r--p 00000000 08:02 929341                     /usr/lib/locale/locale-archive
7f0db2ba4000-7f0db2bc9000 rw-p 00000000 00:00 0 
7f0db2bc9000-7f0db2beb000 r--p 00000000 08:02 923932                     /usr/lib/libc.so.6
7f0db2beb000-7f0db2d45000 r-xp 00022000 08:02 923932                     /usr/lib/libc.so.6
7f0db2d45000-7f0db2d9d000 r--p 0017c000 08:02 923932                     /usr/lib/libc.so.6
7f0db2d9d000-7f0db2da1000 r--p 001d4000 08:02 923932                     /usr/lib/libc.so.6
7f0db2da1000-7f0db2da3000 rw-p 001d8000 08:02 923932                     /usr/lib/libc.so.6
7f0db2da3000-7f0db2db2000 rw-p 00000000 00:00 0 
7f0db2dd0000-7f0db2dd1000 r--p 00000000 08:02 923793                     /usr/lib/ld-linux-x86-64.so.2
7f0db2dd1000-7f0db2df7000 r-xp 00001000 08:02 923793                     /usr/lib/ld-linux-x86-64.so.2
7f0db2df7000-7f0db2e01000 r--p 00027000 08:02 923793                     /usr/lib/ld-linux-x86-64.so.2
7f0db2e01000-7f0db2e03000 r--p 00031000 08:02 923793                     /usr/lib/ld-linux-x86-64.so.2
7f0db2e03000-7f0db2e05000 rw-p 00033000 08:02 923793                     /usr/lib/ld-linux-x86-64.so.2
7ffd55089000-7ffd550aa000 rw-p 00000000 00:00 0                          [stack]
7ffd550f6000-7ffd550fa000 r--p 00000000 00:00 0                          [vvar]
7ffd550fa000-7ffd550fc000 r-xp 00000000 00:00 0                          [vdso]
ffffffffff600000-ffffffffff601000 --xp 00000000 00:00 0                  [vsyscall]
```


When you read the contents of `/proc/self/map`, you will encounter multiple lines, with each line representing a separate memory mapping region. Each line contains various fields separated by whitespace, representing different attributes of the memory mapping. The common fields include:

- **Start and End Addresses**: The starting and ending virtual addresses of the memory mapping region.
- **Permissions**: The permissions assigned to the memory mapping, such as read, write, and execute.
- **Offset**: The offset in the file (if the mapping is backed by a file) or zero otherwise.
- **Device and Inode**: The device and inode number of the file backing the mapping.
- **File descriptor**: If the mapping is associated with a file opened by the process, the file descriptor number is mentioned in this field.
- **Flags**: Additional flags indicating special characteristics of the mapping.
- **Inode and Path**: The inode number and path of the file backing the mapping (if available).




## Memory layout of a process

Memory space allocated to a running program/process is called process memory (AKA virtual memory). It allows multiple programs to run concurrently and provides each program with a dedicated and isolated memory space. The purpose of process memory is to facilitate the execution of programs by providing a private address space for each process, shielding them from interfering with one another.


A typical memory layout consists of many segments.

1. Text segment (code segment)
2. Initialized data segment (data segment)
3. Uninitialized data segment (bss segment)
4. Heap
5. Stack


### Text segment (Code Segment)
The code segment contains the executable instructions of the program. It is typically read-only and stores the program's machine code instructions, constants, and literals.


### Initialized data segment (Data Segment)
This segment contains initialized static variables like global variables and local static variables which have a defined value and can be modified.


### Uninitialized data segment (BSS Segment)
This segment contains uninitialized static data, both variables and constants. On most systems, kernel automatically zeros this segment.

### Heap
This segment contains dynamically allocated memory. It is usually managed by malloc, calloc, realloc, free (and their sibling functions too). The heap segment is shared by all threads, shared libraries, and dynamically loaded modules in a process. Heap memory segment grows towards higher memory addresses.

### Stack
This region of memory is used for managing function calls and local variables. It is an essential part of the execution environment and plays a crucial role in program flow control. 

This is typically located in higher parts of the memory and grows towards lower parts (towards heap memory). A **stack pointer** register keeps track of the top of the stack, this gets adjusted each time a new value is pushed to or poped from the stack.

![](https://upload.wikimedia.org/wikipedia/commons/thumb/5/50/Program_memory_layout.pdf/page1-225px-Program_memory_layout.pdf.jpg#center)

This is what the whole memory layout looks like altogether, but in this article we are focusing mainly on stack segment.

## More about stack (practical)

Stack memory is organized into stack frames, each representing the activation record of a function call. A stack frame contains information such as function parameters, local variables, return addresses, and other metadata necessary for function execution.

Let's start with a simple example of function calls to understand how stack works.

```c
#include<stdio.h>

// Prints are garbage value from stack and returns the same.
int func2() {
   int var2;
   printf("var2 (%p) = %d\n", &var2, var2);
   return var2;
}

// Prints var1 value as 55 and returns it.
int func1() {
   int var1 = 55;
   printf("var1 (%p) = %d\n", &var1, var1);
   return var1;
}

// Main function
int main() {
    // Calls both functions and stores their return values
   int v1 = func1();
   int v2 = func2();

    // Compares their return values and print message
   if(v1 == v2)
      printf("Both values are equal\n");
   else
      printf("Both values are not equal\n");
   
   return 10;
}
```

To get a better understanding for this, I'll put this down in steps of what this program will do upon execution.

1. Call `func1()`, get a return value and store that in `v1` variable.
2. Call `func2()`, get a return value and store that in `v2` variable
3. Compare both values and print appropriate message.

That's it. Quite straightforward, isn't it?. Let's see the output of this program before jumping to conclusions.

```
var1 (0x7ffeaf266d1c) = 55
var2 (0x7ffeaf266d1c) = 55
Both values are equal
```

![](https://media.giphy.com/media/edGzBC6GDOhutW32ps/giphy.gif#center)

Both `var1` and `var2` have same values and infact have same memory addresses... even though they belong to different functions and have separate stack frame and everything.




To understand this we'll have to go deeper with a debugger (I'm using GDB).

```c
>>> disas main
Dump of assembler code for function main:
   // Prologue
   0x00000000000011a6 <+0>:	    push   rbp
   0x00000000000011a7 <+1>:	    mov    rbp,rsp
   
   // Creating space in stack for variables. 0x10 is 16 bytes (4 bytes for each int variable)
   // 4(v1) + 4(v2) + 8 (padding)
   0x00000000000011aa <+4>:	    sub    rsp,0x10
   
   // Function call and store value in [rbp-0x4]
   0x00000000000011ae <+8>:	    mov    eax,0x0
   0x00000000000011b3 <+13>:	call   0x1174 <func1>
   0x00000000000011b8 <+18>:	mov    DWORD PTR [rbp-0x4],eax
   
   // Another function call and store value in [rbp-0x8]
   0x00000000000011bb <+21>:	mov    eax,0x0
   0x00000000000011c0 <+26>:	call   0x1149 <func2>
   0x00000000000011c5 <+31>:	mov    DWORD PTR [rbp-0x8],eax
   
   // Compare values stored in [rbp-0x4] & [rbp-0x8]
   0x00000000000011c8 <+34>:	mov    eax,DWORD PTR [rbp-0x4]
   0x00000000000011cb <+37>:	cmp    eax,DWORD PTR [rbp-0x8]
   
   // if not equal then jump to <main+59>
   0x00000000000011ce <+40>:	jne    0x11e1 <main+59>
   // else print this message
   0x00000000000011d0 <+42>:	lea    rax,[rip+0xe4d]        # 0x2024
   0x00000000000011d7 <+49>:	mov    rdi,rax
   0x00000000000011da <+52>:	call   0x1030 <puts@plt>
   
   // And finally jump to <main+74> (epilogue)
   0x00000000000011df <+57>:	jmp    0x11f0 <main+74>
   // if comparision failed: land here and print this message
   0x00000000000011e1 <+59>:	lea    rax,[rip+0xe52]        # 0x203a
   0x00000000000011e8 <+66>:	mov    rdi,rax
   0x00000000000011eb <+69>:	call   0x1030 <puts@plt>
   
   // Finally epilogue - set return value and leave
   0x00000000000011f0 <+74>:	mov    eax,0xa
   0x00000000000011f5 <+79>:	leave
   0x00000000000011f6 <+80>:	ret
End of assembler dump.
```
In the disassembly code, the `main` function calls both the functions and stores their respective return values in `[rbp-0x4]` and `[rbp-0x8]` memory locations. Since these variables are specific to `main` function, they will be created in the stack memory (inside the stack frame for `main` function).

```c
>>> disas func1
Dump of assembler code for function func1:
   // Prologue
   0x0000000000001174 <+0>:	    push   rbp
   0x0000000000001175 <+1>:	    mov    rbp,rsp
   // Create memory for the variable -- 4(var1) + 12(padding) = 16 (0x10)
   0x0000000000001178 <+4>:	    sub    rsp,0x10
   // Store 0x37(55) in [rbp-0x4]
   0x000000000000117c <+8>:	    mov    DWORD PTR [rbp-0x4],0x37
   // Print this value with a specific message
   0x0000000000001183 <+15>:	mov    edx,DWORD PTR [rbp-0x4]
   0x0000000000001186 <+18>:	lea    rax,[rbp-0x4]
   0x000000000000118a <+22>:	mov    rsi,rax
   0x000000000000118d <+25>:	lea    rax,[rip+0xe80]        # 0x2014
   0x0000000000001194 <+32>:	mov    rdi,rax
   0x0000000000001197 <+35>:	mov    eax,0x0
   0x000000000000119c <+40>:	call   0x1040 <printf@plt>
   // Epilogue: set this value as return value and leave
   0x00000000000011a1 <+45>:	mov    eax,DWORD PTR [rbp-0x4]
   0x00000000000011a4 <+48>:	leave
   0x00000000000011a5 <+49>:	ret
End of assembler dump.
```

The above function, when called, will create another stack frame just after the main function's stack frame... and will create it's local variables in that region.

This function will print the value of the local variable and then return back to the main function. This action will remove the stack frame created by resetting the stack pointer and base pointer register values... BUT the actual values stored in memory location is still not overwritten by anything. So technically, the values are still present there and can be accessed if the memory location can be pointed to.

```c
>>> disas func2
Dump of assembler code for function func2:
   // Prologue
   0x0000000000001149 <+0>:	    push   rbp
   0x000000000000114a <+1>:	    mov    rbp,rsp
   // Create memory for the variable -- 4(var2) + 12(padding) = 16 (0x10)
   0x000000000000114d <+4>:	    sub    rsp,0x10
   // Print the value with a specific message
   0x0000000000001151 <+8>:	    mov    edx,DWORD PTR [rbp-0x4]
   0x0000000000001154 <+11>:	lea    rax,[rbp-0x4]
   0x0000000000001158 <+15>:	mov    rsi,rax
   0x000000000000115b <+18>:	lea    rax,[rip+0xea2]        # 0x2004
   0x0000000000001162 <+25>:	mov    rdi,rax
   0x0000000000001165 <+28>:	mov    eax,0x0
   0x000000000000116a <+33>:	call   0x1040 <printf@plt>
   // Epilogue: set this value as return value and leave
   0x000000000000116f <+38>:	mov    eax,DWORD PTR [rbp-0x4]
   0x0000000000001172 <+41>:	leave
   0x0000000000001173 <+42>:	ret
End of assembler dump.
```

After the `func1` has returned, it's time for `func2` to create it's own stack frame and it's own local variables.

Coincidently, the memory location used by `func2` is exactly the same location that was used by `func1` earlier. And on top of that, both functions have `int` type variables which means that the memory location used by `var1` in `func1` will be used by `var2` in `func2`. 

Stack frame for both functions will *kind of* overlap each other. That should explain why we we're getting the same results and same memory locations in the program output earlier. 



This theory should be enough to understand what's going on.... But practical is more fun. Accept it!!

![](https://media.giphy.com/media/hu0tQXRqT06YOFvd8D/giphy.gif#center)

I'm going to place some breakpoints in the code and check the status of the stack on each hit. For me, such interesting points are where new variables or function's stack frame will be created. This will help me to analyze the change in stack as we go forward.


(***NOTE**: Sometimes I over-use breakpoints. Don't judge me :|* )

```c{linenos=false}
>>> info break
Num     Type           Disp Enb Address            What
// 0x5555555551a6 <main+0>:	    push   rbp
1       breakpoint     keep y   0x00005555555551a6 in main at func_calls.c:16

// 0x5555555551aa <main+4>:	    sub    rsp,0x10
2       breakpoint     keep y   0x00005555555551aa in main at func_calls.c:16

// 0x5555555551b3 <main+13>:	call   0x555555555174 <func1>
3       breakpoint     keep y   0x00005555555551b3 in main at func_calls.c:17

// 0x5555555551c0 <main+26>:	call   0x555555555149 <func2>
4       breakpoint     keep y   0x00005555555551c0 in main at func_calls.c:18

// 0x5555555551f5 <main+79>:	leave
5       breakpoint     keep y   0x00005555555551f5 in main at func_calls.c:26

// 0x555555555174 <func1>:	    push   rbp
6       breakpoint     keep y   0x0000555555555174 in func1 at func_calls.c:9

// 0x555555555178 <func1+4>:	sub    rsp,0x10
7       breakpoint     keep y   0x0000555555555178 in func1 at func_calls.c:9

// 0x5555555551a5 <func1+49>:	ret
8       breakpoint     keep y   0x00005555555551a5 in func1 at func_calls.c:13

// 0x555555555149 <func2>:	    push   rbp
9       breakpoint     keep y   0x0000555555555149 in func2 at func_calls.c:3

// 0x55555555514d <func2+4>:	sub    rsp,0x10
10      breakpoint     keep y   0x000055555555514d in func2 at func_calls.c:3

// 0x555555555173 <func2+42>:	ret
11      breakpoint     keep y   0x0000555555555173 in func2 at func_calls.c:7
```

I've set up 11 break points here which will help me check the change in stack during the execution of the program.

After running the program in debugger, it'll stop at the first break point which is just before the point where `main` function's stack frame will begin.

```bash
>>> x/10xg $rsp
0x7fffffffe218:	0x00007ffff7de0790	0x00007fffffffe310
0x7fffffffe228:	0x00005555555551a6	0x0000000155554040
0x7fffffffe238:	0x00007fffffffe328	0x00007fffffffe328
0x7fffffffe248:	0x86b5da47f7ba01f3	0x0000000000000000
0x7fffffffe258:	0x00007fffffffe338	0x0000555555557dd8
```

Theoretically, we know if we step over another instruction, value from `rbp` will be stored in the stack. So let's check the value of `rbp` right now and then monitor the stack (after stepping over) to see if it is the same value we are expecting it to be.

```bash
// Check the base pointer before stepping over instruction
>>> p $rbp
$1 = (void *) 0x1

// Check the next instruction to be executed
>>> x $rip
=> 0x5555555551a6 <main>:	push   rbp

// Step over an instruction
>>> ni

// Monitor stack
>>> x/10xg $rsp
0x7fffffffe210:	0x0000000000000001	0x00007ffff7de0790
0x7fffffffe220:	0x00007fffffffe310	0x00005555555551a6
0x7fffffffe230:	0x0000000155554040	0x00007fffffffe328
0x7fffffffe240:	0x00007fffffffe328	0x8f2aa2d53bd1951c
0x7fffffffe250:	0x0000000000000000	0x00007fffffffe338
```

Now our stack has a new item in it, that is `rbp` value. If you notice, previously the top of the stack was at `0x7fffffffe218`, but after adding one item the top of stack is `0x7fffffffe210`. It decreased, which indicates that stack grows downwards; towards lower memory addresses.


The stack does not change up until the breakpoint 2 on `0x5555555551aa <main+4>:	    sub    rsp,0x10`... But on stepping another instruction, we can see a new space of **16 bytes** in the stack.

```
>>> x/10xg $rsp
0x7fffffffe200:	0x0000000000000000	0x00007ffff7ffdab0
0x7fffffffe210:	0x0000000000000001	0x00007ffff7de0790
0x7fffffffe220:	0x00007fffffffe310	0x00005555555551a6
0x7fffffffe230:	0x0000000155554040	0x00007fffffffe328
0x7fffffffe240:	0x00007fffffffe328	0x8f2aa2d53bd1951c
```

Before this, the stack pointer was on `0x7fffffffe210`, now the stack pointer is on `0x7fffffffe200`... so `0x7fffffffe210 - 0x7fffffffe200 = 0x10 (16)`. Simple maths!!

We just moved the stack pointer down to create space required, no cleaning was done... hence the stack is already filled with some garbage value that resided in the memory long before the stack occupied this new memory region.

Now let's call the function `func1` and see what that function call will add to the stack.

```
>>> x/10xg $rsp
0x7fffffffe1f8:	0x00005555555551b8	0x0000000000000000
0x7fffffffe208:	0x00007ffff7ffdab0	0x0000000000000001
0x7fffffffe218:	0x00007ffff7de0790	0x00007fffffffe310
0x7fffffffe228:	0x00005555555551a6	0x0000000155554040
0x7fffffffe238:	0x00007fffffffe328	0x00007fffffffe328
```
Just after making the function call and before executing the first instruction of the `func1`, we have a new value in the stack. This new value is the return address for the main function, which will be used to continue execution after the func1 call is finished.

```
>>> x/i 0x00005555555551b8
   0x5555555551b8 <main+18>:	mov    DWORD PTR [rbp-0x4],eax
```

(Note: if this value is somehow overwritten, we can make our function to return to another function or instruction. This is something which comes under a technique called Return Oriented Programming aka ROP.)

After the first instruction of the `func1` function, that is pushing the `rbp` on the stack, we can see an updated stack.


```
>>> x/10xg $rsp
0x7fffffffe1f0:	0x00007fffffffe210	0x00005555555551b8
0x7fffffffe200:	0x0000000000000000	0x00007ffff7ffdab0
0x7fffffffe210:	0x0000000000000001	0x00007ffff7de0790
0x7fffffffe220:	0x00007fffffffe310	0x00005555555551a6
0x7fffffffe230:	0x0000000155554040	0x00007fffffffe328
```

This points to the place where the `main` function's `base pointer` is. Think this as a chain like connection which links to the next stop.

Now after creating some space in this new stack frame for `var1`.... Stack looks like this. (*Notice that the stack keeps on growing down, towards low memory regions.*)

```
>>> x/10xg $rsp
0x7fffffffe1e0:	0x0000000000000000	0x0000000000000000
0x7fffffffe1f0:	0x00007fffffffe210	0x00005555555551b8
0x7fffffffe200:	0x0000000000000000	0x00007ffff7ffdab0
0x7fffffffe210:	0x0000000000000001	0x00007ffff7de0790
0x7fffffffe220:	0x00007fffffffe310	0x00005555555551a6
```


The `leave` instruction does a lot of the work. It copies the `rbp` to `rsp` and then restores the old `rbp` from the stack again.

This will move the base pointer back to `0x00007fffffffe210` and the stack pointer to `0x00007fffffffe1f8`.

This instruction basically releases a stack frame set up by a function. That means now our new stack will look something like this.

```
>>> x/10xg $rsp
0x7fffffffe1f8:	0x00005555555551b8	0x0000000000000000
0x7fffffffe208:	0x00007ffff7ffdab0	0x0000000000000001
0x7fffffffe218:	0x00007ffff7de0790	0x00007fffffffe310
0x7fffffffe228:	0x00005555555551a6	0x0000000155554040
0x7fffffffe238:	0x00007fffffffe328	0x00007fffffffe328
```


The next `ret` statement will take off the top of the value from stack and move the instruction control to that memory location... that is, set the `rip` value to that memory location. This means that the next instruction to be executed will be <main+18> which stores the `eax` (return value from the previous function call) into `[rbp-0x4]` location on stack.

```
>>> x/i 0x00005555555551b8
=> 0x5555555551b8 <main+18>:	mov    DWORD PTR [rbp-0x4],eax
```

So when `func1` returns back, our stack looks something like this.


```
>>> x/10xg $rsp
0x7fffffffe200:	0x0000000000000000	0x00007ffff7ffdab0
0x7fffffffe210:	0x0000000000000001	0x00007ffff7de0790
0x7fffffffe220:	0x00007fffffffe310	0x00005555555551a6
0x7fffffffe230:	0x0000000155554040	0x00007fffffffe328
0x7fffffffe240:	0x00007fffffffe328	0x42033bae0bc1441d
```

Now for the next function `func2`, our stack is like this, just before the `sub    rsp,0x10` instruction. This instruction will create enough memory for the variables to be.

```
>>> x/10xg $rsp
0x7fffffffe1f0:	0x00007fffffffe210	0x00005555555551c5
0x7fffffffe200:	0x0000000000000000	0x00000037f7ffdab0
0x7fffffffe210:	0x0000000000000001	0x00007ffff7de0790
0x7fffffffe220:	0x00007fffffffe310	0x00005555555551a6
0x7fffffffe230:	0x0000000155554040	0x00007fffffffe328
```


Once the space in stack memory is created, the stack looks like this

```
>>> x/10xg $rsp
0x7fffffffe1e0:	0x0000000000000000	0x0000003700000000
0x7fffffffe1f0:	0x00007fffffffe210	0x00005555555551c5
0x7fffffffe200:	0x0000000000000000	0x00000037f7ffdab0
0x7fffffffe210:	0x0000000000000001	0x00007ffff7de0790
0x7fffffffe220:	0x00007fffffffe310	0x00005555555551a6
```


If you notice, these are the same memory locations which were used by `func1` to store `0x37` (`mov    DWORD PTR [rbp-0x4],0x37`).... and since the stack is not cleaned properly, the value from that function is still in the place it was left to be picked by our variable `var2`. 

This can be checked with `x/w ($rbp-0x4)`

```
>>> x/w ($rbp-0x4)
0x7fffffffe1ec:	0x00000037
```


So the value is just as it was left by `func1`... :|


Now after returning back to `main` function, the stack looks like this.


```
>>> x/10xg $rsp
0x7fffffffe200:	0x0000000000000000	0x0000003700000037
0x7fffffffe210:	0x0000000000000001	0x00007ffff7de0790
0x7fffffffe220:	0x00007fffffffe310	0x00005555555551a6
0x7fffffffe230:	0x0000000155554040	0x00007fffffffe328
0x7fffffffe240:	0x00007fffffffe328	0x42033bae0bc1441d
```

And both `v1` and `v2`, are same values and were picked from the same memory locations when in their respective function frames.

```
>>> x/w $rbp-0x4
0x7fffffffe20c:	0x00000037
>>> x/w $rbp-0x8
0x7fffffffe208:	0x00000037
```
![](https://media.giphy.com/media/3ohzdNuQy40h2dUCGI/giphy.gif#center)

This explains the *somewhat* unusual behaviour of the program. There is nothing special about this, its just the way the stack memory works and a well crafted example presented to you.

In next article, I'll try to cover stack overflows... which is a cool technique to insert the data in a variable which will overflow and overwrite the data for another variable. Try to think around this, if you may.
