---
title: "Intro to Re: C : part-2"
date: 2023-03-19T22:07:39+05:30
draft: false
showtoc: false
tags: [RE, linux, C programming]
series: [Reverse Engineering]
description: How to reverse engineer a basic C program
# math: true
# ShowBreadCrumbs: false
# searchHidden: true
# hideMeta: true
---


Reverese engineering is a powerful tool for any software developer. However, as with any tool, it is only as good as the person using it. Understanding reverse engineering and how to use it is essential for both novices and seasoned developers. 

According to wikipedia,

> Reverse engineering, also called back engineering, is the process by which a man-made object is deconstructed to reveal its designs, architecture, or to extract knowledge from the object; similar to scientific research, the only difference being that scientific research is about a natural phenomenon.


Because there will be a lot of new things, I'm attempting to keep this article more practical and experimental...I encourage you to follow along and experiment with your own examples as well. 


![](https://media.giphy.com/media/109fP7pua6Osgw/giphy.gif#center)

The following is a very simple C program that starts the `main()` function and exits by returning a value `0`.

```c
int main() {
    return 0;
}
```

This source code will be compiled to create an executable binary file. When we disassemble the compiled binary, we get something like this... 

(Note: There are [many disassemblers](https://en.wikipedia.org/wiki/Disassembler#Examples_of_disassemblers) [^disas_examples] you can use. Nearly all the disassembled instructions in this blog were produced with [godbolt](https://godbolt.org/))

[^disas_examples]: https://en.wikipedia.org/wiki/Disassembler#Examples_of_disassemblers

```asm
main:
        push    rbp
        mov     rbp, rsp
        mov     eax, 0
        pop     rbp
        ret
```

In this example, function `prologue` and `epilogue` are the first two and last two instructions, respectively. These are used to build a frame for a function. This frame contains all of the local variables used or defined in this function.

Whenever a function is created, a new frame is created, all the local variables are stored in respective memory blocks inside this frame and finally the frame is discarded when the function returns.

We'll go into more detail about this later, but for now, just remember that there are `prologue` and `epilogue` instructions that mark the beginning and end of a function. 

With this out of the picture, the instruction at line 4 appears to be in charge of returning `0` to the caller function (whoever called `main()`). 

The idea is to use the `eax` register as a storage area for return values. When a function returns something, it simply stores the value in the `eax`/`rax` register so that the caller function can read it later if necessary. 

Now we know some background theory, let's try to change the return value and see how our assembly instructions reflect the change.


```c
int main(){
    return 6;
}
```
The assembly instructions for the above code are nearly identical to the previous one, with the exception of the return value at `line 4`. Now eax register stores the value `6` instead of `0`. 

```asm
main:
        push    rbp
        mov     rbp, rsp
        mov     eax, 6
        pop     rbp
        ret

```

If you are familiar with basic C programming, you know that we can avoid returning value by changing the return type of the function `main()` from `int` to `void`.


```c
void main() {}
```

No return statements here! So we can safely assume that the disassembly for this function should consist only of a `prologue` and an `epilogue`, with **no** `mov eax 0` kind of instructions. 

Why?? No `return` statements in the source code, so no need to store anything in `eax` register. Save some CPU cycles. Makes sense, right? Let's check!!


```asm
main:
        push    rbp
        mov     rbp, rsp
        nop
        pop     rbp
        ret
```

The `prologue` and `epilogue` can be seen in the above disassembly, along with a new instruction `nop` that does nothing. A `nop` statement in C is a null statement that can be a semicolon (`;`), an empty block (`{}`), or any other equivalent statement. 

If you're a Pythonista, you're already familiar with the `pass` statement, which has no effect when executed. This serves as a `nop` in python language. 

In the disassembly, we now know what the `prologue`, `epilogue`, and `return` statement look like. Let’s see what variables look like when disassembled.

Everything in the following code is identical to the previous example; the only statement added here is an `int` variable definition. 

```c
void main(){
    int a = 1;
}
```

We have already established that all the local variables for a function are created inside the function frame (which starts with a `prologue` and ends with an `epilogue`). With that in mind, let's take a look at the disassembly for this source code.

```asm
main:
        push    rbp
        mov     rbp, rsp
        mov     DWORD PTR [rbp-4], 1
        nop
        pop     rbp
        ret
```

A new instruction detected - `mov     DWORD PTR [rbp-4], 1`. This is storing value `1` at some location pointed by `rbp-4`. To gain a better understanding of how that works, we'll have to take a detour so that we can debug the binary and see the registers and memory in action.

I'll start `gdb` with the compiled version of the above source code, and then disassemble the main function.

```
>>> disas main
Dump of assembler code for function main:
   0x0000555555555119 <+0>:     push   rbp
   0x000055555555511a <+1>:     mov    rbp,rsp
=> 0x000055555555511d <+4>:     mov    DWORD PTR [rbp-0x4],0x1
   0x0000555555555124 <+11>:    nop
   0x0000555555555125 <+12>:    pop    rbp
   0x0000555555555126 <+13>:    ret
End of assembler dump.
```

In this scenario, I'm at the `mov DWORD PTR [rbp-0x4],0x1` instruction. That means, `rip` register is pointing to this instruction, which means this will be the next instruction executed by the CPU. We can also confirm this by checking the value stored in `rip` register.

```
>>> p/x $rip
$1 = 0x55555555511d
```

See, told ya!

![](https://media.giphy.com/media/v1.Y2lkPTc5MGI3NjExNGZjOTIyNThjNDdmOTFmZDcyMjYxZDE3ZjIxYmU3MmZiNmMxYTBiNCZjdD1n/DT3gYifPXBt5voafS1/giphy.gif#center)


Now, let's print the value from `rbp` register to figure out where the `0x1` will be stored eventually.

```
>>> p/x $rbp
$2 = 0x7fffffffdbd0

>>> p/x $rbp-4
$3 = 0x7fffffffdbcc

>>> p/x (int *)($rbp-4)
$4 = 0x7fffffffdbcc

>>> p/x *(int *)($rbp-4)
$5 = 0x7fff
```

The last value is the current value at the location pointed by `rbp-4` memory location. Consider the below mindmap diagram to get a clear picture.

```goat{linenos=false}





                                                                                  MEMORY


                                                                            ┌────────────────────┐
                                                                            │                    │
                                                                            │                    │
                                                                            │                    │
                                                                            │                    │
                                                                            │                    │
                                                                            │                    │
                                                                            │                    │
                                                                            │                    │
                                                                            ├────────────────────┤
                                 ┌────────────────────────► 0x7fffffffdbd0  │                    │   │  (rbp)
                                 │                                          ├────────────────────┤   │
                                 │                                          │                    │   │
                                 │                                          ├────────────────────┤   │
                                 │                                          │                    │   │
      RBP register               │                                          ├────────────────────┤   │
       ┌──────────────────┐      │                                          │                    │   │
       │                  │      │                                          ├────────────────────┤   │
       │  0x7fffffffdbd0  ├──────┘                                          │                    │   ▼
       │                  │                                 0x7fffffffdbcc  │    0x7fff          │     (rbp - 4)
       └──────────────────┘                                                 ├────────────────────┤
                                                                            │                    │
                                                                            │                    │
                                                                            │                    │
                                                                            │                    │
                                                                            │                    │
                                                                            │                    │
                                                                            │                    │
                                                                            │                    │
                                                                            │                    │
                                                                            │                    │
                                                                            │                    │
                                                                            │                    │
                                                                            └────────────────────┘
```

The `rbp` register holds a memory location, you subtract 4 from the rbp to make enough space for a `int` type data and then store the value at that location.

After executing the instruction, we'll get `0x1` in this location.

```gdb
>>> si

>>> disas main
Dump of assembler code for function main:
   0x0000555555555119 <+0>:     push   rbp
   0x000055555555511a <+1>:     mov    rbp,rsp
   0x000055555555511d <+4>:     mov    DWORD PTR [rbp-0x4],0x1
=> 0x0000555555555124 <+11>:    nop
   0x0000555555555125 <+12>:    pop    rbp
   0x0000555555555126 <+13>:    ret
End of assembler dump.


>>> p/x *(int *)($rbp-4)
$7 = 0x1
```


```goat{linenos=false}





                                                                                  MEMORY


                                                                            ┌────────────────────┐
                                                                            │                    │
                                                                            │                    │
                                                                            │                    │
                                                                            │                    │
                                                                            │                    │
                                                                            │                    │
                                                                            │                    │
                                                                            │                    │
                                                                            ├────────────────────┤
                                 ┌────────────────────────► 0x7fffffffdbd0  │                    │   │  (rbp)
                                 │                                          ├────────────────────┤   │
                                 │                                          │                    │   │
                                 │                                          ├────────────────────┤   │
                                 │                                          │                    │   │
      RBP register               │                                          ├────────────────────┤   │
       ┌──────────────────┐      │                                          │                    │   │
       │                  │      │                                          ├────────────────────┤   │
       │  0x7fffffffdbd0  ├──────┘                                          │                    │   ▼
       │                  │                                 0x7fffffffdbcc  │    0x1             │     (rbp - 4)
       └──────────────────┘                                                 ├────────────────────┤
                                                                            │                    │
                                                                            │                    │
                                                                            │                    │
                                                                            │                    │
                                                                            │                    │
                                                                            │                    │
                                                                            │                    │
                                                                            │                    │
                                                                            │                    │
                                                                            │                    │
                                                                            │                    │
                                                                            │                    │
                                                                            └────────────────────┘
```

This should give you a little idea of how the references work and where does the `0x1` is actually stored.

With that out of our plate, let's get back to the original instruction in question - `mov     DWORD PTR [rbp-4], 1`. This instruction surely stores the value `0x1` in the location pointed by `rbp - 4` memory location.


Then there is the same old `nop` and `epilogue`. Nothing new here. Let's add more variables to the function.


```c
void main() {
    int a = 1, b=2, c=3, d=4;
    int e = 5;
}
```

Below is the disassembly for this 

```asm
main:
        push    rbp
        mov     rbp, rsp
        mov     DWORD PTR [rbp-4], 1
        mov     DWORD PTR [rbp-8], 2
        mov     DWORD PTR [rbp-12], 3
        mov     DWORD PTR [rbp-16], 4
        mov     DWORD PTR [rbp-20], 5
        nop
        pop     rbp
        ret
```

Each variable is created in sequence and have a space of `4` bytes to save an `int` type data in it. `Int` data types have a size of `4` bytes, so this makes sense.

Another commonly used data type is `char`, which requires a smaller size than `int`, `1 byte` in total. 

 
```c
void main(){
    char a = 1, b=2, c=3, d=4;
    char e = 5;
}
```

Yes, the above program is completely correct from the standpoint of a compiler. When storing data in char types, we don't always need single quotes.

Anyway, the disassembly for this is as follows:- 

```asm
main:
        push    rbp
        mov     rbp, rsp
        mov     BYTE PTR [rbp-1], 1
        mov     BYTE PTR [rbp-2], 2
        mov     BYTE PTR [rbp-3], 3
        mov     BYTE PTR [rbp-4], 4
        mov     BYTE PTR [rbp-5], 5
        nop
        pop     rbp
        ret
```
There is now a noticeable difference between this disassembly and the previous one. The `DWORD` has been changed to `BYTE` in this case, which means that less memory is required to store this data, as evidenced by the memory locations used for storage - `rbp-1,` `rbp-2`, and so on - each of which has only `1 byte` of storage. 

Fun thing, with the help of some computer maths, we can find out that the highest number that can be stored in 1 byte storage location is 127. So if we store any number bigger than this, it'll start shifting to the negative range and then loop back.


```c
void main(){
    char a = 127;
    char b = 128;
    char c = 129;
    char d = 130;
}
```

disassembly for this code:- 

```asm
main:
        push    rbp
        mov     rbp, rsp
        mov     BYTE PTR [rbp-1], 127
        mov     BYTE PTR [rbp-2], -128
        mov     BYTE PTR [rbp-3], -127
        mov     BYTE PTR [rbp-4], -126
        nop
        pop     rbp
        ret
```


Good, now we also know how `int` and `char` looks like in their disassembly output. Let's mix things up a bit to level up and learn something new.


```c
void main(){
    char a = 127;
    char b = 128;
    char c = 129;
    char d = 130;
    int e = 131;
}
```

4 `char` and then 1 `int`. 

- `rbp-1` for `a`.
- `rbp-2` for `b`.
- `rbp-3` for `c`.
- `rbp-4` for `d`.
- `rbp-8` for `e`. Because int takes 4 bytes (This can depend on the OS, architecture and compiler you are using).

Let's check the disassembly to see if we got this right or not.


```asm
main:
        push    rbp
        mov     rbp, rsp
        mov     BYTE PTR [rbp-1], 127
        mov     BYTE PTR [rbp-2], -128
        mov     BYTE PTR [rbp-3], -127
        mov     BYTE PTR [rbp-4], -126
        mov     DWORD PTR [rbp-8], 131
        nop
        pop     rbp
        ret
```


Woohooo!! 

![](https://media.giphy.com/media/xT1Ra6fhiJdvL7A8hi/giphy.gif#center)

Though there is one thing I want you to notice... The `int` value did not follow the same pattern as `char` values because `char` is smaller in size and cannot store a very large value, whereas `int` is comparatively larger and can store the assigned value without any storage issues. As a result, the `int` value remained unchanged from the source code, while the `char` values became negative.

Now, let's change the order of the variables and see how it gets interesting.


```c
void main(){
    char a = 127;
    char b = 128;
    int e = 131;
    char c = 129;
    char d = 130;
}
```

disassembly for this code:-

```asm
main:
        push    rbp
        mov     rbp, rsp
        mov     BYTE PTR [rbp-1], 127
        mov     BYTE PTR [rbp-2], -128
        mov     DWORD PTR [rbp-8], 131
        mov     BYTE PTR [rbp-9], -127
        mov     BYTE PTR [rbp-10], -126
        nop
        pop     rbp
        ret
```


Well, you can see the size needed for each variable has changed. It's time to be aware of a concept called [**Data structure alignment**](https://en.wikipedia.org/wiki/Data_structure_alignment).

According to wikipedia,

> The CPU in modern computer hardware performs reads and writes to memory most efficiently when the data is naturally aligned, which generally means that the data's memory address is a multiple of the data size. For instance, in a 32-bit architecture, the data may be aligned if the data is stored in four consecutive bytes and the first byte lies on a 4-byte boundary. 

Read [this article](https://developer.ibm.com/articles/pa-dalign/) [^ibm_pa_dalign] to gain better understanding of data alignment and why is it even a concerning thing for us.

[^ibm_pa_dalign]: https://developer.ibm.com/articles/pa-dalign/


For now, just know that usually 64-bit CPUs have a native 4-byte load. That means, it can pick up 4 bytes in a single turn and use them for something.

Let's visualize the memory layout for the variables in above code.

```goat{linenos=false}
           4 bytes
        ◄──────────────►

      1      2     3     4
    ┌─────┬─────┬─────┬─────┐
    │     │     │     │     │
    │ 127 │ -128│     │     │
    │     │     │     │     │
    ├─────┴─────┴─────┴─────┤
    │                       │
    │          131          │
    │                       │
    ├─────┬─────┬─────┬─────┤
    │     │     │     │     │
    │ -127│ -126│     │     │
    │     │     │     │     │
    ├─────┼─────┼─────┼─────┤
    │     │     │     │     │
    │     │     │     │     │


                            │
            loads like this │
      ◄─────────────────────┘
```


The above diagram of memory is aligned in a 4 byte load. The first 2 variables are a `BYTE` type so the values are places in the first 2 blocks of the first row. Next variable is an `INT` type, which is equivalent to 4 bytes. But there are no 4 bytes available in the first row, so in order to keep the atomicity, this variable was stored in the next row. *([WHY atomicity??](https://stackoverflow.com/questions/36624881/why-is-integer-assignment-on-a-naturally-aligned-variable-atomic-on-x86))*

So does it mean that we have that 2 bytes space just lying there?? - YES! you can use it if you want. Take a look at the below source code.

```c
void main(){
    char a = 127;
    char b = 128;
    char x = 111;
    char y = 112;
    int e = 131;
    char c = 129;
    char d = 130;
}
```


```asm
main:
        push    rbp
        mov     rbp, rsp
        mov     BYTE PTR [rbp-1], 127
        mov     BYTE PTR [rbp-2], -128
        mov     BYTE PTR [rbp-3], 111
        mov     BYTE PTR [rbp-4], 112
        mov     DWORD PTR [rbp-8], 131
        mov     BYTE PTR [rbp-9], -127
        mov     BYTE PTR [rbp-10], -126
        nop
        pop     rbp
        ret
```


No change in the layout... variable `x` and `y` will fill up the 2 Bytes space that were just lying there to be used. 

But if we add 1 more variable `z`, then the whole thing moves.

```c
void main(){
    char a = 127;
    char b = 128;
    char x = 111;
    char y = 112;
    char Z = 123;
    int e = 131;
    char c = 129;
    char d = 130;
}
```


```asm
main:
        push    rbp
        mov     rbp, rsp
        mov     BYTE PTR [rbp-1], 127
        mov     BYTE PTR [rbp-2], -128
        mov     BYTE PTR [rbp-3], 111
        mov     BYTE PTR [rbp-4], 112
        mov     BYTE PTR [rbp-5], 123
        mov     DWORD PTR [rbp-12], 131
        mov     BYTE PTR [rbp-13], -127
        mov     BYTE PTR [rbp-14], -126
        nop
        pop     rbp
        ret
```

If you are still visualizing the way I displayed above, now we have 3 bytes space in the row 2. This space is added by the compiler and is termed as **padding**. We can not always avoid padding, but a good programmer can arrange the variables in a way to minimize this.


That was a long detour, but definetily worth it. I'll cover more examples in later parts of this series. Till then, ciao!!


![](https://media.giphy.com/media/ie6l26rkxYA885tLqo/giphy.gif#center)
