---
title: "Elf Chronicles: Relocations (6/?)"
date: 2023-12-08T14:17:56+05:30
draft: true
# showtoc: false
tags:
    - "C"
    - "ELF"
    - "RE"
series:
    - "ELF Chronicles"
description: "Exploring general concept of ELF relocations"
# math: true
# ShowBreadCrumbs: false
# searchHidden: true
# hideMeta: true
---


In previous article about Symbol Tables, we talked about the below diagram ....


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


...and how the compiler was unaware of the final addresses for many symbols. When things get a bit confusing for the compiler, it takes the easy route by putting zeros in the addresses and creating relocation entries for the linker/loader to sort out.

The linker combines all the `.o` files, causing changes to the positions of different parts. For example, the `main` function in `main.o` and `addFunc` in `libarithmatic.o` both start off at position `0x0`. But when you link these files, this setup causes issues, so some tweaks are needed.

In this situation, the compiler and assembler team up to produce the `.o` file, but they don't know for sure where each part will end up in the eventual `calc` binary. So, they play it safe by leaving these spots empty and make notes in the relocations section. This tells the linker that these positions need some adjustments later on.

## Relocations

According to [ELF specification (version 1.2)](https://refspecs.linuxbase.org/elf/elf.pdf)

>> Relocation is the process of connecting symbolic references with symbolic definitions.



Relocation is a straightforward concept in coding. When you're compiling code, the compiler doesn't always know the exact addresses for everything in the program. ELF relocations become important when the addresses of symbols are uncertain during compilation, often because the final addresses are determined by the linker or loader at a later stage. It's similar to arranging pieces in a puzzle without having all the details upfront.


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

Now, there are three essential elements needed for relocation to take place:

1. The spot where the adjustment needs to be made.
2. The symbol that's part of the adjustment.
3. An algorithm specifying how to calculate and apply the necessary fix.

The compiler stores all this information in a special section identified by type - either `REL` (0x9) or `RELA` (0x4).

- `REL` is used for basic relocation entries.
- `RELA` is essentially the same as `REL`, but with an extra addend value. *This doesn't significantly impact the concept, though.*

You can easily identify these sections using `readelf`;...

```
❯ readelf --section-headers --wide main.o
There are 14 section headers, starting at offset 0x5a8:

Section Headers:
  [Nr] Name              Type            Address          Off    Size   ES Flg Lk Inf Al
  [ 0]                   NULL            0000000000000000 000000 000000 00      0   0  0
  [ 1] .text             PROGBITS        0000000000000000 000040 000143 00  AX  0   0  1
  [ 2] .rela.text        RELA            0000000000000000 0003e0 000138 18   I 11   1  8
  [ 3] .data             PROGBITS        0000000000000000 000183 000000 00  WA  0   0  1
  [ 4] .bss              NOBITS          0000000000000000 000183 000000 00  WA  0   0  1
  [ 5] .rodata           PROGBITS        0000000000000000 000183 000041 00   A  0   0  1
  [ 6] .comment          PROGBITS        0000000000000000 0001c4 00001c 01  MS  0   0  1
  [ 7] .note.GNU-stack   PROGBITS        0000000000000000 0001e0 000000 00      0   0  1
  [ 8] .note.gnu.property NOTE            0000000000000000 0001e0 000030 00   A  0   0  8
  [ 9] .eh_frame         PROGBITS        0000000000000000 000210 000038 00   A  0   0  8
  [10] .rela.eh_frame    RELA            0000000000000000 000518 000018 18   I 11   9  8
  [11] .symtab           SYMTAB          0000000000000000 000248 000138 18     12   4  8
  [12] .strtab           STRTAB          0000000000000000 000380 000059 00      0   0  1
  [13] .shstrtab         STRTAB          0000000000000000 000530 000074 00      0   0  1
Key to Flags:
  W (write), A (alloc), X (execute), M (merge), S (strings), I (info),
  L (link order), O (extra OS processing required), G (group), T (TLS),
  C (compressed), x (unknown), o (OS specific), E (exclude),
  D (mbind), l (large), p (processor specific)
```

My parser gives me the same results... (with different looks)

```
 [ 00 ] Section Name:                            Type: 0x0       Flags: 0x0      Addr: 0x0       Offset: 0x0             Size: 0         Link: 0         Info: 0x0       Addralign: 0x0          Entsize: 0
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 [ 01 ] Section Name: .text                      Type: 0x1       Flags: 0x6      Addr: 0x0       Offset: 0x40            Size: 323       Link: 0         Info: 0x0       Addralign: 0x1          Entsize: 0
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 [ 02 ] Section Name: .rela.text                 Type: 0x4       Flags: 0x40     Addr: 0x0       Offset: 0x3e0           Size: 312       Link: 11        Info: 0x1       Addralign: 0x8          Entsize: 24
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 [ 03 ] Section Name: .data                      Type: 0x1       Flags: 0x3      Addr: 0x0       Offset: 0x183           Size: 0         Link: 0         Info: 0x0       Addralign: 0x1          Entsize: 0
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 [ 04 ] Section Name: .bss                       Type: 0x8       Flags: 0x3      Addr: 0x0       Offset: 0x183           Size: 0         Link: 0         Info: 0x0       Addralign: 0x1          Entsize: 0
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 [ 05 ] Section Name: .rodata                    Type: 0x1       Flags: 0x2      Addr: 0x0       Offset: 0x183           Size: 65        Link: 0         Info: 0x0       Addralign: 0x1          Entsize: 0
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 [ 06 ] Section Name: .comment                   Type: 0x1       Flags: 0x30     Addr: 0x0       Offset: 0x1c4           Size: 28        Link: 0         Info: 0x0       Addralign: 0x1          Entsize: 1
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 [ 07 ] Section Name: .note.GNU-stack            Type: 0x1       Flags: 0x0      Addr: 0x0       Offset: 0x1e0           Size: 0         Link: 0         Info: 0x0       Addralign: 0x1          Entsize: 0
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 [ 08 ] Section Name: .note.gnu.property         Type: 0x7       Flags: 0x2      Addr: 0x0       Offset: 0x1e0           Size: 48        Link: 0         Info: 0x0       Addralign: 0x8          Entsize: 0
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 [ 09 ] Section Name: .eh_frame                  Type: 0x1       Flags: 0x2      Addr: 0x0       Offset: 0x210           Size: 56        Link: 0         Info: 0x0       Addralign: 0x8          Entsize: 0
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 [ 10 ] Section Name: .rela.eh_frame             Type: 0x4       Flags: 0x40     Addr: 0x0       Offset: 0x518           Size: 24        Link: 11        Info: 0x9       Addralign: 0x8          Entsize: 24
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 [ 11 ] Section Name: .symtab                    Type: 0x2       Flags: 0x0      Addr: 0x0       Offset: 0x248           Size: 312       Link: 12        Info: 0x4       Addralign: 0x8          Entsize: 24
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 [ 12 ] Section Name: .strtab                    Type: 0x3       Flags: 0x0      Addr: 0x0       Offset: 0x380           Size: 89        Link: 0         Info: 0x0       Addralign: 0x1          Entsize: 0
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 [ 13 ] Section Name: .shstrtab                  Type: 0x3       Flags: 0x0      Addr: 0x0       Offset: 0x530           Size: 116       Link: 0         Info: 0x0       Addralign: 0x1          Entsize: 0
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
```



In any case, identifying the relocation sections is straightforward -- REL (0x9) or RELA (0x4).

```
# From readelf
  [Nr] Name              Type            Address          Off    Size   ES Flg Lk Inf Al
  [ 2] .rela.text        RELA            0000000000000000 0003e0 000138 18   I 11   1  8
  [10] .rela.eh_frame    RELA            0000000000000000 000518 000018 18   I 11   9  8


# From my parser
 [ 02 ] Section Name: .rela.text                 Type: 0x4       Flags: 0x40     Addr: 0x0       Offset: 0x3e0           Size: 312       Link: 11        Info: 0x1       Addralign: 0x8          Entsize: 24
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 [ 10 ] Section Name: .rela.eh_frame             Type: 0x4       Flags: 0x40     Addr: 0x0       Offset: 0x518           Size: 24        Link: 11        Info: 0x9       Addralign: 0x8          Entsize: 24
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
```

(Note: To keep things clear in this article and to maintain simplicity, we're going to ignore the `.rela.eh_frame`. We can dive into that particular aspect another time.)

We can use the details shared earlier to pinpoint the real data of the relocation section. This means we'll be finding the section data by using the section header entry -- a process we've gone through multiple times before.

```

            [ 02 ] Section Name: .rela.text
                      Type: 0x4
                      Flags: 0x40
                      Addr: 0x0
            ┌─────────Offset: 0x3e0
            │      ┌──Size: 312
            │      │  Link: 11
            │      │  Info: 0x1
            │      │  Addralign: 0x8
            │      │  Entsize: 24
            │      │        │
            │      │        │
            │      │        │
            ▼      ▼        ▼
❯ xxd -s 0x3e0 -l 0x138 -c 0x18 main.o | nl -v0 -
     0  000003e0: 1a00 0000 0000 0000 0200 0000 0300 0000 fcff ffff ffff ffff  ........................
     1  000003f8: 2700 0000 0000 0000 0400 0000 0500 0000 fcff ffff ffff ffff  '.......................
     2  00000410: 3d00 0000 0000 0000 0200 0000 0300 0000 1500 0000 0000 0000  =.......................
     3  00000428: 4a00 0000 0000 0000 0400 0000 0600 0000 fcff ffff ffff ffff  J.......................
     4  00000440: 8700 0000 0000 0000 0400 0000 0700 0000 fcff ffff ffff ffff  ........................
     5  00000458: a400 0000 0000 0000 0400 0000 0800 0000 fcff ffff ffff ffff  ........................
     6  00000470: c100 0000 0000 0000 0400 0000 0900 0000 fcff ffff ffff ffff  ........................
     7  00000488: de00 0000 0000 0000 0400 0000 0a00 0000 fcff ffff ffff ffff  ........................
     8  000004a0: ee00 0000 0000 0000 0200 0000 0300 0000 1e00 0000 0000 0000  ........................
     9  000004b8: f600 0000 0000 0000 0400 0000 0b00 0000 fcff ffff ffff ffff  ........................
    10  000004d0: 1701 0000 0000 0000 0200 0000 0300 0000 2f00 0000 0000 0000  ................/.......
    11  000004e8: 2401 0000 0000 0000 0400 0000 0500 0000 fcff ffff ffff ffff  $.......................
    12  00000500: 3d01 0000 0000 0000 0400 0000 0c00 0000 fcff ffff ffff ffff  =.......................
```

Linux kernel has some specific structures to define `REL` and `RELA` entries....

```c
/* https://elixir.bootlin.com/linux/latest/source/include/uapi/linux/elf.h#L171 */
typedef struct elf64_rel {
  Elf64_Addr r_offset;	/* Location at which to apply the action */
  Elf64_Xword r_info;	/* index and type of relocation */
} Elf64_Rel;

/* https://elixir.bootlin.com/linux/latest/source/include/uapi/linux/elf.h#L182 */
typedef struct elf64_rela {
  Elf64_Addr r_offset;	/* Location at which to apply the action */
  Elf64_Xword r_info;	/* index and type of relocation */
  Elf64_Sxword r_addend;	/* Constant addend used to compute value */
} Elf64_Rela;

/* https://elixir.bootlin.com/linux/latest/source/include/uapi/linux/elf.h#L163 */
#define ELF64_R_SYM(i)			((i) >> 32)
#define ELF64_R_TYPE(i)			((i) & 0xffffffff)
```


After parsing this section's data, I got following results.

```
[ 02 ] Section Name: .rela.text        Type: 0x4       Flags: 0x40     Addr: 0x0       Offset: 0x3e0           Size: 312       Link: 11        Info: 0x1       Addralign: 0x8          Entsize: 24
     [  0 ] Offset: 0x1a         Info: 0x000300000002 (Sym: 0x3 | Type: 0x2)     Addend: -4
     [  1 ] Offset: 0x27         Info: 0x000500000004 (Sym: 0x5 | Type: 0x4)     Addend: -4
     [  2 ] Offset: 0x3d         Info: 0x000300000002 (Sym: 0x3 | Type: 0x2)     Addend: 21
     [  3 ] Offset: 0x4a         Info: 0x000600000004 (Sym: 0x6 | Type: 0x4)     Addend: -4
     [  4 ] Offset: 0x87         Info: 0x000700000004 (Sym: 0x7 | Type: 0x4)     Addend: -4
     [  5 ] Offset: 0xa4         Info: 0x000800000004 (Sym: 0x8 | Type: 0x4)     Addend: -4
     [  6 ] Offset: 0xc1         Info: 0x000900000004 (Sym: 0x9 | Type: 0x4)     Addend: -4
     [  7 ] Offset: 0xde         Info: 0x000a00000004 (Sym: 0xa | Type: 0x4)     Addend: -4
     [  8 ] Offset: 0xee         Info: 0x000300000002 (Sym: 0x3 | Type: 0x2)     Addend: 30
     [  9 ] Offset: 0xf6         Info: 0x000b00000004 (Sym: 0xb | Type: 0x4)     Addend: -4
     [ 10 ] Offset: 0x117        Info: 0x000300000002 (Sym: 0x3 | Type: 0x2)     Addend: 47
     [ 11 ] Offset: 0x124        Info: 0x000500000004 (Sym: 0x5 | Type: 0x4)     Addend: -4
     [ 12 ] Offset: 0x13d        Info: 0x000c00000004 (Sym: 0xc | Type: 0x4)     Addend: -4
```

Let's pause for a moment to grasp the significance of each member in this structure and how it aids the linker in the relocation process.

(👇 *Shamelessly stolen from [`man 5 elf`](https://www.man7.org/linux/man-pages/man5/elf.5.html)* 👇)

### r_offset
This member gives the location at which to apply the relocation.
- For a relocatable file, the value is the byte offset from the beginning of the section where relocation is to be applied.
- For an executable file or shared object, the value is the virtual address of the storage unit affected by the relocation.

### r_info
This member gives both the symbol table index with respect to which the relocation must be made and the type of relocation to apply. (Linux kernel provides a macro to filter these values out from it - `ELF64_R_SYM` and `ELF64_R_TYPE`)

### r_addend
This member specifies a constant addend used to compute the value to be stored into the relocatable field.

## Analysis
Now armed with the theoretical knowledge, let's delve into how the linker utilizes this information to perform a relocation. We'll begin by examining the details of the first relocation entry.

```
[  0 ] Offset: 0x1a         Info: 0x000300000002 (Sym: 0x3 | Type: 0x2)     Addend: -4
```


Now there are 2 things I want you to think about before we even start with the relocation process...

1. We know `r_offset` holds the offset from beginning of the section where relocation is to be applied. **Which section is that here??**
2. And `ELF64_R_SYM` from `r_info` stores the index in symbol table. But we can obviously have more than 1 symbol table, so **Which symbol table we are talking about here??**


Answer >> To identify that, you just have to check `sh_info` and `sh_link` members from the section header entry.



In our case, the associated symbol table and the section where relocations will apply can be viewed like this:-

```


             ┌────────────────────────────────────┐
             │  [ 11 ] Section Name: .symtab      │◄───────┐
             │          Type: 0x2                 │        │
             │          Flags: 0x0                │        │
             │          Addr: 0x0                 │        │
             │          Offset: 0x248             │        │
             │          Size: 312                 │        │
             │          Link: 12                  │        │
             │          Info: 0x4                 │        │
             │          Addralign: 0x8            │        │
             │          Entsize: 24               │        │
             │                                    │        │
             └────────────────────────────────────┘        │
                                                           │
                         /* all the symbols associated     │
                            are in section 11 */           │
                                                           │
                                                           │
             ┌────────────────────────────────────┐        │
             │  [ 02 ] Section Name: .rela.text   │        │
             │          Type: 0x4                 │        │
             │          Flags: 0x40               │        │
             │          Addr: 0x0                 │        │
             │          Offset: 0x3e0             │        │
             │          Size: 312                 │        │
             │          Link: 11 ─────────────────┼────────┘
     ┌───────┼───────── Info: 0x1                 │
     │       │          Addralign: 0x8            │
     │       │          Entsize: 24               │
     │       │                                    │
     │       └────────────────────────────────────┘
     │
     │
     │ /* relocations apply to section with index 1 */
     │
     │       ┌───────────────────────────────────┐
     └──────►│   [ 01 ] Section Name: .text      │
             │           Type: 0x1               │
             │           Flags: 0x6              │
             │           Addr: 0x0               │
             │           Offset: 0x40            │
             │           Size: 323               │
             │           Link: 0                 │
             │           Info: 0x0               │
             │           Addralign: 0x1          │
             │           Entsize: 0              │
             │                                   │
             └───────────────────────────────────┘
```

Traditionally, the chosen naming scheme for the relocation section indicates the section where relocations are intended to be applied. For example, if relocations are to be applied on `.text` section, then the relocation entries will be under `.rela.text` or `.rel.text`. However, it's crucial to note that this is merely a tradition and not a strict requirement.

*In the wisdom of ancient gods, it is advised not to depend solely on names.*

![](https://media.giphy.com/media/Y3ptrXcCeZZdj0yPZH/giphy.gif#center)

Now that you know where to apply relocation and where to look for symbols... let's look at the relocation entry we started with


```
[  0 ] Offset: 0x1a         Info: 0x000300000002 (Sym: 0x3 | Type: 0x2)     Addend: -4
```


We can now understand that:
- The relocation is to be applied at offset `0x1a` in `.text` section.
- The symbol associated with this relocation is 3rd symbol in `.symtab` section.


So, if you want to see the big picture,


```




              ┌────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
              │ [ 11 ] Section Name: .symtab                    Type: 0x2       Flags: 0x0      Addr: 0x0       Offset: 0x248           Size: 312       Link: 12        Info: 0x4       Addralign: 0x8 │
              │      [  0 ] Name:                                        Info: 0x00 (Bind: 0x0 | Type: 0x0)      Other: 0x0      Shndx: 0x0      Value: 0x000000000000   Size: 0x0                     │
              │      [  1 ] Name: main.c                                 Info: 0x04 (Bind: 0x0 | Type: 0x4)      Other: 0x0      Shndx: 0xfff1   Value: 0x000000000000   Size: 0x0                     │
              │      [  2 ] Name:                                        Info: 0x03 (Bind: 0x0 | Type: 0x3)      Other: 0x0      Shndx: 0x1      Value: 0x000000000000   Size: 0x0                     │
        ┌─────┼────► [  3 ] Name:                                        Info: 0x03 (Bind: 0x0 | Type: 0x3)      Other: 0x0      Shndx: 0x5      Value: 0x000000000000   Size: 0x0                     │
        │     │      [  4 ] Name: main                                   Info: 0x18 (Bind: 0x1 | Type: 0x2)      Other: 0x0      Shndx: 0x1      Value: 0x000000000000   Size: 0x143                   │
        │     │      [  5 ] Name: printf                                 Info: 0x16 (Bind: 0x1 | Type: 0x0)      Other: 0x0      Shndx: 0x0      Value: 0x000000000000   Size: 0x0                     │
        │     │      [  6 ] Name: __isoc99_scanf                         Info: 0x16 (Bind: 0x1 | Type: 0x0)      Other: 0x0      Shndx: 0x0      Value: 0x000000000000   Size: 0x0                     │
        │     │      [  7 ] Name: addFunc                                Info: 0x16 (Bind: 0x1 | Type: 0x0)      Other: 0x0      Shndx: 0x0      Value: 0x000000000000   Size: 0x0                     │
        │     │      [  8 ] Name: subFunc                                Info: 0x16 (Bind: 0x1 | Type: 0x0)      Other: 0x0      Shndx: 0x0      Value: 0x000000000000   Size: 0x0                     │
        │     │      [  9 ] Name: mulFunc                                Info: 0x16 (Bind: 0x1 | Type: 0x0)      Other: 0x0      Shndx: 0x0      Value: 0x000000000000   Size: 0x0                     │
        │     │      [ 10 ] Name: divFunc                                Info: 0x16 (Bind: 0x1 | Type: 0x0)      Other: 0x0      Shndx: 0x0      Value: 0x000000000000   Size: 0x0                     │
        │     │      [ 11 ] Name: puts                                   Info: 0x16 (Bind: 0x1 | Type: 0x0)      Other: 0x0      Shndx: 0x0      Value: 0x000000000000   Size: 0x0                     │
        │     │      [ 12 ] Name: __stack_chk_fail                       Info: 0x16 (Bind: 0x1 | Type: 0x0)      Other: 0x0      Shndx: 0x0      Value: 0x000000000000   Size: 0x0                     │
        │     │                                                                                                                                                                                        │
        │     └────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
        │
        └──────────────────────────────────────────────────────────┐
                                                                   │
                                                                   │
                                                                   │
                                                                   │
                                                                   │
          [  0 ] Offset: 0x1a         Info: 0x000300000002 (Sym: 0x3 | Type: 0x2)     Addend: -4
                           │
                           │
                           │
                           │
                           └─────────────────────────┐
                                                     │
                                                     │
                    ┌────────────────────────────────┼───────────────────────────────────────────────────┐
                    │                                │                                                   │
                    │   Disassembly of section .text:│                                                   │
                    │                                │                                                   │
                    │   0000000000000000 <main>:     │                                                   │
                    │      0:   55                   │  push   rbp                                       │
                    │      1:   48 89 e5             │  mov    rbp,rsp                                   │
                    │      4:   48 83 ec 20          │  sub    rsp,0x20                                  │
                    │      8:   64 48 8b 04 25 28 00 │  mov    rax,QWORD PTR fs:0x28                     │
                    │      f:   00 00             ┌──┘                                                   │
                    │     11:   48 89 45 f8       ▼     mov    QWORD PTR [rbp-0x8],rax                   │
                    │     15:   31 c0   ┌───────────┐   xor    eax,eax                                   │
                    │     17:   48 8d 05│00 00 00 00│   lea    rax,[rip+0x0]        # 1e <main+0x1e>     │
                    │     1e:   48 89 c7└───────────┘   mov    rdi,rax                                   │
                    │     21:   b8 00 00 00 00          mov    eax,0x0                                   │
                    │     26:   e8 00 00 00 00          call   2b <main+0x2b>                            │
                    │     2b:   48 8d 4d f0             lea    rcx,[rbp-0x10]                            │
                    │                                                                                    │
                    └────────────────────────────────────────────────────────────────────────────────────┘

```


I get that things might look like a mess, a real puzzle at first. But trust me, with a bit of experience, you'll start to figure it out.


Now that we've uncovered the secret behind the symbol and pinpointed where the relocation is going to happen, it's time to figure out the algorithm we're going to use for relocation. Just peek into the `Type` part of `r_info`. In our case, it's holding the number `0x2`.

With a quick look at the gcc source code, I can tell you that this algorithm is `R_X86_64_PC32`... And another brief look at the source code of [`mold` (a modern linker)](https://github.com/rui314/mold/) helps me fully comprehend the algorithm...

```c
/* https://sourceware.org/git/?p=glibc.git;a=blob;f=elf/elf.h;hb=2bd00179885928fd95fcabfafc50e7b5c6e660d2#l3579 */
#define R_X86_64_PC32           2       /* PC relative 32 bit signed */


/* https://github.com/rui314/mold/blob/main/elf/arch-x86-64.cc#L433C1-L436C13 */
case R_X86_64_PC32:
case R_X86_64_PLT32:
  write32s(S + A - P);
  break;
```


Alright, so the magical spell for this relocation is `S + A - P`.... Now, let's break it down

```
S = value of symbol
A = Addend (r_addend)
P = place of relocation (r_offset is used to calculate this)
```

The addend (`A`) is simply `-4`. Why `-4`?? I don't know 🤷 `¯\_(ツ)_/¯` (If you happen to know it, please share your wisdom with this stuppid child... pretty please! 🥺👉👈)

And the place of relocation (`P`) should be `0x1a`... right? **WRONG!!**... the place of relocation will be location of `.text` section + `0x1a`. Linker will know where `.text` will be after the merge process, so it'll be easy for the linker to get the exact location.

Finally, value of symbol (`S`), this one is a bit tricky here... You need to take a look at the symbol table for this

```
[ 11 ] Section Name: .symtab                    Type: 0x2       Flags: 0x0      Addr: 0x0       Offset: 0x248           Size: 312       Link: 12        Info: 0x4       Addralign: 0x8          Entsize: 24
    [  0 ] Name:                                        Info: 0x00 (Bind: 0x0 | Type: 0x0)      Other: 0x0      Shndx: 0x0      Value: 0x000000000000   Size: 0x0
    [  1 ] Name: main.c                                 Info: 0x04 (Bind: 0x0 | Type: 0x4)      Other: 0x0      Shndx: 0xfff1   Value: 0x000000000000   Size: 0x0
    [  2 ] Name:                                        Info: 0x03 (Bind: 0x0 | Type: 0x3)      Other: 0x0      Shndx: 0x1      Value: 0x000000000000   Size: 0x0
    [  3 ] Name:                                        Info: 0x03 (Bind: 0x0 | Type: 0x3)      Other: 0x0      Shndx: 0x5      Value: 0x000000000000   Size: 0x0
    [  4 ] Name: main                                   Info: 0x18 (Bind: 0x1 | Type: 0x2)      Other: 0x0      Shndx: 0x1      Value: 0x000000000000   Size: 0x143
    [  5 ] Name: printf                                 Info: 0x16 (Bind: 0x1 | Type: 0x0)      Other: 0x0      Shndx: 0x0      Value: 0x000000000000   Size: 0x0
    [  6 ] Name: __isoc99_scanf                         Info: 0x16 (Bind: 0x1 | Type: 0x0)      Other: 0x0      Shndx: 0x0      Value: 0x000000000000   Size: 0x0
    [  7 ] Name: addFunc                                Info: 0x16 (Bind: 0x1 | Type: 0x0)      Other: 0x0      Shndx: 0x0      Value: 0x000000000000   Size: 0x0
    [  8 ] Name: subFunc                                Info: 0x16 (Bind: 0x1 | Type: 0x0)      Other: 0x0      Shndx: 0x0      Value: 0x000000000000   Size: 0x0
    [  9 ] Name: mulFunc                                Info: 0x16 (Bind: 0x1 | Type: 0x0)      Other: 0x0      Shndx: 0x0      Value: 0x000000000000   Size: 0x0
    [ 10 ] Name: divFunc                                Info: 0x16 (Bind: 0x1 | Type: 0x0)      Other: 0x0      Shndx: 0x0      Value: 0x000000000000   Size: 0x0
    [ 11 ] Name: puts                                   Info: 0x16 (Bind: 0x1 | Type: 0x0)      Other: 0x0      Shndx: 0x0      Value: 0x000000000000   Size: 0x0
    [ 12 ] Name: __stack_chk_fail                       Info: 0x16 (Bind: 0x1 | Type: 0x0)      Other: 0x0      Shndx: 0x0      Value: 0x000000000000   Size: 0x0
```

Just focus on the symbol for our case... index 3.


```c
[  3 ] Name:                                 /* No name for symbol */
        Info: 0x03 (Bind: 0x0 | Type: 0x3)   /* Bind: STB_LOCAL | Type: STT_SECTION */
        Other: 0x0                           /* default visibility */
        Shndx: 0x5                           /* section 5; .rodata in our case */
        Value: 0x000000000000                /* no value; I wonder what could be the value for "STT_SECTION" type symbol */
        Size: 0x0                            /* Unknown size */
```


Putting all the pieces together, it's crystal clear now that this symbol is casually pointing towards `.rodata` section. Picture this section as a treasure trove of read-only data. An example? Think of it like a collection of strings that `printf` and its buddies use to sprinkle some magic onto your screen. It's like the VIP lounge for data that's there to be seen but not messed with.


looking back at our relocation entry, we can now understand it better


```
[  0 ] Offset: 0x1a         Info: 0x000300000002 (Sym: 0x3 | Type: 0x2)     Addend: -4
```

- Apply relocation at offset `0x1a` in `.text` section.
- Use symbol `3` for relocation... that points to `.rodata` section.
- Relocation algorithm will be `R_X86_64_PC32` (`S + A - P`)


There are many tools like `readelf` and `objdump`, that can show you relocation entries with all these things simplified.

```
❯ readelf --relocs  --wide calc/main.o

Relocation section '.rela.text' at offset 0x3e0 contains 13 entries:
    Offset             Info             Type               Symbol's Value  Symbol's Name + Addend
000000000000001a  0000000300000002 R_X86_64_PC32          0000000000000000 .rodata - 4
0000000000000027  0000000500000004 R_X86_64_PLT32         0000000000000000 printf - 4
000000000000003d  0000000300000002 R_X86_64_PC32          0000000000000000 .rodata + 15
000000000000004a  0000000600000004 R_X86_64_PLT32         0000000000000000 __isoc99_scanf - 4
0000000000000087  0000000700000004 R_X86_64_PLT32         0000000000000000 addFunc - 4
00000000000000a4  0000000800000004 R_X86_64_PLT32         0000000000000000 subFunc - 4
00000000000000c1  0000000900000004 R_X86_64_PLT32         0000000000000000 mulFunc - 4
00000000000000de  0000000a00000004 R_X86_64_PLT32         0000000000000000 divFunc - 4
00000000000000ee  0000000300000002 R_X86_64_PC32          0000000000000000 .rodata + 1e
00000000000000f6  0000000b00000004 R_X86_64_PLT32         0000000000000000 puts - 4
0000000000000117  0000000300000002 R_X86_64_PC32          0000000000000000 .rodata + 2f
0000000000000124  0000000500000004 R_X86_64_PLT32         0000000000000000 printf - 4
000000000000013d  0000000c00000004 R_X86_64_PLT32         0000000000000000 __stack_chk_fail - 4
```

```
❯ objdump -M intel -dr calc/main.o

calc/main.o:     file format elf64-x86-64


Disassembly of section .text:

0000000000000000 <main>:
   0:   55                      push   rbp
   1:   48 89 e5                mov    rbp,rsp
   4:   48 83 ec 20             sub    rsp,0x20
   8:   64 48 8b 04 25 28 00    mov    rax,QWORD PTR fs:0x28
   f:   00 00
  11:   48 89 45 f8             mov    QWORD PTR [rbp-0x8],rax
  15:   31 c0                   xor    eax,eax
  17:   48 8d 05 00 00 00 00    lea    rax,[rip+0x0]        # 1e <main+0x1e>
                        1a: R_X86_64_PC32       .rodata-0x4
  1e:   48 89 c7                mov    rdi,rax
  21:   b8 00 00 00 00          mov    eax,0x0
  26:   e8 00 00 00 00          call   2b <main+0x2b>
                        27: R_X86_64_PLT32      printf-0x4
  2b:   48 8d 4d f0             lea    rcx,[rbp-0x10]
  2f:   48 8d 55 eb             lea    rdx,[rbp-0x15]
  33:   48 8d 45 ec             lea    rax,[rbp-0x14]
  37:   48 89 c6                mov    rsi,rax
  3a:   48 8d 05 00 00 00 00    lea    rax,[rip+0x0]        # 41 <main+0x41>
                        3d: R_X86_64_PC32       .rodata+0x15
  41:   48 89 c7                mov    rdi,rax
  44:   b8 00 00 00 00          mov    eax,0x0
  49:   e8 00 00 00 00          call   4e <main+0x4e>
                        4a: R_X86_64_PLT32      __isoc99_scanf-0x4
  4e:   0f b6 45 eb             movzx  eax,BYTE PTR [rbp-0x15]
  52:   0f be c0                movsx  eax,al
  55:   83 f8 2f                cmp    eax,0x2f
  58:   74 74                   je     ce <main+0xce>
  5a:   83 f8 2f                cmp    eax,0x2f
  5d:   0f 8f 88 00 00 00       jg     eb <main+0xeb>
  63:   83 f8 2d                cmp    eax,0x2d
  66:   74 2c                   je     94 <main+0x94>
  68:   83 f8 2d                cmp    eax,0x2d
  6b:   7f 7e                   jg     eb <main+0xeb>
  6d:   83 f8 2a                cmp    eax,0x2a
  70:   74 3f                   je     b1 <main+0xb1>
  72:   83 f8 2b                cmp    eax,0x2b
  75:   75 74                   jne    eb <main+0xeb>
  77:   f3 0f 10 45 f0          movss  xmm0,DWORD PTR [rbp-0x10]
  7c:   8b 45 ec                mov    eax,DWORD PTR [rbp-0x14]
  7f:   0f 28 c8                movaps xmm1,xmm0
  82:   66 0f 6e c0             movd   xmm0,eax
  86:   e8 00 00 00 00          call   8b <main+0x8b>
                        87: R_X86_64_PLT32      addFunc-0x4
  8b:   66 0f 7e c0             movd   eax,xmm0
  8f:   89 45 f4                mov    DWORD PTR [rbp-0xc],eax
  92:   eb 6d                   jmp    101 <main+0x101>
  94:   f3 0f 10 45 f0          movss  xmm0,DWORD PTR [rbp-0x10]
  99:   8b 45 ec                mov    eax,DWORD PTR [rbp-0x14]
  9c:   0f 28 c8                movaps xmm1,xmm0
  9f:   66 0f 6e c0             movd   xmm0,eax
  a3:   e8 00 00 00 00          call   a8 <main+0xa8>
                        a4: R_X86_64_PLT32      subFunc-0x4
  a8:   66 0f 7e c0             movd   eax,xmm0
  ac:   89 45 f4                mov    DWORD PTR [rbp-0xc],eax
  af:   eb 50                   jmp    101 <main+0x101>
  b1:   f3 0f 10 45 f0          movss  xmm0,DWORD PTR [rbp-0x10]
  b6:   8b 45 ec                mov    eax,DWORD PTR [rbp-0x14]
  b9:   0f 28 c8                movaps xmm1,xmm0
  bc:   66 0f 6e c0             movd   xmm0,eax
  c0:   e8 00 00 00 00          call   c5 <main+0xc5>
                        c1: R_X86_64_PLT32      mulFunc-0x4
  c5:   66 0f 7e c0             movd   eax,xmm0
  c9:   89 45 f4                mov    DWORD PTR [rbp-0xc],eax
  cc:   eb 33                   jmp    101 <main+0x101>
  ce:   f3 0f 10 45 f0          movss  xmm0,DWORD PTR [rbp-0x10]
  d3:   8b 45 ec                mov    eax,DWORD PTR [rbp-0x14]
  d6:   0f 28 c8                movaps xmm1,xmm0
  d9:   66 0f 6e c0             movd   xmm0,eax
  dd:   e8 00 00 00 00          call   e2 <main+0xe2>
                        de: R_X86_64_PLT32      divFunc-0x4
  e2:   66 0f 7e c0             movd   eax,xmm0
  e6:   89 45 f4                mov    DWORD PTR [rbp-0xc],eax
  e9:   eb 16                   jmp    101 <main+0x101>
  eb:   48 8d 05 00 00 00 00    lea    rax,[rip+0x0]        # f2 <main+0xf2>
                        ee: R_X86_64_PC32       .rodata+0x1e
  f2:   48 89 c7                mov    rdi,rax
  f5:   e8 00 00 00 00          call   fa <main+0xfa>
                        f6: R_X86_64_PLT32      puts-0x4
  fa:   b8 01 00 00 00          mov    eax,0x1
  ff:   eb 2c                   jmp    12d <main+0x12d>
 101:   66 0f ef d2             pxor   xmm2,xmm2
 105:   f3 0f 5a 55 f4          cvtss2sd xmm2,DWORD PTR [rbp-0xc]
 10a:   66 48 0f 7e d0          movq   rax,xmm2
 10f:   66 48 0f 6e c0          movq   xmm0,rax
 114:   48 8d 05 00 00 00 00    lea    rax,[rip+0x0]        # 11b <main+0x11b>
                        117: R_X86_64_PC32      .rodata+0x2f
 11b:   48 89 c7                mov    rdi,rax
 11e:   b8 01 00 00 00          mov    eax,0x1
 123:   e8 00 00 00 00          call   128 <main+0x128>
                        124: R_X86_64_PLT32     printf-0x4
 128:   b8 00 00 00 00          mov    eax,0x0
 12d:   48 8b 55 f8             mov    rdx,QWORD PTR [rbp-0x8]
 131:   64 48 2b 14 25 28 00    sub    rdx,QWORD PTR fs:0x28
 138:   00 00
 13a:   74 05                   je     141 <main+0x141>
 13c:   e8 00 00 00 00          call   141 <main+0x141>
                        13d: R_X86_64_PLT32     __stack_chk_fail-0x4
 141:   c9                      leave
 142:   c3                      ret
```



## Conclustion

Throughout this article, we explored the significance of relocations in ELF binaries, examining how compilers, assemblers, and linkers collaborate to produce executable files. We delved into the role of relocation sections, uncovering their purpose in accommodating changes to addresses and offsets during both compile-time and link-time.

Since symbols and relocations combined are a huge topic in itself, I'm adding few links that I think are interesting and will help to better grasp the whole concept in practicality

- @xianeizhang's notes (https://people.cs.pitt.edu/~xianeizhang/notes/Linking.html)

- Understanding the ELF specimen (https://hub.packtpub.com/understanding-elf-specimen/)

- Cloudflare blogs on "How to execute an object file" - [part 1](https://blog.cloudflare.com/how-to-execute-an-object-file-part-1/) and [part 2](https://blog.cloudflare.com/how-to-execute-an-object-file-part-2/)

- An amazing talk by [@Anders Schau Knatten](https://no.linkedin.com/in/anders-schau-knatten-34170619) on "[How symbols work and why we need them](https://www.youtube.com/watch?v=iBQo962Sx0g)" (youtube)

See you next time.

![](https://media.giphy.com/media/xT1R9F8M2RGQtovtni/giphy.gif#center)
