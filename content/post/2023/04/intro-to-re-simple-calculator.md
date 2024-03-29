---
title: "Intro to RE: C : A Simple Calculator"
date: 2023-04-03T21:59:48+05:30
draft: false
showtoc: false
tags: [RE, linux, C programming]
series: ["RE:C"]
description: How to reverse engineer a simple calculator program from scratch
# math: true
# ShowBreadCrumbs: false
# searchHidden: true
# hideMeta: true
---


We covered a wide range of topics in earlier articles that were helpful in comprehending how many lower-level processes operate. This blog will concentrate on applying those ideas to recreate C program after reverse engineering a simple calculator binary.

![](https://media.giphy.com/media/Pmv6m86yGQCjkLjmqx/giphy.gif#center)


It is always a good idea to observe how the target software responds to various inputs. This gives you a sense of the internal logic that might be operating.

If we run this program without any arguments, we will get an error message stating that we need to pass more arguments as well as the usage guide is printed.

```txt{linenos=false}
❯ ./calc

Not enough arguments passed
Usage: ./calc <num1> <operator> <num2>

```

So I can assume that there are checks in place which will see if we've passed enough arguments. If not, it'll exit with the above message.

So its obvious for anybody now to try it with the arguments this time.

```txt{linenos=false}
❯ ./calc 5 + 3
5 + 3 = 8


❯ ./calc 100 / 5
100 / 5 = 20
```

This works now and gives us the required output in a good looking way. That should do it for now.

It's time to open up our hacker tools and disassemble the binary.

Disassembly:-

```asm
addFunc:
        push    rbp
        mov     rbp, rsp
        mov     DWORD PTR [rbp-4], edi
        mov     DWORD PTR [rbp-8], esi
        mov     edx, DWORD PTR [rbp-4]
        mov     eax, DWORD PTR [rbp-8]
        add     eax, edx
        pop     rbp
        ret
subFunc:
        push    rbp
        mov     rbp, rsp
        mov     DWORD PTR [rbp-4], edi
        mov     DWORD PTR [rbp-8], esi
        mov     eax, DWORD PTR [rbp-4]
        sub     eax, DWORD PTR [rbp-8]
        pop     rbp
        ret
mulFunc:
        push    rbp
        mov     rbp, rsp
        mov     DWORD PTR [rbp-4], edi
        mov     DWORD PTR [rbp-8], esi
        mov     eax, DWORD PTR [rbp-4]
        imul    eax, DWORD PTR [rbp-8]
        pop     rbp
        ret
divFunc:
        push    rbp
        mov     rbp, rsp
        mov     DWORD PTR [rbp-4], edi
        mov     DWORD PTR [rbp-8], esi
        mov     eax, DWORD PTR [rbp-4]
        cdq
        idiv    DWORD PTR [rbp-8]
        pop     rbp
        ret
.LC0:
        .string "Not enough arguments passed"
.LC1:
        .string "Usage: ./calc <num1> <operator> <num2>"
die:
        push    rbp
        mov     rbp, rsp
        mov     edi, OFFSET FLAT:.LC0
        call    puts
        mov     edi, OFFSET FLAT:.LC1
        call    puts
        mov     edi, 1
        call    exit
.LC2:
        .string "%d %c %d = %d\n"
main:
        push    rbp
        mov     rbp, rsp
        sub     rsp, 48
        mov     DWORD PTR [rbp-36], edi
        mov     QWORD PTR [rbp-48], rsi
        cmp     DWORD PTR [rbp-36], 3
        jg      .L11
        mov     eax, 0
        call    die
.L11:
        mov     rax, QWORD PTR [rbp-48]
        add     rax, 8
        mov     rax, QWORD PTR [rax]
        mov     rdi, rax
        call    atoi
        mov     DWORD PTR [rbp-12], eax
        mov     rax, QWORD PTR [rbp-48]
        add     rax, 24
        mov     rax, QWORD PTR [rax]
        mov     rdi, rax
        call    atoi
        mov     DWORD PTR [rbp-16], eax
        mov     rax, QWORD PTR [rbp-48]
        add     rax, 16
        mov     rax, QWORD PTR [rax]
        movzx   eax, BYTE PTR [rax]
        mov     BYTE PTR [rbp-17], al
        movsx   eax, BYTE PTR [rbp-17]
        cmp     eax, 47
        je      .L12
        cmp     eax, 47
        jg      .L13
        cmp     eax, 45
        je      .L14
        cmp     eax, 45
        jg      .L13
        cmp     eax, 42
        je      .L15
        cmp     eax, 43
        jne     .L13
        mov     QWORD PTR [rbp-8], OFFSET FLAT:addFunc
        jmp     .L13
.L14:
        mov     QWORD PTR [rbp-8], OFFSET FLAT:subFunc
        jmp     .L13
.L15:
        mov     QWORD PTR [rbp-8], OFFSET FLAT:mulFunc
        jmp     .L13
.L12:
        mov     QWORD PTR [rbp-8], OFFSET FLAT:divFunc
        nop
.L13:
        mov     edx, DWORD PTR [rbp-16]
        mov     eax, DWORD PTR [rbp-12]
        mov     rcx, QWORD PTR [rbp-8]
        mov     esi, edx
        mov     edi, eax
        call    rcx
        mov     esi, eax
        movsx   edx, BYTE PTR [rbp-17]
        mov     ecx, DWORD PTR [rbp-16]
        mov     eax, DWORD PTR [rbp-12]
        mov     r8d, esi
        mov     esi, eax
        mov     edi, OFFSET FLAT:.LC2
        mov     eax, 0
        call    printf
        nop
        leave
        ret
```

This is far too large to handle at once. So I'll start with smaller functions and see what they do before moving on to the `main` function.


*(In my opinion, there is no specific reverse engineering flow. You could start from anywhere that suits your needs and the project at hand.)*


Now the first function I'll start with is going to be the `addFunc`.


```asm
addFunc:
        push    rbp
        mov     rbp, rsp
        mov     DWORD PTR [rbp-4], edi
        mov     DWORD PTR [rbp-8], esi
        mov     edx, DWORD PTR [rbp-4]
        mov     eax, DWORD PTR [rbp-8]
        add     eax, edx
        pop     rbp
        ret
```

The first three lines are simply the function's label and prologue.


Then there are two `mov` statements (lines 4 and 5) that involve `edi` (first argument) and `esi` (second argument) (second argument). They are the function arguments passed from the calling function to this function. These values are then saved in the `[rbp-4]` and `[rbp-8]` local variables.

Given the variable size requirements (4 bytes each), it is safe to assume that the passed variables are of the `int` type. That is, we are passing this function two `int` values.

Then, at lines 6, 7, and 8, we simply load the values from the variables into some registers and then add them up. In this case, the result of the add instruction will be stored in the `eax` register. Remember, this is also the register where the return value of a function is stored. So when we return back from this function, we have our addition result in the `eax` register.

Finally, at line 9 & 10, we make a graceful return from the function.

Similar behaviour is followed by next 3 functions - `subFunc`, `mulFunc` and `divFunc`.

```asm
subFunc:
        ; Prologue
        push    rbp
        mov     rbp, rsp
        ; Function arguments
        mov     DWORD PTR [rbp-4], edi
        mov     DWORD PTR [rbp-8], esi
        ; Load values and perform subtraction
        mov     eax, DWORD PTR [rbp-4]
        sub     eax, DWORD PTR [rbp-8]
        ; Return value is already in EAX, just return
        pop     rbp
        ret


mulFunc:
        ; Prologue
        push    rbp
        mov     rbp, rsp
        ; Function arguments
        mov     DWORD PTR [rbp-4], edi
        mov     DWORD PTR [rbp-8], esi
        ; Load values and perform multiplication
        mov     eax, DWORD PTR [rbp-4]
        imul    eax, DWORD PTR [rbp-8]
        ; Return value is already in EAX, just return
        pop     rbp
        ret


divFunc:
        ; Prologue
        push    rbp
        mov     rbp, rsp
        ; Function arguments
        mov     DWORD PTR [rbp-4], edi
        mov     DWORD PTR [rbp-8], esi
        ; Load values and perform division
        mov     eax, DWORD PTR [rbp-4]
        cdq
        idiv    DWORD PTR [rbp-8]
        ; Return value is already in EAX, just return
        pop     rbp
        ret
```


Now we are done with all the functions that perform the computations, next comes the function which prints the help message and then exits.

```asm
.LC0:
        .string "Not enough arguments passed"
.LC1:
        .string "Usage: ./calc <num1> <operator> <num2>"
die:
        push    rbp
        mov     rbp, rsp
        mov     edi, OFFSET FLAT:.LC0
        call    puts
        mov     edi, OFFSET FLAT:.LC1
        call    puts
        mov     edi, 1
        call    exit
```

The name implies that the purpose of this function is to terminate the execution of the `calc` program. Lines 8-9 show that this function is printing something using the `puts` function. The function's argument is a string starting at offset `.LC0` - `"Insufficient arguments passed"`. Lines 9-10 contain another call for `puts` with an argument from offset `.LC1` - `"Usage: ./calc <num1> <operator> <num2>"`.

These both `puts` statements combined give the error message we got when we tried to execute the program without any arguments.

Finally, it terminates with an `exit` function call (no return), and the function's argument was the integer value `1`. Typically, the return value for successful execution is zero, and all non-zero values denote some type of execution error. **So exiting with 1 indicates an error.**


Now we know what each function do at atomic level. Let's load our big guns and take a shot at `main()` function.

```asm
.LC2:
        .string "%d %c %d = %d\n"
main:
        push    rbp
        mov     rbp, rsp
        sub     rsp, 48
        mov     DWORD PTR [rbp-36], edi
        mov     QWORD PTR [rbp-48], rsi
        cmp     DWORD PTR [rbp-36], 3
        jg      .L11
        mov     eax, 0
        call    die
.L11:
        mov     rax, QWORD PTR [rbp-48]
        add     rax, 8
        mov     rax, QWORD PTR [rax]
        mov     rdi, rax
        call    atoi
        mov     DWORD PTR [rbp-12], eax
        mov     rax, QWORD PTR [rbp-48]
        add     rax, 24
        mov     rax, QWORD PTR [rax]
        mov     rdi, rax
        call    atoi
        mov     DWORD PTR [rbp-16], eax
        mov     rax, QWORD PTR [rbp-48]
        add     rax, 16
        mov     rax, QWORD PTR [rax]
        movzx   eax, BYTE PTR [rax]
        mov     BYTE PTR [rbp-17], al
        movsx   eax, BYTE PTR [rbp-17]
        cmp     eax, 47
        je      .L12
        cmp     eax, 47
        jg      .L13
        cmp     eax, 45
        je      .L14
        cmp     eax, 45
        jg      .L13
        cmp     eax, 42
        je      .L15
        cmp     eax, 43
        jne     .L13
        mov     QWORD PTR [rbp-8], OFFSET FLAT:addFunc
        jmp     .L13
.L14:
        mov     QWORD PTR [rbp-8], OFFSET FLAT:subFunc
        jmp     .L13
.L15:
        mov     QWORD PTR [rbp-8], OFFSET FLAT:mulFunc
        jmp     .L13
.L12:
        mov     QWORD PTR [rbp-8], OFFSET FLAT:divFunc
        nop
.L13:
        mov     edx, DWORD PTR [rbp-16]
        mov     eax, DWORD PTR [rbp-12]
        mov     rcx, QWORD PTR [rbp-8]
        mov     esi, edx
        mov     edi, eax
        call    rcx
        mov     esi, eax
        movsx   edx, BYTE PTR [rbp-17]
        mov     ecx, DWORD PTR [rbp-16]
        mov     eax, DWORD PTR [rbp-12]
        mov     r8d, esi
        mov     esi, eax
        mov     edi, OFFSET FLAT:.LC2
        mov     eax, 0
        call    printf
        nop
        leave
        ret
```


The `main` function still appears to be too large to handle all at once, so I'll cut it into smaller chunks to bite off and digest properly.

The prologue comes first... setting up the function frame; nothing new here.

```asm
main:
        push    rbp
        mov     rbp, rsp
```

Then we make some space in the frame to store the local variables.

```asm
        sub     rsp, 48
```


And then storing `edi` and `rsi` values in there. If you remember these two registers indicate the first two arguments passed to a function. In this case, the arguments to main function will be the `argc` and `argv`.

- `argc` --> Count of the cli arguments passed to it.
- `argv` --> Pointer to the list of arguments passed.

Next up is the comparision...which will check the count of args. This decides the branch the program execution takes - die or live.

```asm
        cmp     DWORD PTR [rbp-36], 3
        jg      .L11
        mov     eax, 0
        call    die
```

This compares `[rbp-36]` (`argc`) with hardcoded value of `3`. If the value from `argc` is greater than `3` then it'll make a jump to `.L11`, otherwise it'll call `die` function... which will print some messages and then `exit` with status `1`. We already looked at that.


So if we write a C program for what we know about `main` till now, we'll get something like:

```c
void main(int argc, char* argv[])
{
        if (argc < 3) {
                die();
        }
        else {
                /* Keep on going to know more */
        }
}
```

Next statements are something new, so you'll have to believe me when I say that this is equivalent to `argv[1]`.

```asm
.L11:
        mov     rax, QWORD PTR [rbp-48]
        add     rax, 8
        mov     rax, QWORD PTR [rax]
```

Before I start explaining, take a look at this diagram.


```goat


                                                        ┌──────────────────────┐
                                                        │                      │
                                                        ├──────────────────────┤
                                                        │                      │
                                                        ├──────────────────────┤
               ┌────────────────────────────►   rbp-48  │ some_pointer_value   │ ─────────┐
               │                                        ├──────────────────────┤          │
               │                                        │                      │          │
               │                                        ├──────────────────────┤          │
   rax                                                  │                      │          │
      ┌────────────────────┐                            ├──────────────────────┤          │
      │                    │                            │                      │          │
      └────────────────────┘                            │                      │          │
                                                        │                      │          │
                                                        │                      │          │
                                                        │                      │          ▼
                                                        ├──────────────────────┤
                                                        │ ptr_to_prog_name     │ some_pointer_value
                                                        ├──────────────────────┤
                                                        │  ptr_to_arg1         │ some_pointer_value + 8
                                                        ├──────────────────────┤
                                                        │  ptr_to_arg2         │ some_pointer_value + 16
                                                        ├──────────────────────┤
                                                        │  ptr_to+arg3         │ some_pointer_value + 24
                                                        ├──────────────────────┤
                                                        │                      │
                                                        │                      │
                                                        │                      │



```

Now with this in your mind, we can start to understand the above 3 instructions.

- `mov     rax, QWORD PTR [rbp-48]`

This instruction loads the `some_pointer_value` into `rax` register.

- `add     rax, 8`

This adds up `8` to `rax` value. That means now the resultant value is `some_pointer_value + 8`. If you don't know it already, `8` is the size of a pointer on most x86_64 machines. So if we want to add for 2 pointers, we'll need something like `some_pointer_value + 16`.

- `mov     rax, QWORD PTR [rax]`

Now we load the value from that location. In C language, this would be equivalent to `*(argv + 1)` or `argv[1]` or if you are feeling funny `1[argv]`. **[HOW??](https://stackoverflow.com/questions/381542/with-arrays-why-is-it-the-case-that-a5-5a)** [^array]

[^array]: https://stackoverflow.com/questions/381542/with-arrays-why-is-it-the-case-that-a5-5a


Now, next instruction calls `atoi` function with the `argv[1]` as it's first argument.

```asm
        mov     rdi, rax
        call    atoi
```

`atoi` function changes character value to respective integer value. As an example, `'1'` (in char) will be converted to `1` (in int).

```asm
        mov     DWORD PTR [rbp-12], eax
```

And then whatever is returned by that function will be stored in a local variable pointed by `rbp-12`. Remember, `eax` register is used to store the return values from called functions (`atio`) to caller function (`main`).

Next set of instructions is quite similar to what we just saw.

```asm
        mov     rax, QWORD PTR [rbp-48]
        add     rax, 24
        mov     rax, QWORD PTR [rax]
        mov     rdi, rax
        call    atoi
        mov     DWORD PTR [rbp-16], eax
```

If you notice, here we are adding `24`*(8 * 3 = 24)* so that means `arg3` is being used - `argv[3]`. Till this point we have successfully converted `argv[1]` and `argv[3]` to integers. These are the number1 and number2 values for our litte calculator.

Now `argv[2]`... our operator character... 1 byte value.


```asm
        mov     rax, QWORD PTR [rbp-48]
        add     rax, 16
        mov     rax, QWORD PTR [rax]
        movzx   eax, BYTE PTR [rax]
        mov     BYTE PTR [rbp-17], al
```

This adds `16` - means `argv[2]`. So we load the value and then just pick up the lowest 1 byte value `al` from the whole thing and store it in `[rbp-17]` location.


Let's update our C program with the new findings.

```c
void main(int argc, char* argv[])
{
        if (argc < 3) {
                die();
        }
        else {
                int num1 = atoi(argv[1]);
                int num2 = atoi(argv[3]);
                char op = argv[2];
                /* Keep on going to know more */
        }
}
```

Next few lines of disassembly code looks like a conditional branch.... So many "compare and jump" instructions.


```asm
        movsx   eax, BYTE PTR [rbp-17]
        cmp     eax, 47
        je      .L12
        cmp     eax, 47
        jg      .L13
        cmp     eax, 45
        je      .L14
        cmp     eax, 45
        jg      .L13
        cmp     eax, 42
        je      .L15
        cmp     eax, 43
        jne     .L13
        mov     QWORD PTR [rbp-8], OFFSET FLAT:addFunc
        jmp     .L13
.L14:
        mov     QWORD PTR [rbp-8], OFFSET FLAT:subFunc
        jmp     .L13
.L15:
        mov     QWORD PTR [rbp-8], OFFSET FLAT:mulFunc
        jmp     .L13
.L12:
        mov     QWORD PTR [rbp-8], OFFSET FLAT:divFunc
        nop
```

The value being compared is stored in `[rbp-17]` in this case. This contains the operator character from `argv[2]`. If you convert the values this is compared to to char equivalents, you'll get the following:

- 47 is `/`,
- 45 is `-`,
- 42 is `*`, and
- 43 is `+`.

With some calculations, you can conclude where the control will jump. To sum up,

- if the operator is 47 (`/`), then it'll jump to `.L12`. Which will move the offset location for `divFunc` function to a local variable.

- if the operator is 45 (`-`), then it'll move offset location for `subFunc` function to that local variable.

- if the operator is 42 (`*`), then it'll move offset location for `mulFunc` function to that local variable.

- if the operator is 42 (`+`), then it'll move offset location for `addFunc` function to that local variable.

- In any other given input, it'll simply move forward to `.L13`.



```asm
.L13:
        mov     edx, DWORD PTR [rbp-16]
        mov     eax, DWORD PTR [rbp-12]
        mov     rcx, QWORD PTR [rbp-8]
        mov     esi, edx
        mov     edi, eax
        call    rcx
        mov     esi, eax

        movsx   edx, BYTE PTR [rbp-17]
        mov     ecx, DWORD PTR [rbp-16]
        mov     eax, DWORD PTR [rbp-12]
        mov     r8d, esi
        mov     esi, eax
        mov     edi, OFFSET FLAT:.LC2
        mov     eax, 0
        call    printf
        nop
        leave
        ret
```


Before reaching `.L13`, we have 4 variables in our program:-

- number1 - stored at `[rbp-12]`
- number2 - stored at `[rbp-16]`
- operator - stored at `[rbp-17]`
- pointer to the function which is selected on the basis of operator. This is stored at `[rbp-8]`.



Now that we're in `.L13`,

We assign the `number1`, `number2`, and the `function pointer` to `eax`, `edx`, and `rcx`, respectively. The `function` is then called with the arguments `number1` and `number2`. Finally, the outcome is saved.


The remainder of the `.L13` consists entirely of calling a `printf` function with a format string and other local variables. The final result will look like this: `<number1> <operator> <number2> = <result_from_function>`. I'll leave it to you to dissect and solve the final puzzle piece. After printing, the function then exits gracefully.


Now we can put everything together, and the complete code should look something like this.


```c
#include <stdio.h>
#include <stdlib.h>

// Function to perform addition
int addFunc(int a, int b) {
    return a + b;
}

// Function to perform subtraction
int subFunc(int a, int b) {
    return a - b;
}

// Function to perform multiplication
int mulFunc(int a, int b) {
    return a * b;
}

// Function to perform division
int divFunc(int a, int b) {
    return a / b;
}

// Function to print usage message and exit
void die() {
    printf("Not enough arguments passed\n");
    printf("Usage: ./calc <num1> <operator> <num2>\n");
    exit(1);
}

// main function
void main(int argc, char *argv[])
{
    if(argc < 4) die();

    int x = atoi(argv[1]);
    int y = atoi(argv[3]);
    char option = *argv[2];
    int (*fp) (int, int);

    switch(option) {
        case '+': {
                      fp = addFunc;
                      break;
                  }

        case '-': {
                      fp = subFunc;
                      break;
                  }

        case '*': {
                      fp = mulFunc;
                      break;
                  }

        case '/': {
                      fp = divFunc;
                      break;
                  }

    }

    printf("%d %c %d = %d\n", x, option, y, fp(x,y));
}
```


This may not be what the original developer wrote, but it will certainly behave the same. That is the entire point of reverse engineering. Dissecting something with your tools to understand how it behaves and then creating something that mimics the behavior.

I'll encourage you to go and try to reverse engineer some more binaries. You can build your own if you want or just download some from online platforms like [crackme.one](https://crackmes.one/). Have fun!!

![](https://media.giphy.com/media/OB4Sjggq8aMJnq4sLQ/giphy.gif#center)
