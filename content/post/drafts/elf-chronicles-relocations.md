---
title: "Elf Chronicles: Relocations"
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

In previous article about Symbol Tables, we talked about the below diagram


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

And that raised a question, **How did `main.o` call these functions when the address of these functions is not known to the compiler??** - *short answer* -- relocations.

(Note:- If you have not read the last article about symbol tables, I strongly suggest to give it a read before reading this one)


## Relocations

According to [ELF specification (version 1.2)](https://refspecs.linuxbase.org/elf/elf.pdf)

>> Relocation is the process of connecting symbolic references with symbolic definitions.


The idea of relocation is pretty simple: write programs separately and then link them together. In last article we saw the below snippet that shows us the before v/s after for the `main` function.


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


So what happens here is that the `compiler+assembler` creates the `.o` file but are not sure of where these symbol definitions will be located in the final `calc` binary. So, it leaves the entries as zeroes and makes an entry in `relocations` section to inform the linker that these locations needs some fixups.

Now here are 3 things that is required for a relocation to happen:

- location where this fix is to be made
- The symbol that is involved in the fix
- An algorithm that is to be used to calculate and apply the fixup.


In ELF files, relocations are recorded in a separate section with type - `REL` (0x9) or `RELA` (0x4).


You can easily identify these sections using `readelf`

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



Either ways, you can easily identify the relocation sections from here.

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

(Note:- I'm totally ignoring the `.rela.eh_frame` for the sanity of this article and our brains, we shall see that some other time)

We can use information from above to find the actual data of relocation section


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


That gives us 12 relocation entries, if we process them usign the below structure, we'll get some decent results


```c
/* https://elixir.bootlin.com/linux/latest/source/include/uapi/linux/elf.h#L182 */

typedef struct elf64_rela {
  Elf64_Addr r_offset;	/* Location at which to apply the action */
  Elf64_Xword r_info;	/* index and type of relocation */
  Elf64_Sxword r_addend;	/* Constant addend used to compute value */
} Elf64_Rela;
```


After parsing this section's data, I got below results.

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

Let's take a moment to understand what each member of this structure is and how it will help the linker in relocation process.

(👇 *Shamelessly stolen from [`man 5 elf`](https://www.man7.org/linux/man-pages/man5/elf.5.html)* 👇)
### r_offset
This member gives the location at which to apply the relocation.
- For a relocatable file, the value is the byte offset from the beginning of the section where relocation is to be applied.
- For an executable file or shared object, the value is the virtual address of the storage unit affected by the relocation.

### r_info
This member gives both the symbol table index with respect to which the relocation must be made and the type of relocation to apply.

### r_addend
This member specifies a constant addend used to compute the value to be stored into the relocatable field.

## Analysis
Now with the theoretical knowledge, let's try to understand how linker uses these values to perform a relocation. Let's start with the first relocation entry


```
[  0 ] Offset: 0x1a         Info: 0x000300000002 (Sym: 0x3 | Type: 0x2)     Addend: -4
```

It's `r_offset` value is `0x1a`. As the definition says, this is the beginning of the section where relocation is to be applied. But which section these relocations apply to??

Answer >> To identify that, you just have to check `sh_info` member from the section header entry. For relocation type section headers entry, this member points to the section to which the relocations apply.


```
          ┌────────────────────────────────────┐
          │  [ 02 ] Section Name: .rela.text   │
          │          Type: 0x4                 │
          │          Flags: 0x40               │
          │          Addr: 0x0                 │
          │          Offset: 0x3e0             │
          │          Size: 312                 │
          │          Link: 11                  │
  ┌───────┼───────── Info: 0x1                 │
  │       │          Addralign: 0x8            │
  │       │          Entsize: 24               │
  │       │                                    │
  │       └────────────────────────────────────┘
  │
  │
  │
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


Next thing is the `r_info` member, which is a combination of 2 things - Associated `Symbol` and its `Type`.


In linux kernel, this is defined as
```c
/* `i` is `r_info` */
#define ELF64_R_SYM(i)			((i) >> 32)
#define ELF64_R_TYPE(i)			((i) & 0xffffffff)
```


But wait, there can be more than one symbol table in the ELF file... which symbol table we are supposed to look at for the associated symbol?


Answer >> Look again at the `sh_link` member of the relocation. This member points to the associated symbol table.



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

By tradition, the naming scheme chosen for the relocation section is such that it indicates the section where relocations are to be applied. But this is just a tradition and not a strict requirement... Never rely on that information!!



Now that you know where to apply relocation and where to look for symbols... let's look at the relocation entry we started with


```
[  0 ] Offset: 0x1a         Info: 0x000300000002 (Sym: 0x3 | Type: 0x2)     Addend: -4
```


We can now understand that:
- The relocation is to be applied at offset `0x1a` in `.text` section.
- The symbol associated with this relocation is 3rd symbol in `.symtab` section.


So if you want to look at the whole picture


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
   └────────────────────────────────────────────────────────────────────────────────────┐
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
                                                │
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


I know things are looking a bit complex and tangled... and they are actually, atleast at first, but you'll get a hang of it with some practice.


Now we know the symbol and the location where relocation will take place... it's time to identify what algorithm we are going to use. That can be picked up from the `Type` part of `r_info`. For our case, that's `0x2`.

A quick look at gcc source code, I can say that this algorithm is `R_X86_64_PC32`... And another quick look at the source code of `mold` (modern linker -- another linker tool because it seemed easier to read)


```
https://github.com/rui314/mold/blob/main/elf/arch-x86-64.cc#L433C1-L436C13

    case R_X86_64_PC32:
    case R_X86_64_PLT32:
      write32s(S + A - P);
      break;
```


So the algorithm that will be used for the relocation is `S + A - P`.... let's break this down and understand each of these variables..

```
S = value of symbol
A = Addend
P = place of relocation
```

The linker will combine all `.o` files together and then perform the relocations, so at the time of relocation, the `.rodata` will likely be placed somewhere other than the current position. But the idea is whereever the `.rodata` section is located, pick up the offset and use that as `S` value in algorithm.

Then the addend is simply `-4`. Why `-4`?? I don't know; If you happen to know it, please share your wisdom with this stupid child... pretty please!

Now the place of relocation is `0x1a`... Super simple, right! This is one of the easiest relocation algorithms, where the position is calculated in relation to program counter (PC), hence the name `......._PC32`.

This can all be simplified if you just use the old `readelf` program.

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
