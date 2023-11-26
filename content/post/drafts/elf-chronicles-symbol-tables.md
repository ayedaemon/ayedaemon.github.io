---
title: "Elf Chronicles: Symbol Tables"
date: 2023-10-29T22:15:08+05:30
draft: true
# showtoc: false
tags:
    - "C"
    - "ELF"
    - "RE"
series:
    - "ELF Chronicles"
description: "Exploring ELF symbol tables"
# math: true
# ShowBreadCrumbs: false
# searchHidden: true
# hideMeta: true
---


## ... prologue

At this point I hope you have a general idea of how a C program goes through multiple stages/passes and finally an ELF file is generated. Below is a diagram to jog your memory on this

```

  ┌──────────────────┐
  │                  │
  │    hello.c       │  // C source
  │                  │
  └────────┬─────────┘
           │
           │
           │ /* Compile */
           │
           │
           │
           ▼
  ┌──────────────────┐
  │                  │
  │    hello.s       │  // assembler source
  │                  │
  └────────┬─────────┘
           │
           │
           │ /* assemble */
           │
           │
           ▼
  ┌──────────────────┐
  │                  │
  │     hello.o      │  // Assembled program (ELF - relocatable)
  │                  │
  └────────┬─────────┘
           │
           │
           │ /* link */
           │
           │
           ▼
  ┌──────────────────┐
  │                  │
  │      hello       │  // Executable binary (ELF - executable)
  │                  │
  └──────────────────┘
```

Creating a simple *hello* program is very straight-forward, let me show you how this flow works when we are building something that has more than 1 source file. This is generally what most of the "real-world" projects do, they create multiple files with different functionalities and then merge them together to complete the program with the desired features only.


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



```c
/*
File: libarithmatic.c
*/

float addFunc (float a, float b) {
    return a + b;
}

float subFunc (float a, float b) {
    return a - b;
}

float mulFunc (float a, float b) {
    return a * b;
}

float divFunc (float a, float b) {
    if (b == 0) {
        return 0.0;
    }
    return a / b;
}

```


```c
/*
File: libarithmatic.h
*/

#ifndef ARITHMATIC_H
#define ARITHMATIC_H

float addFunc (float, float);
float subFunc (float, float);
float mulFunc (float, float);
float divFunc (float, float);

float magicFunc (float a, float b);
#endif
```


```c
/*
File: main.c
*/

#include <stdio.h>
#include "libarithmatic.h"


int main() {
    float num1, num2, result;
    char operator;

    printf("Enter equation (9 * 6): ");
    scanf("%f %c %f", &num1, &operator, &num2);

    switch (operator) {
        case '+':
            result = addFunc(num1, num2);
            break;
        case '-':
            result = subFunc(num1, num2);
            break;
        case '*':
            result = mulFunc(num1, num2);
            break;
        case '/':
            result = divFunc(num1, num2);
            break;
        default:
            printf("Invalid operator\n");
            return 1;
    }

    printf("Result: %.2f\n", result);

    return 0;
}

```

Luckily gcc provides some features, that helps us to make this process easier.
```
❯ gcc --help
Usage: gcc [options] file...
Options:
<... OMITTED ...>
  -E                       Preprocess only; do not compile, assemble or link.
  -S                       Compile only; do not assemble or link.
  -c                       Compile and assemble, but do not link.

```

So if you follow these commands, you’ll be fine


```
# Compile + assemble -> generates main.o
gcc -c main.c

# Compile + assemble -> generates libarithmatic.o
gcc -c libarithmatic.c

# Linking -> generates calc
gcc main.o libarithmatic.o -o calc
```


![](https://media.giphy.com/media/9wG8hpQRkHMoDbCqzu/giphy.gif#center)



This is our first time so far writing multiple files for a program. So let's take a moment to understand how this works.

First, we create a `libarithmatic.c` file with all of the required arithmatic functions - `addFunc`, `subFunc`, `mulFunc`, and `divFunc`. Since this file contains these functions (function definitions), the intermediate object file for this file will have related information as well.


Then comes the `main.c` file, where we have declared the `main` function. Inside the main function, we have used arithmatic functions which are not defined in this file. This will give an error at compilation time when those functions will not be found, so as a promise we give a declaration that these functions are present somewhere and they will be found in later steps by linker. Here those definitions are present in `libarithmatic.h` file -- header file for `libarithmatic.c`.


So when we are compiling `libarithmatic.c`, it'll create a `libarithmatic.o` file which will have 4 arithmatic functions as defined. On the other hand, `main.c` will generate a `main.o` file that will have a `main` function which will be trying to call the arithmatic functions - `addFunc`, `subFunc`, `mulFunc`, and `divFunc`.


**Question** - How did `main.o` call these functions when the address of these functions is not known to the compiler??

**Answer** - Compiler takes `main.c` and `libarithmatic.h` (a promise that these will be present when linking), and then generates the `main.o` with all of the `call` instructions... but because of the fact that it does not know the address of the functions to be called these addresses are left blank. These blanks will be filled by linker during `relocation` process.

Here is a proof that all of them are empty before linking and have all of the addresses fixed up after linking

```
## Before linking - main.o
❯ objdump -M intel -D -j .text main.o | grep call
 26:        e8 00 00 00 00          call   2b <main+0x2b>
 49:        e8 00 00 00 00          call   4e <main+0x4e>
 86:        e8 00 00 00 00          call   8b <main+0x8b>
 a3:        e8 00 00 00 00          call   a8 <main+0xa8>
 c0:        e8 00 00 00 00          call   c5 <main+0xc5>
 dd:        e8 00 00 00 00          call   e2 <main+0xe2>
 f5:        e8 00 00 00 00          call   fa <main+0xfa>
123:        e8 00 00 00 00          call   128 <main+0x128>
13c:        e8 00 00 00 00          call   141 <main+0x141>

## After linking - calc
❯ objdump -M intel -D -j .text calc | grep call
1138:       e8 63 ff ff ff          call   10a0 <_start+0x30>
118f:       e8 bc fe ff ff          call   1050 <printf@plt>
11b2:       e8 a9 fe ff ff          call   1060 <__isoc99_scanf@plt>
11ef:       e8 b8 00 00 00          call   12ac <addFunc>
120c:       e8 b5 00 00 00          call   12c6 <subFunc>
1229:       e8 b2 00 00 00          call   12e0 <mulFunc>
1246:       e8 af 00 00 00          call   12fa <divFunc>
125e:       e8 cd fd ff ff          call   1030 <puts@plt>
128c:       e8 bf fd ff ff          call   1050 <printf@plt>
12a5:       e8 96 fd ff ff          call   1040 <__stack_chk_fail@plt>
```



## Symbols and symbol tables

**Now the question is that how does linker know which blanks to fill and how to fill them??** ...here comes the role of *symbols* and *symbol tables*.


When writing a program, we often use "names" to reference "objects" in our code, like function "names" and variable "names". These "names" are commonly referred to as `symbols`. (*yeah, deal with it now!*)


Keep in mind that not all "names" are symbols. For example, a local variables to a function won't be treated as symbols. If you think it through, you don't need linker to handle that data so what's the point of adding that info as a symbol, right?

Another worth noting thing is that unlike string tables, symbol tables have a well-defined structure, and both Glibc and the Linux kernel define a struct for this (`Elf64_Sym` for 64-bit files).


```c
/*
Glibc
https://sourceware.org/git/?p=glibc.git;a=blob;f=elf/elf.h;hb=2bd00179885928fd95fcabfafc50e7b5c6e660d2#l530
*/

typedef struct
{
  Elf64_Word    st_name;                /* Symbol name (string tbl index) */
  unsigned char st_info;                /* Symbol type and binding */
  unsigned char st_other;               /* Symbol visibility */
  Elf64_Section st_shndx;               /* Section index */
  Elf64_Addr    st_value;               /* Symbol value */
  Elf64_Xword   st_size;                /* Symbol size */
} Elf64_Sym;

/*
Linux kernel v6.5.8
https://elixir.bootlin.com/linux/v6.5.8/source/include/uapi/linux/elf.h#L197
*/

typedef struct elf64_sym {
  Elf64_Word    st_name;		/* Symbol name, index in string tbl */
  unsigned char	st_info;	    /* Type and binding attributes */
  unsigned char	st_other;	    /* No defined meaning, 0 */
  Elf64_Half    st_shndx;		/* Associated section index */
  Elf64_Addr    st_value;		/* Value of the symbol */
  Elf64_Xword   st_size;		/* Associated symbol size */
} Elf64_Sym;

```

Let's see what each member of this struct resembles

### st_name

Similar to other name fields in the ELF specification, this member stores the **index** or **offset** in the associated string table.

### st_info

This member represents a combined value derived from two different but related attributes: `bind` and `type`.

Both, [Linux Kernel](https://elixir.bootlin.com/linux/v6.5.8/source/include/uapi/linux/elf.h#L136) and [glibc](https://sourceware.org/git/?p=glibc.git;a=blob;f=elf/elf.h;hb=2bd00179885928fd95fcabfafc50e7b5c6e660d2#l572) provide definitions and macros to work with this member.

#### 1. Bind

The "bind" bits provide information about where this symbol can be seen and used... There are 3 kinds of binding defined by linux kernel

```c
/*
https://elixir.bootlin.com/linux/v6.5.8/source/include/uapi/linux/elf.h#L123
*/
#define STB_LOCAL  0    /* not visible outside the object file */
#define STB_GLOBAL 1    /* visible to all object files */
#define STB_WEAK   2    /* like globals, but with lower precedence */
```

But glibc adds few more to this list

```c
/*
https://sourceware.org/git/?p=glibc.git;a=blob;f=elf/elf.h;hb=2bd00179885928fd95fcabfafc50e7b5c6e660d2#l582
*/
#define STB_LOCAL       0               /* Local symbol */
#define STB_GLOBAL      1               /* Global symbol */
#define STB_WEAK        2               /* Weak symbol */
#define STB_NUM         3               /* Number of defined types.  */
#define STB_LOOS        10              /* Start of OS-specific */
#define STB_GNU_UNIQUE  10              /* Unique symbol.  */
#define STB_HIOS        12              /* End of OS-specific */
#define STB_LOPROC      13              /* Start of processor-specific */
#define STB_HIPROC      15              /* End of processor-specific */
```

Kernel and glibc both provide a macro to extract the `bind` value from the provided `st_info` member - `#define ELF_ST_BIND(x)	((x) >> 4)`

#### 2. Type

`type` bits tells about the type of symbol - function, file, variable, etc. One could say -- A general classification for the symbol.

Linux kernel defines total 7 types

```c
/*
https://elixir.bootlin.com/linux/v6.5.8/source/include/uapi/linux/elf.h#L128
*/

#define STT_NOTYPE  0   /* Unspecified */
#define STT_OBJECT  1   /* data objects like variables, arrays, etc*/
#define STT_FUNC    2   /* functions or other executable codes*/
#define STT_SECTION 3   /* associated with a section;
                           mainly used for relocations
                           (we'll see relocations in later articles)*/
#define STT_FILE    4   /* name of the source file*/
#define STT_COMMON  5   /* just like STT_OBJECT, but for tentative values */
#define STT_TLS     6   /* stores thread local data which is unique to each thread */
```

And again our beloved glibc expanded these definitions

```c
/*
https://sourceware.org/git/?p=glibc.git;a=blob;f=elf/elf.h;hb=2bd00179885928fd95fcabfafc50e7b5c6e660d2#l595
*/
#define STT_NOTYPE      0               /* Symbol type is unspecified */
#define STT_OBJECT      1               /* Symbol is a data object */
#define STT_FUNC        2               /* Symbol is a code object */
#define STT_SECTION     3               /* Symbol associated with a section */
#define STT_FILE        4               /* Symbol's name is file name */
#define STT_COMMON      5               /* Symbol is a common data object */
#define STT_TLS         6               /* Symbol is thread-local data object*/
#define STT_NUM         7               /* Number of defined types.  */
#define STT_LOOS        10              /* Start of OS-specific */
#define STT_GNU_IFUNC   10              /* Symbol is indirect code object */
#define STT_HIOS        12              /* End of OS-specific */
#define STT_LOPROC      13              /* Start of processor-specific */
#define STT_HIPROC      15              /* End of processor-specific */
```

Kernel and glibc both provide a macro to extract the `type` value from the provided `st_info` member - `#define ELF_ST_TYPE(x)    ((x) & 0xf)`

### st_other

If you examine the `Elf64_Sym` struct in both the kernel and Glibc *source code*, you'll notice that the kernel doesn't currently have any use case for this field and marks it as such. However, Glibc uses this field to track the visibility of the symbol.

```
/*
https://sourceware.org/git/?p=glibc.git;a=blob;f=elf/elf.h;hb=2bd00179885928fd95fcabfafc50e7b5c6e660d2#l626
*/
#define STV_DEFAULT     0               /* Default symbol visibility rules - as specified by symbol binding*/
#define STV_INTERNAL    1               /* Processor specific hidden class */
#define STV_HIDDEN      2               /* Sym unavailable in other modules */
#define STV_PROTECTED   3               /* Not preemptible, not exported */
```

From what I understand, **symbol visibility** (*yup, this is what glibc calls `st_other`*) extends the concept of **symbol binding** and provides more control over symbol access.

You can read more about this member from [here](https://developer.ibm.com/articles/au-aix-symbol-visibility/) [^ibm_sym_vis] and [here](https://unix.stackexchange.com/questions/472660/what-are-difference-between-the-elf-symbol-visibility-levels) [^so_sym_vis].


[^ibm_sym_vis]: https://developer.ibm.com/articles/au-aix-symbol-visibility/

[^so_sym_vis]: https://unix.stackexchange.com/questions/472660/what-are-difference-between-the-elf-symbol-visibility-levels


### st_shndx

This attribute indicates the section associated with this symbol. It holds the section index corresponding to the sections in the section header.

### st_value

Indeed, each symbol should have both a name and an associated value. This member holds the value associated with the respective symbol.

### st_size

Many symbols come with associated sizes, for function type symbols this will be the size of that function. If a symbol doesn't have a size or its size is unknown, this member holds a value of zero.





## Analysis

Now that we have a foundational understanding, we can apply this knowledge to analyze our previous files.

### 1. `libarithmatic.o`

To keep things straightforward, I'll begin by listing all the sections in the `libarithmatic.o` file. (*This is the output from my parser, you can use hexdumps or any other parser of your choice...*)

```
[ 00 ] Section Name:                            Type: 0x0       Flags: 0x0      Addr: 0x0       Offset: 0x0             Size: 0         Link: 0         Info: 0x0       Addralign: 0x0          Entsize: 0
[ 01 ] Section Name: .text                      Type: 0x1       Flags: 0x6      Addr: 0x0       Offset: 0x40            Size: 130       Link: 0         Info: 0x0       Addralign: 0x1          Entsize: 0
[ 02 ] Section Name: .data                      Type: 0x1       Flags: 0x3      Addr: 0x0       Offset: 0xc2            Size: 0         Link: 0         Info: 0x0       Addralign: 0x1          Entsize: 0
[ 03 ] Section Name: .bss                       Type: 0x8       Flags: 0x3      Addr: 0x0       Offset: 0xc2            Size: 0         Link: 0         Info: 0x0       Addralign: 0x1          Entsize: 0
[ 04 ] Section Name: .comment                   Type: 0x1       Flags: 0x30     Addr: 0x0       Offset: 0xc2            Size: 28        Link: 0         Info: 0x0       Addralign: 0x1          Entsize: 1
[ 05 ] Section Name: .note.GNU-stack            Type: 0x1       Flags: 0x0      Addr: 0x0       Offset: 0xde            Size: 0         Link: 0         Info: 0x0       Addralign: 0x1          Entsize: 0
[ 06 ] Section Name: .note.gnu.property         Type: 0x7       Flags: 0x2      Addr: 0x0       Offset: 0xe0            Size: 48        Link: 0         Info: 0x0       Addralign: 0x8          Entsize: 0
[ 07 ] Section Name: .eh_frame                  Type: 0x1       Flags: 0x2      Addr: 0x0       Offset: 0x110           Size: 152       Link: 0         Info: 0x0       Addralign: 0x8          Entsize: 0
[ 08 ] Section Name: .rela.eh_frame             Type: 0x4       Flags: 0x40     Addr: 0x0       Offset: 0x288           Size: 96        Link: 9         Info: 0x7       Addralign: 0x8          Entsize: 24
[ 09 ] Section Name: .symtab                    Type: 0x2       Flags: 0x0      Addr: 0x0       Offset: 0x1a8           Size: 168       Link: 10        Info: 0x3       Addralign: 0x8          Entsize: 24
[ 10 ] Section Name: .strtab                    Type: 0x3       Flags: 0x0      Addr: 0x0       Offset: 0x250           Size: 49        Link: 0         Info: 0x0       Addralign: 0x1          Entsize: 0
[ 11 ] Section Name: .shstrtab                  Type: 0x3       Flags: 0x0      Addr: 0x0       Offset: 0x2e8           Size: 103       Link: 0         Info: 0x0       Addralign: 0x1          Entsize: 0
```



Now we can easily filter out the symbol table from this (`Type: 0x2`)


```
[ 09 ] Section Name: .symtab     Type: 0x2       Flags: 0x0      Addr: 0x0       Offset: 0x1a8           Size: 168       Link: 10        Info: 0x3       Addralign: 0x8          Entsize: 24
```

If you go back and revisit the article about section headers and check the explaination about members, you'll be able to conclude this -- `.symtab` section is linked to `.strtab` section. So the offset values from `st_name` of symbol table can be resolved to proper strings using this string table.


```


           ┌─────────────────────────────────┐
           │                                 │
           │ [ 09 ] Section Name: .symtab    │
           │           Type: 0x2             │
           │           Flags: 0x0            │
           │           Addr: 0x0             │
           │           Offset: 0x1a8         │
           │           Size: 168             │
      ┌────┼────────── Link: 10              │
      │    │           Info: 0x3             │
      │    │           Addralign: 0x8        │
      │    │           Entsize: 24           │
      │    │                                 │
      │    │                                 │
      │    └─────────────────────────────────┘
      │
      │
      │
      │
      │    ┌─────────────────────────────────┐
      │    │                                 │
      └────┤► [ 10 ] Section Name: .strtab   │
           │            Type: 0x3            │
           │            Flags: 0x0           │
           │            Addr: 0x0            │
           │            Offset: 0x250        │
           │            Size: 49             │
           │            Link: 0              │
           │            Info: 0x0            │
           │            Addralign: 0x1       │
           │            Entsize: 0           │
           │                                 │
           │                                 │
           └─────────────────────────────────┘


```




Now we can begin with the interesting stuff and the first step will be to pull out the `.symtab` section and parse it.



```

############ Explaination #################
#
# xxd
#   -s 0x1a8            # start point (Offset: 0x1a8)
#   -l 168              # total length (Size: 168)
#   -c 24               # bytes per line (Entsize: 24) - I wanted to get each entry in a single line for uniformity
#   libarithmatic.o     # filename
#   | nl -v0            # line numbers starting from 0
#
#############################################

❯ xxd -s 0x1a8 -l 168 -c 24 libarithmatic.o | nl -v0
    0  000001a8: 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  ........................
    1  000001c0: 0100 0000 0400 f1ff 0000 0000 0000 0000 0000 0000 0000 0000  ........................
    2  000001d8: 0000 0000 0300 0100 0000 0000 0000 0000 0000 0000 0000 0000  ........................
    3  000001f0: 1100 0000 1200 0100 0000 0000 0000 0000 1a00 0000 0000 0000  ........................
    4  00000208: 1900 0000 1200 0100 1a00 0000 0000 0000 1a00 0000 0000 0000  ........................
    5  00000220: 2100 0000 1200 0100 3400 0000 0000 0000 1a00 0000 0000 0000  !.......4...............
    6  00000238: 2900 0000 1200 0100 4e00 0000 0000 0000 3400 0000 0000 0000  ).......N.......4.......

```


If we parse this data using the struct `Elf64_Sym`, we'll get something like this


```

                             typedef struct {

  +------------------------------Elf64_Word    st_name;
  |
  |        +---------------------unsigned char st_info;
  |        |
  |        |     +---------------unsigned char st_other;
  |        |     |
  |        |     |    +----------Elf64_Section st_shndx;
  |        |     |    |
  |        |     |    |          Elf64_Addr    st_value;----+
  |        |     |    |                                     |
  |        |     |    |          Elf64_Xword   st_size;-----+-----------------+
  |        |     |    |                                     |                 |
  |        |     |    |      } Elf64_Sym;                   |                 |
  |        |     |    |                                     |                 |
  |        |     |    |                                     |                 |
  |        |     |    +-------------------+                 |                 |
  |        |     |                        |                 |                 |
  |        |     +------------------+     |                 |                 |
  |        |                        |     |                 |                 |
  |        +-------------------+    |     |                 |                 |
  |                            |    |     |                 |                 |
  +-------------------+        |    |     |                 |                 |
                      |        |    |     |                 |                 |
                      v        v    v     v                 v                 v
Index |  Offset  |
    0 | 000001a8:| 0000 0000 | 00 | 00 | 0000 | 0000 0000 0000 0000 | 0000 0000 0000 0000 |
    1 | 000001c0:| 0100 0000 | 04 | 00 | f1ff | 0000 0000 0000 0000 | 0000 0000 0000 0000 |
    2 | 000001d8:| 0000 0000 | 03 | 00 | 0100 | 0000 0000 0000 0000 | 0000 0000 0000 0000 |
    3 | 000001f0:| 1100 0000 | 12 | 00 | 0100 | 0000 0000 0000 0000 | 1a00 0000 0000 0000 |
    4 | 00000208:| 1900 0000 | 12 | 00 | 0100 | 1a00 0000 0000 0000 | 1a00 0000 0000 0000 |
    5 | 00000220:| 2100 0000 | 12 | 00 | 0100 | 3400 0000 0000 0000 | 1a00 0000 0000 0000 |
    6 | 00000238:| 2900 0000 | 12 | 00 | 0100 | 4e00 0000 0000 0000 | 3400 0000 0000 0000 |

```


From my parser, I got this result

```
[  0 ] Name:                   Info: 0x00 (Bind: 0x0 | Type: 0x0)      Other: 0x0      Shndx: 0x0      Value: 0x000000000000   Size: 0x0
[  1 ] Name: libarithmatic.c   Info: 0x04 (Bind: 0x0 | Type: 0x4)      Other: 0x0      Shndx: 0xfff1   Value: 0x000000000000   Size: 0x0
[  2 ] Name:                   Info: 0x03 (Bind: 0x0 | Type: 0x3)      Other: 0x0      Shndx: 0x1      Value: 0x000000000000   Size: 0x0
[  3 ] Name: addFunc           Info: 0x18 (Bind: 0x1 | Type: 0x2)      Other: 0x0      Shndx: 0x1      Value: 0x000000000000   Size: 0x1a
[  4 ] Name: subFunc           Info: 0x18 (Bind: 0x1 | Type: 0x2)      Other: 0x0      Shndx: 0x1      Value: 0x00000000001a   Size: 0x1a
[  5 ] Name: mulFunc           Info: 0x18 (Bind: 0x1 | Type: 0x2)      Other: 0x0      Shndx: 0x1      Value: 0x000000000034   Size: 0x1a
[  6 ] Name: divFunc           Info: 0x18 (Bind: 0x1 | Type: 0x2)      Other: 0x0      Shndx: 0x1      Value: 0x00000000004e   Size: 0x34
```

![](https://media.giphy.com/media/S5tVD09nBR2nJWJMen/giphy.gif#center)

For the sake of simplicity and the scope of this article, I'll focus on discussing the four functions in this table and leave the rest for you to explore and learn.


We can observe that the `st_info` value for all of these symbols is the same, which implies that their "**bind**" and "**type**" values are identical (*duhh*). According to the information we've gathered, these symbols are `GLOBAL` (bind=0x1) and of `FUNC` (type=0x2) type. This indicates that these symbols are basically global functions and can be called from other files as well.


It's worth noting that there's a very cool tool called ["**ftrace**" by elfmaster](https://github.com/elfmaster/ftrace/), which utilizes [this information](https://github.com/elfmaster/ftrace/blob/master/ftrace.c#L441) to trace function calls, specifically focusing on function calls and not other symbols.



Furthermore, the `st_other` field is empty for these members, indicating `default` symbol visibility. There's nothing noteworthy to discuss here.

So we move on to the `sh_shndx` (section index) member. This member tells us that all of these symbols are associated with section `0x1` (which is `.text`, and that does make sense -- Code of these functions should be in `.text` section only).

The `st_value` field indicates the offset within the `.text` section at which these functions begin. So, if you start executing instructions from offset `0x34` in the `.text` section, you'll be running the `mulFunc` function. *Makes sense??*

The linker will perform `relocation` on the object files and generate a final executable binary that will have all the values in correct places. At that point we won't need the `mulFunc` string in our ELF file.

Last but not least, the `st_size` field provides the size of the function. This helps the magical entity reading the code determine when to stop and understand the boundaries of the function.


### 2. `main.o`


Performing the same initial process for the `main.o` file, you will be able yield its symbol table, as shown below.


```
 [ 11 ] Section Name: .symtab       Type: 0x2       Flags: 0x0      Addr: 0x0       Offset: 0x248           Size: 312       Link: 12        Info: 0x4       Addralign: 0x8          Entsize: 24
     [  0 ] Name:                    Info: 0x00 (Bind: 0x0 | Type: 0x0)      Other: 0x0      Shndx: 0x0      Value: 0x000000000000   Size: 0x0
     [  1 ] Name: main.c             Info: 0x04 (Bind: 0x0 | Type: 0x4)      Other: 0x0      Shndx: 0xfff1   Value: 0x000000000000   Size: 0x0
     [  2 ] Name:                    Info: 0x03 (Bind: 0x0 | Type: 0x3)      Other: 0x0      Shndx: 0x1      Value: 0x000000000000   Size: 0x0
     [  3 ] Name:                    Info: 0x03 (Bind: 0x0 | Type: 0x3)      Other: 0x0      Shndx: 0x5      Value: 0x000000000000   Size: 0x0
     [  4 ] Name: main               Info: 0x18 (Bind: 0x1 | Type: 0x2)      Other: 0x0      Shndx: 0x1      Value: 0x000000000000   Size: 0x143
     [  5 ] Name: printf             Info: 0x16 (Bind: 0x1 | Type: 0x0)      Other: 0x0      Shndx: 0x0      Value: 0x000000000000   Size: 0x0
     [  6 ] Name: __isoc99_scanf     Info: 0x16 (Bind: 0x1 | Type: 0x0)      Other: 0x0      Shndx: 0x0      Value: 0x000000000000   Size: 0x0
     [  7 ] Name: addFunc            Info: 0x16 (Bind: 0x1 | Type: 0x0)      Other: 0x0      Shndx: 0x0      Value: 0x000000000000   Size: 0x0
     [  8 ] Name: subFunc            Info: 0x16 (Bind: 0x1 | Type: 0x0)      Other: 0x0      Shndx: 0x0      Value: 0x000000000000   Size: 0x0
     [  9 ] Name: mulFunc            Info: 0x16 (Bind: 0x1 | Type: 0x0)      Other: 0x0      Shndx: 0x0      Value: 0x000000000000   Size: 0x0
     [ 10 ] Name: divFunc            Info: 0x16 (Bind: 0x1 | Type: 0x0)      Other: 0x0      Shndx: 0x0      Value: 0x000000000000   Size: 0x0
     [ 11 ] Name: puts               Info: 0x16 (Bind: 0x1 | Type: 0x0)      Other: 0x0      Shndx: 0x0      Value: 0x000000000000   Size: 0x0
     [ 12 ] Name: __stack_chk_fail   Info: 0x16 (Bind: 0x1 | Type: 0x0)      Other: 0x0      Shndx: 0x0      Value: 0x000000000000   Size: 0x0
```


In this case, things get a bit more interesting. Let's begin with the same set of symbols: `addFunc`, `subFunc`, `mulFunc`, and `divFunc`.

You'll notice that these symbols are global, but they don't have any associated types. This is expected since the symbols are not defined in this file; they are just being called. At this stage, we're not certain if there's anything like these symbols elsewhere, which is why all the other members are zeroed out (undefined). This essentially instructs the *magical* linker to locate the values of these symbols (linkers are pretty good at this; they will give errors if the symbols aren't found).

Now, you'll also notice the presence of `printf` and `puts` symbols. This may raise a question: "**I didn't use puts in my code, so why is it there?**"

Answer: It's compiler magic! The compiler observed that the line `printf("Enter equation (9 * 6): ");` could be expressed as `puts("Enter equation (9 * 6): ");`, so it made this conversion during compilation. To confirm this, you can generate the compiled code using `gcc -S` and check the `call` to `puts` function.


Now, let's examine our mighty `main` symbol. The `st_info` indicates that it's a `GLOBAL` `function` (with `bind=0x1` and `type=0x2`). This function is located in the 1st section (`sh_shndx: 0x1`) of `main.o`, which in our case is the `.text` section. The function begins at offset `0x0`, and its size is `0x143`. *Pretty simple, right?*

(Note: I'm leaving `__isoc99_scanf` and `__stack_chk_fail` for you. Google them!)



### 3. `calc`

This represents the ultimate outcome of the entire compilation, assembly, and linking process -- the final ELF executable binary. However, the process to obtain its **symbol table** remains same.

Here is the `symtab` for this ELF binary
```
[ 27 ] Section Name: .symtab       Type: 0x2       Flags: 0x0      Addr: 0x0       Offset: 0x3050          Size: 768       Link: 28        Info: 0x7       Addralign: 0x8          Entsize: 24
     [  0 ] Name:                                 Info: 0x00 (Bind: 0x0 | Type: 0x0)      Other: 0x0      Shndx: 0x0      Value: 0x000000000000   Size: 0x0
     [  1 ] Name: main.c                          Info: 0x04 (Bind: 0x0 | Type: 0x4)      Other: 0x0      Shndx: 0xfff1   Value: 0x000000000000   Size: 0x0
     [  2 ] Name: libarithmatic.c                 Info: 0x04 (Bind: 0x0 | Type: 0x4)      Other: 0x0      Shndx: 0xfff1   Value: 0x000000000000   Size: 0x0
     [  3 ] Name:                                 Info: 0x04 (Bind: 0x0 | Type: 0x4)      Other: 0x0      Shndx: 0xfff1   Value: 0x000000000000   Size: 0x0
     [  4 ] Name: _DYNAMIC                        Info: 0x01 (Bind: 0x0 | Type: 0x1)      Other: 0x0      Shndx: 0x15     Value: 0x000000003de0   Size: 0x0
     [  5 ] Name: __GNU_EH_FRAME_HDR              Info: 0x00 (Bind: 0x0 | Type: 0x0)      Other: 0x0      Shndx: 0x11     Value: 0x000000002048   Size: 0x0
     [  6 ] Name: _GLOBAL_OFFSET_TABLE_           Info: 0x01 (Bind: 0x0 | Type: 0x1)      Other: 0x0      Shndx: 0x17     Value: 0x000000003fe8   Size: 0x0
     [  7 ] Name: __libc_start_main@GLIBC_2.34    Info: 0x18 (Bind: 0x1 | Type: 0x2)      Other: 0x0      Shndx: 0x0      Value: 0x000000000000   Size: 0x0
     [  8 ] Name: _ITM_deregisterTMCloneTable     Info: 0x32 (Bind: 0x2 | Type: 0x0)      Other: 0x0      Shndx: 0x0      Value: 0x000000000000   Size: 0x0
     [  9 ] Name: data_start                      Info: 0x32 (Bind: 0x2 | Type: 0x0)      Other: 0x0      Shndx: 0x18     Value: 0x000000004020   Size: 0x0
     [ 10 ] Name: subFunc                         Info: 0x18 (Bind: 0x1 | Type: 0x2)      Other: 0x0      Shndx: 0xe      Value: 0x0000000012c6   Size: 0x1a
     [ 11 ] Name: puts@GLIBC_2.2.5                Info: 0x18 (Bind: 0x1 | Type: 0x2)      Other: 0x0      Shndx: 0x0      Value: 0x000000000000   Size: 0x0
     [ 12 ] Name: _edata                          Info: 0x16 (Bind: 0x1 | Type: 0x0)      Other: 0x0      Shndx: 0x18     Value: 0x000000004030   Size: 0x0
     [ 13 ] Name: _fini                           Info: 0x18 (Bind: 0x1 | Type: 0x2)      Other: 0x2      Shndx: 0xf      Value: 0x000000001330   Size: 0x0
     [ 14 ] Name: __stack_chk_fail@GLIBC_2.4      Info: 0x18 (Bind: 0x1 | Type: 0x2)      Other: 0x0      Shndx: 0x0      Value: 0x000000000000   Size: 0x0
     [ 15 ] Name: printf@GLIBC_2.2.5              Info: 0x18 (Bind: 0x1 | Type: 0x2)      Other: 0x0      Shndx: 0x0      Value: 0x000000000000   Size: 0x0
     [ 16 ] Name: addFunc                         Info: 0x18 (Bind: 0x1 | Type: 0x2)      Other: 0x0      Shndx: 0xe      Value: 0x0000000012ac   Size: 0x1a
     [ 17 ] Name: __data_start                    Info: 0x16 (Bind: 0x1 | Type: 0x0)      Other: 0x0      Shndx: 0x18     Value: 0x000000004020   Size: 0x0
     [ 18 ] Name: __gmon_start__                  Info: 0x32 (Bind: 0x2 | Type: 0x0)      Other: 0x0      Shndx: 0x0      Value: 0x000000000000   Size: 0x0
     [ 19 ] Name: __dso_handle                    Info: 0x17 (Bind: 0x1 | Type: 0x1)      Other: 0x2      Shndx: 0x18     Value: 0x000000004028   Size: 0x0
     [ 20 ] Name: _IO_stdin_used                  Info: 0x17 (Bind: 0x1 | Type: 0x1)      Other: 0x0      Shndx: 0x10     Value: 0x000000002000   Size: 0x4
     [ 21 ] Name: divFunc                         Info: 0x18 (Bind: 0x1 | Type: 0x2)      Other: 0x0      Shndx: 0xe      Value: 0x0000000012fa   Size: 0x34
     [ 22 ] Name: _end                            Info: 0x16 (Bind: 0x1 | Type: 0x0)      Other: 0x0      Shndx: 0x19     Value: 0x000000004038   Size: 0x0
     [ 23 ] Name: _start                          Info: 0x18 (Bind: 0x1 | Type: 0x2)      Other: 0x0      Shndx: 0xe      Value: 0x000000001070   Size: 0x26
     [ 24 ] Name: __bss_start                     Info: 0x16 (Bind: 0x1 | Type: 0x0)      Other: 0x0      Shndx: 0x19     Value: 0x000000004030   Size: 0x0
     [ 25 ] Name: mulFunc                         Info: 0x18 (Bind: 0x1 | Type: 0x2)      Other: 0x0      Shndx: 0xe      Value: 0x0000000012e0   Size: 0x1a
     [ 26 ] Name: main                            Info: 0x18 (Bind: 0x1 | Type: 0x2)      Other: 0x0      Shndx: 0xe      Value: 0x000000001169   Size: 0x143
     [ 27 ] Name: __isoc99_scanf@GLIBC_2.7        Info: 0x18 (Bind: 0x1 | Type: 0x2)      Other: 0x0      Shndx: 0x0      Value: 0x000000000000   Size: 0x0
     [ 28 ] Name: __TMC_END__                     Info: 0x17 (Bind: 0x1 | Type: 0x1)      Other: 0x2      Shndx: 0x18     Value: 0x000000004030   Size: 0x0
     [ 29 ] Name: _ITM_registerTMCloneTable       Info: 0x32 (Bind: 0x2 | Type: 0x0)      Other: 0x0      Shndx: 0x0      Value: 0x000000000000   Size: 0x0
     [ 30 ] Name: __cxa_finalize@GLIBC_2.2.5      Info: 0x34 (Bind: 0x2 | Type: 0x2)      Other: 0x0      Shndx: 0x0      Value: 0x000000000000   Size: 0x0
     [ 31 ] Name: _init                           Info: 0x18 (Bind: 0x1 | Type: 0x2)      Other: 0x2      Shndx: 0xc      Value: 0x000000001000   Size: 0x0
```


The linking process did introduce numerous symbols that exceed the combined count of symbols in both individual object files. To keep things simple (* *once again* *), we won't dive into the specifics of what these additional symbols do, and we can think of them as a result of linker magic.

Our primary focus for now remains on the symbols and their properties, even if we don't have detailed knowledge of their functions.

![](https://media.giphy.com/media/94OJTU1036zxU3OBhD/giphy.gif#center)


These are the symbols we defined ourselves...
```
[ 10 ] Name: subFunc                         Info: 0x18 (Bind: 0x1 | Type: 0x2)      Other: 0x0      Shndx: 0xe      Value: 0x0000000012c6   Size: 0x1a
[ 16 ] Name: addFunc                         Info: 0x18 (Bind: 0x1 | Type: 0x2)      Other: 0x0      Shndx: 0xe      Value: 0x0000000012ac   Size: 0x1a
[ 21 ] Name: divFunc                         Info: 0x18 (Bind: 0x1 | Type: 0x2)      Other: 0x0      Shndx: 0xe      Value: 0x0000000012fa   Size: 0x34
[ 25 ] Name: mulFunc                         Info: 0x18 (Bind: 0x1 | Type: 0x2)      Other: 0x0      Shndx: 0xe      Value: 0x0000000012e0   Size: 0x1a
[ 26 ] Name: main                            Info: 0x18 (Bind: 0x1 | Type: 0x2)      Other: 0x0      Shndx: 0xe      Value: 0x000000001169   Size: 0x143
```

We can observe the similarities in various members between `libarithmatic.o` and `main.o`. The notable difference I can identify is the `sh_shndx` value, which has changed but still points to the `.text` section of `calc` file. The important point is that it should reference the `.text` section, regardless of the section index value.

Another difference is in the `st_value`. With the addition of numerous new symbols in this file, the positions of these symbols have shifted. Initially, we had the `main` function in `main.o` and `addFunc` in `libarithmatic.o`, both at offset `0x0`. However, when combining them into a single file, one of them had to adjust its offset to make room for the other. This is precisely what occurred here, and there are also other symbols (of function type) that occupied the initial offsets, causing our defined functions to compromise on their offsets.

One more intriguing detail is the `_start` symbol, which has an offset of `0x000000001070`. This offset serves as the **entry point** of our ELF executable binary. You can verify this using `readelf` or any method you prefer. If you happen to overwrite the `entrypoint` value in `ELF file headers`, you'll be calling some other function instead of `_start` function of `glibc`. Since `_start` function performs some startup actions for C runtime environment, so the modified binary may or may not work as intended.


I'm sure that's enough for today, ta-ta!


![](https://media.giphy.com/media/mP8GermRyOFWV8PQeq/giphy.gif#center)
