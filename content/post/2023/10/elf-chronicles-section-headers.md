---
title: "ELF Chronicles: Section Headers"
date: 2023-10-19T00:24:06+05:30
draft: false
# showtoc: false
tags: ["C", "ELF", "RE"]
series:
  - "ELF Chronicles"
description: "Exploring ELF Section Headers"
# math: true
# ShowBreadCrumbs: false
# searchHidden: true
# hideMeta: true
---


## Intro

Assuming you've got ELF headers like `Elf64_Ehdr` or `Elf32_Ehdr` at your fingertips, and you're armed with the know-how and tools to decipher their contents effortlessly.

For this article I'll be using the below C code to generate the ELF file.

```c
/* file: hello_world.c */

#include <stdio.h>

// A macro
#define HELLO_MSG1 "Hello World1"

// A global variable
char HELLO_MSG2[] = "Hello World2";


// main function
int main() {
    // local variable for main
    char HELLO_MSG3[] = "Hello World3";
    // Print messages
    printf("%s\n", HELLO_MSG1);
    printf("%s\n", HELLO_MSG2);
    printf("%s\n", HELLO_MSG3);
    return 0;
}
```

You can get the ELF binary by compiling this code.

```{linenos=false}
gcc hello_world.c -o hello_world
```

Now the task at hand is to read/parse the file and get information regarding the sections (`e_shoff`, `e_shentsize`, `e_shnum`, and `e_shstrndx`). Mostly I, another *mere* human, rely on a "industry grade" tool called `readelf` to read an ELF file and figure out stuff.


Now, the challenge on our hands is to crack open the file and unearth some juicy details about the sections. You already know, things like `e_shoff`, `e_shentsize`, `e_shnum`, and `e_shstrndx`. I confess, like any other *mere* human, I often lean on a trusty "industry-grade" tool called `readelf` to do the heavy lifting when it comes to ELF file forensics. (But it's always good to know the manual methods for those 1% kind of situations)

```{linenos=false}
❯ readelf --file-header --wide hello_world
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
  Entry point address:               0x1050
  Start of program headers:          64 (bytes into file)
  Start of section headers:          13608 (bytes into file)
  Flags:                             0x0
  Size of this header:               64 (bytes)
  Size of program headers:           56 (bytes)
  Number of program headers:         13
  Size of section headers:           64 (bytes)
  Number of section headers:         30
  Section header string table index: 29
```

Examining the output above, we can deduce a few key details.
- Firstly, the section headers take their grand entrance at a distance of `13608` bytes (`0x3528` in the mystical language of hex).
- Each of these header entries is precisely `64` bytes in size (`0x40` in hex),
- and in total, we've got a flourishing population of `30` sections (`1e` in hex).


So, it's like having a treasure map telling us exactly where to dig in the ELF file and how big the treasure chests are!



```{linenos=false}
❯ xxd -s 13608 -l $(( 64*30 )) -c 64 hello_world
00003528: 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  ................................................................
00003568: 1b00 0000 0100 0000 0200 0000 0000 0000 1803 0000 0000 0000 1803 0000 0000 0000 1c00 0000 0000 0000 0000 0000 0000 0000 0100 0000 0000 0000 0000 0000 0000 0000  ................................................................
000035a8: 2300 0000 0700 0000 0200 0000 0000 0000 3803 0000 0000 0000 3803 0000 0000 0000 4000 0000 0000 0000 0000 0000 0000 0000 0800 0000 0000 0000 0000 0000 0000 0000  #...............8.......8.......@...............................
000035e8: 3600 0000 0700 0000 0200 0000 0000 0000 7803 0000 0000 0000 7803 0000 0000 0000 2400 0000 0000 0000 0000 0000 0000 0000 0400 0000 0000 0000 0000 0000 0000 0000  6...............x.......x.......$...............................
00003628: 4900 0000 0700 0000 0200 0000 0000 0000 9c03 0000 0000 0000 9c03 0000 0000 0000 2000 0000 0000 0000 0000 0000 0000 0000 0400 0000 0000 0000 0000 0000 0000 0000  I............................... ...............................
00003668: 5700 0000 f6ff ff6f 0200 0000 0000 0000 c003 0000 0000 0000 c003 0000 0000 0000 1c00 0000 0000 0000 0600 0000 0000 0000 0800 0000 0000 0000 0000 0000 0000 0000  W......o........................................................
000036a8: 6100 0000 0b00 0000 0200 0000 0000 0000 e003 0000 0000 0000 e003 0000 0000 0000 c000 0000 0000 0000 0700 0000 0100 0000 0800 0000 0000 0000 1800 0000 0000 0000  a...............................................................
000036e8: 6900 0000 0300 0000 0200 0000 0000 0000 a004 0000 0000 0000 a004 0000 0000 0000 a800 0000 0000 0000 0000 0000 0000 0000 0100 0000 0000 0000 0000 0000 0000 0000  i...............................................................
00003728: 7100 0000 ffff ff6f 0200 0000 0000 0000 4805 0000 0000 0000 4805 0000 0000 0000 1000 0000 0000 0000 0600 0000 0000 0000 0200 0000 0000 0000 0200 0000 0000 0000  q......o........H.......H.......................................
00003768: 7e00 0000 feff ff6f 0200 0000 0000 0000 5805 0000 0000 0000 5805 0000 0000 0000 4000 0000 0000 0000 0700 0000 0100 0000 0800 0000 0000 0000 0000 0000 0000 0000  ~......o........X.......X.......@...............................
000037a8: 8d00 0000 0400 0000 0200 0000 0000 0000 9805 0000 0000 0000 9805 0000 0000 0000 c000 0000 0000 0000 0600 0000 0000 0000 0800 0000 0000 0000 1800 0000 0000 0000  ................................................................
000037e8: 9700 0000 0400 0000 4200 0000 0000 0000 5806 0000 0000 0000 5806 0000 0000 0000 3000 0000 0000 0000 0600 0000 1700 0000 0800 0000 0000 0000 1800 0000 0000 0000  ........B.......X.......X.......0...............................
00003828: a100 0000 0100 0000 0600 0000 0000 0000 0010 0000 0000 0000 0010 0000 0000 0000 1b00 0000 0000 0000 0000 0000 0000 0000 0400 0000 0000 0000 0000 0000 0000 0000  ................................................................
00003868: 9c00 0000 0100 0000 0600 0000 0000 0000 2010 0000 0000 0000 2010 0000 0000 0000 3000 0000 0000 0000 0000 0000 0000 0000 1000 0000 0000 0000 1000 0000 0000 0000  ................ ....... .......0...............................
000038a8: a700 0000 0100 0000 0600 0000 0000 0000 5010 0000 0000 0000 5010 0000 0000 0000 7101 0000 0000 0000 0000 0000 0000 0000 1000 0000 0000 0000 0000 0000 0000 0000  ................P.......P.......q...............................
000038e8: ad00 0000 0100 0000 0600 0000 0000 0000 c411 0000 0000 0000 c411 0000 0000 0000 0d00 0000 0000 0000 0000 0000 0000 0000 0400 0000 0000 0000 0000 0000 0000 0000  ................................................................
00003928: b300 0000 0100 0000 0200 0000 0000 0000 0020 0000 0000 0000 0020 0000 0000 0000 1100 0000 0000 0000 0000 0000 0000 0000 0400 0000 0000 0000 0000 0000 0000 0000  ................. ....... ......................................
00003968: bb00 0000 0100 0000 0200 0000 0000 0000 1420 0000 0000 0000 1420 0000 0000 0000 2400 0000 0000 0000 0000 0000 0000 0000 0400 0000 0000 0000 0000 0000 0000 0000  ................. ....... ......$...............................
000039a8: c900 0000 0100 0000 0200 0000 0000 0000 3820 0000 0000 0000 3820 0000 0000 0000 7c00 0000 0000 0000 0000 0000 0000 0000 0800 0000 0000 0000 0000 0000 0000 0000  ................8 ......8 ......|...............................
000039e8: d300 0000 0e00 0000 0300 0000 0000 0000 d03d 0000 0000 0000 d02d 0000 0000 0000 0800 0000 0000 0000 0000 0000 0000 0000 0800 0000 0000 0000 0800 0000 0000 0000  .................=.......-......................................
00003a28: df00 0000 0f00 0000 0300 0000 0000 0000 d83d 0000 0000 0000 d82d 0000 0000 0000 0800 0000 0000 0000 0000 0000 0000 0000 0800 0000 0000 0000 0800 0000 0000 0000  .................=.......-......................................
00003a68: eb00 0000 0600 0000 0300 0000 0000 0000 e03d 0000 0000 0000 e02d 0000 0000 0000 e001 0000 0000 0000 0700 0000 0000 0000 0800 0000 0000 0000 1000 0000 0000 0000  .................=.......-......................................
00003aa8: f400 0000 0100 0000 0300 0000 0000 0000 c03f 0000 0000 0000 c02f 0000 0000 0000 2800 0000 0000 0000 0000 0000 0000 0000 0800 0000 0000 0000 0800 0000 0000 0000  .................?......./......(...............................
00003ae8: f900 0000 0100 0000 0300 0000 0000 0000 e83f 0000 0000 0000 e82f 0000 0000 0000 2800 0000 0000 0000 0000 0000 0000 0000 0800 0000 0000 0000 0800 0000 0000 0000  .................?......./......(...............................
00003b28: 0201 0000 0100 0000 0300 0000 0000 0000 1040 0000 0000 0000 1030 0000 0000 0000 1d00 0000 0000 0000 0000 0000 0000 0000 0800 0000 0000 0000 0000 0000 0000 0000  .................@.......0......................................
00003b68: 0801 0000 0800 0000 0300 0000 0000 0000 2d40 0000 0000 0000 2d30 0000 0000 0000 0300 0000 0000 0000 0000 0000 0000 0000 0100 0000 0000 0000 0000 0000 0000 0000  ................-@......-0......................................
00003ba8: 0d01 0000 0100 0000 3000 0000 0000 0000 0000 0000 0000 0000 2d30 0000 0000 0000 1b00 0000 0000 0000 0000 0000 0000 0000 0100 0000 0000 0000 0100 0000 0000 0000  ........0...............-0......................................
00003be8: 0100 0000 0200 0000 0000 0000 0000 0000 0000 0000 0000 0000 4830 0000 0000 0000 7002 0000 0000 0000 1c00 0000 0600 0000 0800 0000 0000 0000 1800 0000 0000 0000  ........................H0......p...............................
00003c28: 0900 0000 0300 0000 0000 0000 0000 0000 0000 0000 0000 0000 b832 0000 0000 0000 5301 0000 0000 0000 0000 0000 0000 0000 0100 0000 0000 0000 0000 0000 0000 0000  .........................2......S...............................
00003c68: 1100 0000 0300 0000 0000 0000 0000 0000 0000 0000 0000 0000 0b34 0000 0000 0000 1601 0000 0000 0000 0000 0000 0000 0000 0100 0000 0000 0000 0000 0000 0000 0000  .........................4......................................
```



With each line presenting itself as a section header entry, it's like experiencing an elegant and straightforward design! Now we just have to map each of these lines to `Elf64_Shdr` (since we have a 64Bit file)

```{linenos=false}
/*
https://elixir.bootlin.com/linux/latest/source/include/uapi/linux/elf.h#L321
*/

typedef struct elf64_shdr {
  Elf64_Word sh_name;		/* Section name, index in string tbl    # 4 bytes */
  Elf64_Word sh_type;		/* Type of section                      # 4 bytes */
  Elf64_Xword sh_flags;		/* Miscellaneous section attributes   # 8 bytes */
  Elf64_Addr sh_addr;		/* Section virtual addr at execution    # 8 bytes */
  Elf64_Off sh_offset;		/* Section file offset                # 8 bytes */
  Elf64_Xword sh_size;		/* Size of section in bytes           # 8 bytes */
  Elf64_Word sh_link;		/* Index of another section             # 4 bytes */
  Elf64_Word sh_info;		/* Additional section information       # 4 bytes */
  Elf64_Xword sh_addralign;	/* Section alignment                # 8 bytes */
  Elf64_Xword sh_entsize;	/* Entry size if section holds table  # 8 bytes */
} Elf64_Shdr;
```

But first, understand why we are doing any of this...

## Section headers
Imagine a [LEGO batmobile](https://www.lego.com/en-us/product/lego-dc-batman-batmobile-tumbler-76240) – it's not just one big block, right? It has different parts, like a roof, doors, wheels, etc. **ELF sections** (not section headers) are like these parts in a computer program. Each section has its own job, some sections hold the variables, some hold the program instructions, while some just hold extra notes. Basically, each section has some data in it and has a specific role for that data.

Section headers is like a index for those sections. It tells you a good amount of details about the section, like
- Name of the section (indirectly :P),
- Type of section,
- Offset of the address in file and memory,
- Size of the section in bytes, etc

Now you know what section headers are and the valuable data they contain, and with the ELF file headers acting as our treasure map, directing us to the precise location of the section headers in the file (`e_shoff`), detailing their entry size (`e_shentsize`) and counting their entries (`e_shnum`).


I've also whipped up a nifty little parser, just for the occasion. It's designed to gracefully dissect an ELF file and lay out the section headers in a more digestible and user-friendly fashion. No more cryptic `hexdumps` or `xxd` outputs for us!



```
[ + ] Section headers begins at: 0x34b8
 [ 00 ] Section Name:                            Type: 0x0       Flags: 0x0      Addr: 0x0       Offset: 0x0             Size: 0         Link: 0         Info: 0x0       Addralign: 0x0          Entsize: 0
 [ 01 ] Section Name: .interp                    Type: 0x1       Flags: 0x2      Addr: 0x318     Offset: 0x318           Size: 28        Link: 0         Info: 0x0       Addralign: 0x1          Entsize: 0
 [ 02 ] Section Name: .note.gnu.property         Type: 0x7       Flags: 0x2      Addr: 0x338     Offset: 0x338           Size: 64        Link: 0         Info: 0x0       Addralign: 0x8          Entsize: 0
 [ 03 ] Section Name: .note.gnu.build-id         Type: 0x7       Flags: 0x2      Addr: 0x378     Offset: 0x378           Size: 36        Link: 0         Info: 0x0       Addralign: 0x4          Entsize: 0
 [ 04 ] Section Name: .note.ABI-tag              Type: 0x7       Flags: 0x2      Addr: 0x39c     Offset: 0x39c           Size: 32        Link: 0         Info: 0x0       Addralign: 0x4          Entsize: 0
 [ 05 ] Section Name: .gnu.hash                  Type: 0xfff6    Flags: 0x2      Addr: 0x3c0     Offset: 0x3c0           Size: 28        Link: 6         Info: 0x0       Addralign: 0x8          Entsize: 0
 [ 06 ] Section Name: .dynsym                    Type: 0xb       Flags: 0x2      Addr: 0x3e0     Offset: 0x3e0           Size: 168       Link: 7         Info: 0x1       Addralign: 0x8          Entsize: 24
 [ 07 ] Section Name: .dynstr                    Type: 0x3       Flags: 0x2      Addr: 0x488     Offset: 0x488           Size: 144       Link: 0         Info: 0x0       Addralign: 0x1          Entsize: 0
 [ 08 ] Section Name: .gnu.version               Type: 0xffff    Flags: 0x2      Addr: 0x518     Offset: 0x518           Size: 14        Link: 6         Info: 0x0       Addralign: 0x2          Entsize: 2
 [ 09 ] Section Name: .gnu.version_r             Type: 0xfffe    Flags: 0x2      Addr: 0x528     Offset: 0x528           Size: 48        Link: 7         Info: 0x1       Addralign: 0x8          Entsize: 0
 [ 10 ] Section Name: .rela.dyn                  Type: 0x4       Flags: 0x2      Addr: 0x558     Offset: 0x558           Size: 192       Link: 6         Info: 0x0       Addralign: 0x8          Entsize: 24
 [ 11 ] Section Name: .rela.plt                  Type: 0x4       Flags: 0x42     Addr: 0x618     Offset: 0x618           Size: 24        Link: 6         Info: 0x17      Addralign: 0x8          Entsize: 24
 [ 12 ] Section Name: .init                      Type: 0x1       Flags: 0x6      Addr: 0x1000    Offset: 0x1000          Size: 27        Link: 0         Info: 0x0       Addralign: 0x4          Entsize: 0
 [ 13 ] Section Name: .plt                       Type: 0x1       Flags: 0x6      Addr: 0x1020    Offset: 0x1020          Size: 32        Link: 0         Info: 0x0       Addralign: 0x10         Entsize: 16
 [ 14 ] Section Name: .text                      Type: 0x1       Flags: 0x6      Addr: 0x1040    Offset: 0x1040          Size: 315       Link: 0         Info: 0x0       Addralign: 0x10         Entsize: 0
 [ 15 ] Section Name: .fini                      Type: 0x1       Flags: 0x6      Addr: 0x117c    Offset: 0x117c          Size: 13        Link: 0         Info: 0x0       Addralign: 0x4          Entsize: 0
 [ 16 ] Section Name: .rodata                    Type: 0x1       Flags: 0x2      Addr: 0x2000    Offset: 0x2000          Size: 18        Link: 0         Info: 0x0       Addralign: 0x4          Entsize: 0
 [ 17 ] Section Name: .eh_frame_hdr              Type: 0x1       Flags: 0x2      Addr: 0x2014    Offset: 0x2014          Size: 36        Link: 0         Info: 0x0       Addralign: 0x4          Entsize: 0
 [ 18 ] Section Name: .eh_frame                  Type: 0x1       Flags: 0x2      Addr: 0x2038    Offset: 0x2038          Size: 124       Link: 0         Info: 0x0       Addralign: 0x8          Entsize: 0
 [ 19 ] Section Name: .init_array                Type: 0xe       Flags: 0x3      Addr: 0x3dd0    Offset: 0x2dd0          Size: 8         Link: 0         Info: 0x0       Addralign: 0x8          Entsize: 8
 [ 20 ] Section Name: .fini_array                Type: 0xf       Flags: 0x3      Addr: 0x3dd8    Offset: 0x2dd8          Size: 8         Link: 0         Info: 0x0       Addralign: 0x8          Entsize: 8
 [ 21 ] Section Name: .dynamic                   Type: 0x6       Flags: 0x3      Addr: 0x3de0    Offset: 0x2de0          Size: 480       Link: 7         Info: 0x0       Addralign: 0x8          Entsize: 16
 [ 22 ] Section Name: .got                       Type: 0x1       Flags: 0x3      Addr: 0x3fc0    Offset: 0x2fc0          Size: 40        Link: 0         Info: 0x0       Addralign: 0x8          Entsize: 8
 [ 23 ] Section Name: .got.plt                   Type: 0x1       Flags: 0x3      Addr: 0x3fe8    Offset: 0x2fe8          Size: 32        Link: 0         Info: 0x0       Addralign: 0x8          Entsize: 8
 [ 24 ] Section Name: .data                      Type: 0x1       Flags: 0x3      Addr: 0x4008    Offset: 0x3008          Size: 16        Link: 0         Info: 0x0       Addralign: 0x8          Entsize: 0
 [ 25 ] Section Name: .bss                       Type: 0x8       Flags: 0x3      Addr: 0x4018    Offset: 0x3018          Size: 8         Link: 0         Info: 0x0       Addralign: 0x1          Entsize: 0
 [ 26 ] Section Name: .comment                   Type: 0x1       Flags: 0x30     Addr: 0x0       Offset: 0x3018          Size: 27        Link: 0         Info: 0x0       Addralign: 0x1          Entsize: 1
 [ 27 ] Section Name: .symtab                    Type: 0x2       Flags: 0x0      Addr: 0x0       Offset: 0x3038          Size: 576       Link: 28        Info: 0x6       Addralign: 0x8          Entsize: 24
 [ 28 ] Section Name: .strtab                    Type: 0x3       Flags: 0x0      Addr: 0x0       Offset: 0x3278          Size: 298       Link: 0         Info: 0x0       Addralign: 0x1          Entsize: 0
 [ 29 ] Section Name: .shstrtab                  Type: 0x3       Flags: 0x0      Addr: 0x0       Offset: 0x33a2          Size: 278       Link: 0         Info: 0x0       Addralign: 0x1          Entsize: 0
```

Here's a sneak peek at what my trusty parser churned out, for your reference. (If you fancy, you can even pit it against the `xxd` results we saw earlier).

Now, it's time to take a deep dive into the inner workings of the `Elf64_Shdr` struct

### 1. sh_name

As I mentioned earlier, among so many sections of an ELF file, there's one special place known as the string table. In this mystical realm, the names of all sections are held in a null-terminated fashion, creating a seamless string of section names. Now, the `sh_name` member, well, it's like a treasure map, pinpointing the exact **offset** within that section. So, if, for instance, `.interp` resides at `X1` bytes within the section, and this section itself is tucked away at `Y1` bytes into the file, the location of this string can be calculated as simply `X1 + Y1` bytes into the file. But, for the sake of simplicity, `sh_name` keeps things straightforward by storing just the `X1` value, and nothing more. To track down the section's exact location, we can rely on the trusty `e_shstrndx` value from the ELF file header.

From programming point of view, accessing the string value for section name will look something like -

```c
(char*)(shdr[ehdr->e_shstrndx].sh_offset + shdr[i].sh_name)
```

### 2. sh_type

This section serves as a delightful teaser, offering a glimpse of the treasures awaiting inside the section itself. Take, for instance, `SHT_STRTAB` (0x3), a section that houses a collection of null-terminated strings, just waiting to be discovered.

When we journey into the Linux kernel, we encounter a bunch of defined section header types -

```c
/*
https://elixir.bootlin.com/linux/v6.5.8/source/include/uapi/linux/elf.h#L271
*/
/* sh_type */
#define SHT_NULL      0
#define SHT_PROGBITS  1
#define SHT_SYMTAB    2
#define SHT_STRTAB    3
#define SHT_RELA      4
#define SHT_HASH      5
#define SHT_DYNAMIC   6
#define SHT_NOTE      7
#define SHT_NOBITS    8
#define SHT_REL       9
#define SHT_SHLIB     10
#define SHT_DYNSYM    11
#define SHT_NUM       12
#define SHT_LOPROC    0x70000000
#define SHT_HIPROC    0x7fffffff
#define SHT_LOUSER    0x80000000
#define SHT_HIUSER    0xffffffff
```

### 3. sh_flags


This is a one-bit flag, that decides whether a specific feature applies to the given section or not...
Linux kernel has some flag types defined -

```c
/*
https://elixir.bootlin.com/linux/v6.5.8/source/include/uapi/linux/elf.h#L290
*/
/* sh_flags */
#define SHF_WRIT            0x1
#define SHF_ALLOC           0x2
#define SHF_EXECINSTR       0x4
#define SHF_RELA_LIVEPATCH  0x00100000
#define SHF_RO_AFTER_INIT   0x00200000
#define SHF_MASKPROC        0xf0000000
```

Playing the guessing game? Well, if you spot a section like `.text` with a type value of `0x6`, it's a hint at what's to come. This section will be allocated a space in memory at runtime, with permission to execute instructions, but don't even think about writing anything to it after the section is loaded.


### 4. sh_addr

Now, if the section is destined for memory, this member plays a pivotal role, holding the keys to the memory kingdom, designating the precise spot where the section lands. But here's a twist – for sections with no memory aspirations, this value becomes a mere placeholder, leaving a little room for some extra, secret bytes. *(wink, wink)*

### 5. sh_offset

Here's the catch: while `sh_addr` spills the beans on the section's memory location, this member focuses on the section's spot in the file. It's like knowing where the script lies before the performance. However, some sections, like the enigmatic `SHT_NOBITS`, are a bit of a puzzle – they claim a spot in the file, but when you try to read data from their supposed location, it's like chasing a ghost; there's nothing substantial to be found. (that is, they don't take any space in file; like a classic "all bark, no bite" scenario)

(**HINT**: Look at offsets and size of `.bss` and `.comment` sections from above listing. `.bss` is a `SHT_NOBITS` kind of section.)

### 6. sh_size

For sections that aren't the enigmatic `SHT_NOBITS` type, this value is a trustworthy measure, mapping out the precise size (in bytes) of the section within the file. For `SHT_NOBITS`, it's a bit of a riddle. While it claims to reveal a section's size in bytes, be warned that when you glance at the size of a `.bss` section and it does says `8` bytes. But again, since there is nothing in the file, it's more of a conceptual size for this type.



### 7. sh_link

This member is used to link a section with another section. One of the use for such kind of linking is to signify some sort of dependency of one section on another. But the actual nature of linking depends on the section type.

(HINT: Checkout `.gnu_hash`, `.dynsym`, and `dynstr` sections)

### 8. sh_info

Think of this member as the mysterious vault, holding extra information that's tailor-made for the section's needs. However, the contents of this vault are shapeshifters, and what you'll find inside depends entirely on the section's unique personality and type.

### 9. sh_addralign

This member holds the alignment information. When it takes on the humble value of `0` or `1`, it's like saying, "No alignment required." But when it strides into the realm of positive powers of `2`, it becomes the architect of alignment, ensuring that the section is perfectly orchestrated for maximum efficiency.


*Alignment is the unsung hero in the world of efficient computing. It's the magic behind how smoothly a computer can access and manipulate data or instructions.*


### 10. sh_entsize

Picture it: there are sections that harbor orderly tables with entries of a fixed size. Now, this member is your trusted guide, revealing the size of each entry in bytes. To find the grand total of entries, you simply divide the section's size by the size of each entry, just like a mathematical maestro.



(NOTE: You can read more about ELF sections and each member of section headers from [`man 5 elf`](https://linux.die.net/man/5/elf); RTFM)


## Practicals

For now, let's just start with what does `strip` command do to ELF sections. And research on why section headers are actually important.

If you are more inclined towards being tech savvy, try to write a program to parse and display the section headers.

To go an extra mile, add a new section to your ELF file (also add it's entry in section headers)...


Here are some links that might give you a starting point.
- https://stackoverflow.com/questions/1088128/adding-section-to-elf-file
- https://reverseengineering.stackexchange.com/questions/14779/how-to-successfully-add-a-code-section-to-an-executable-file-in-linux
- https://stackoverflow.com/questions/29058016/efficiently-adding-a-new-section-in-an-elf-file


## Conclusion

Alright, buckle up, because we've just taken a deep dive into the wild world of ELF section headers! Picture this -
```

   ┌───────────────────────────┐
   │                           │
   │      File Header          │
   │                           │
   │                           │
   ├───────────────────────────┤
   │                           │
   │     Program Header        │
   │                           │
   │                           │
   ├───────────────────────────┤
   │                           │
   │                           │
   │      Section 1            │
   │                           │
   ├───────────────────────────┤
   │      Section 2            │
   ├───────────────────────────┤
   │                           │
   │      Section 3            │
   ├───────────────────────────┤
   │                           │
   │                           │
   │                           │
   │                           │
   │                           │
   │      Section 4            │
   │                           │
   │                           │
   │                           │
   │                           │
   ├───────────────────────────┤
   │                           │
   │                           │
   │      Section 5            │
   │                           │
   │                           │
   │                           │
   ├───────────────────────────┤
   │                           │
   │                           │
   │     Section 6             │
   │                           │
   │                           │
   ├───────────────────────────┤
   │                           │
   │                           │
   │     Section Header        │
   │                           │
   │                           │
   └───────────────────────────┘

```



Think of sections as pieces of a puzzle, each unique in size and placed at different offsets within the file. But fear not, for the section headers play the role of meticulous architects, documenting these diverse sections' whereabouts and characteristics. They're the cool blueprints that grant us insight into the entire file's layout and functionality.
