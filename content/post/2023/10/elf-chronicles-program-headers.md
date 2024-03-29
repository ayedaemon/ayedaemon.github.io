---
title: "ELF Chronicles: Program Headers (3/?)"
date: 2023-10-20T15:21:49+05:30
draft: false
# showtoc: false
tags: ["C", "ELF", "RE"]
series:
  - "ELF Chronicles"
description: "Exploring ELF program Headers"
# math: true
# ShowBreadCrumbs: false
# searchHidden: true
# hideMeta: true
---

In preceding articles, we've delved into the details of ELF file headers and section headers. Section headers provide insight into how data and instructions are organized based on their characteristics and grouped into distinct sections. These sections remain distinct due to variations in their types and permissions (*... and few other things*).

Up to this point, our focus has been on the aspects of the ELF file as it resides on-disk. However, we now turn our attention to what occurs when the file is loaded into memory. How is its arrangement handled? Are all the sections loaded into memory?



This is where the concept of program headers comes into play. Program headers are similar to section headers, but instead of section information, they store segment information. A segment encompasses one or more sections from the ELF file. While program headers hold little significance while the file is on disk, they become imperative when the file needs to be loaded and executed in memory, specifically in the case of executables and shared objects.


Some criteria for grouping sections to form segments can be:
- Type and purpose of the sections (like `.data` and `.bss`),
- Memory Access Permissions and mapping,
- Alignment and Layout,
- Segment size constraints,
- OS and platform requirements, etc


For this article, I'll be using the same C code to generate an ELF file

```c
/*
File: hello_world.c
Compile: gcc hello_world.c -o hello_world
*/

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

Once you have the ELF file, you can get the program header related information from ELF file headers - `e_phoff`, `e_phentsize` and `e_phnum`

I'll use readelf to get this information from the ELF headers. Feel free to use any method of your choice.

```
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
  Entry point address:               0x1040
  Start of program headers:          64 (bytes into file)
  Start of section headers:          13496 (bytes into file)
  Flags:                             0x0
  Size of this header:               64 (bytes)
  Size of program headers:           56 (bytes)
  Number of program headers:         13
  Size of section headers:           64 (bytes)
  Number of section headers:         30
  Section header string table index: 29
```


From the the output above, we can deduce that
- the program headers are located at offset of `64` bytes,
- each of these header entries is `56` bytes in size,
- and in total, we've got `13` entries


Now we can use xxd to get the data out

```
❯ xxd -s 64 -l $(( 54*13 )) -c 54 build/hello
00000040: 0600 0000 0400 0000 4000 0000 0000 0000 4000 0000 0000 0000 4000 0000 0000 0000 d802 0000 0000 0000 d802 0000 0000 0000 0800 0000 0000  ........@.......@.......@.............................
00000076: 0000 0300 0000 0400 0000 1803 0000 0000 0000 1803 0000 0000 0000 1803 0000 0000 0000 1c00 0000 0000 0000 1c00 0000 0000 0000 0100 0000  ......................................................
000000ac: 0000 0000 0100 0000 0400 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 3006 0000 0000 0000 3006 0000 0000 0000 0010  ....................................0.......0.........
000000e2: 0000 0000 0000 0100 0000 0500 0000 0010 0000 0000 0000 0010 0000 0000 0000 0010 0000 0000 0000 8901 0000 0000 0000 8901 0000 0000 0000  ......................................................
00000118: 0010 0000 0000 0000 0100 0000 0400 0000 0020 0000 0000 0000 0020 0000 0000 0000 0020 0000 0000 0000 b400 0000 0000 0000 b400 0000 0000  ................. ....... ....... ....................
0000014e: 0000 0010 0000 0000 0000 0100 0000 0600 0000 d02d 0000 0000 0000 d03d 0000 0000 0000 d03d 0000 0000 0000 4802 0000 0000 0000 5002 0000  ...................-.......=.......=......H.......P...
00000184: 0000 0000 0010 0000 0000 0000 0200 0000 0600 0000 e02d 0000 0000 0000 e03d 0000 0000 0000 e03d 0000 0000 0000 e001 0000 0000 0000 e001  .....................-.......=.......=................
000001ba: 0000 0000 0000 0800 0000 0000 0000 0400 0000 0400 0000 3803 0000 0000 0000 3803 0000 0000 0000 3803 0000 0000 0000 4000 0000 0000 0000  ......................8.......8.......8.......@.......
000001f0: 4000 0000 0000 0000 0800 0000 0000 0000 0400 0000 0400 0000 7803 0000 0000 0000 7803 0000 0000 0000 7803 0000 0000 0000 4400 0000 0000  @.......................x.......x.......x.......D.....
00000226: 0000 4400 0000 0000 0000 0400 0000 0000 0000 53e5 7464 0400 0000 3803 0000 0000 0000 3803 0000 0000 0000 3803 0000 0000 0000 4000 0000  ..D...............S.td....8.......8.......8.......@...
0000025c: 0000 0000 4000 0000 0000 0000 0800 0000 0000 0000 50e5 7464 0400 0000 1420 0000 0000 0000 1420 0000 0000 0000 1420 0000 0000 0000 2400  ....@...............P.td..... ....... ....... ......$.
00000292: 0000 0000 0000 2400 0000 0000 0000 0400 0000 0000 0000 51e5 7464 0600 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  ......$...............Q.td............................
000002c8: 0000 0000 0000 0000 0000 0000 0000 0000 1000 0000 0000 0000 52e5 7464 0400 0000 d02d 0000 0000 0000 d03d 0000 0000 0000 d03d 0000 0000  ........................R.td.....-.......=.......=....
```

Now we just have to map each of these lines to `Elf64_Phdr` (since we have a 64Bit file)


```
/*
https://elixir.bootlin.com/linux/latest/source/include/uapi/linux/elf.h#L260
*/

typedef struct elf64_phdr {
  Elf64_Word p_type;      /* Segment type */
  Elf64_Word p_flags;     /* Segment flags */
  Elf64_Off p_offset;     /* Segment file offset */
  Elf64_Addr p_vaddr;     /* Segment virtual address */
  Elf64_Addr p_paddr;     /* Segment physical address */
  Elf64_Xword p_filesz;   /* Segment size in file */
  Elf64_Xword p_memsz;    /* Segment size in memory */
  Elf64_Xword p_align;    /* Segment alignment, file & memory */
} Elf64_Phdr;
```


Using my nifty little parser, I got this digestible and user-friendly output for the above dump (Feel free to compare it)

```
[ + ] Program headers begins at: 0x40
 [ 00 ] Type: 0x6        Flags: 0x4      Offset: 0x0040          vaddr: 0x40     paddr: 0x40     filesz: 0x728           memsz: 0x728            align: 0x8
 [ 01 ] Type: 0x3        Flags: 0x4      Offset: 0x0318          vaddr: 0x318    paddr: 0x318    filesz: 0x28            memsz: 0x28             align: 0x1
 [ 02 ] Type: 0x1        Flags: 0x4      Offset: 0x0000          vaddr: 0x0      paddr: 0x0      filesz: 0x1584          memsz: 0x1584           align: 0x1000
 [ 03 ] Type: 0x1        Flags: 0x5      Offset: 0x1000          vaddr: 0x1000   paddr: 0x1000   filesz: 0x393           memsz: 0x393            align: 0x1000
 [ 04 ] Type: 0x1        Flags: 0x4      Offset: 0x2000          vaddr: 0x2000   paddr: 0x2000   filesz: 0x180           memsz: 0x180            align: 0x1000
 [ 05 ] Type: 0x1        Flags: 0x6      Offset: 0x2dd0          vaddr: 0x3dd0   paddr: 0x3dd0   filesz: 0x584           memsz: 0x592            align: 0x1000
 [ 06 ] Type: 0x2        Flags: 0x6      Offset: 0x2de0          vaddr: 0x3de0   paddr: 0x3de0   filesz: 0x480           memsz: 0x480            align: 0x8
 [ 07 ] Type: 0x4        Flags: 0x4      Offset: 0x0338          vaddr: 0x338    paddr: 0x338    filesz: 0x64            memsz: 0x64             align: 0x8
 [ 08 ] Type: 0x4        Flags: 0x4      Offset: 0x0378          vaddr: 0x378    paddr: 0x378    filesz: 0x68            memsz: 0x68             align: 0x4
 [ 09 ] Type: 0xe553     Flags: 0x4      Offset: 0x0338          vaddr: 0x338    paddr: 0x338    filesz: 0x64            memsz: 0x64             align: 0x8
 [ 10 ] Type: 0xe550     Flags: 0x4      Offset: 0x2014          vaddr: 0x2014   paddr: 0x2014   filesz: 0x36            memsz: 0x36             align: 0x4
 [ 11 ] Type: 0xe551     Flags: 0x6      Offset: 0x0000          vaddr: 0x0      paddr: 0x0      filesz: 0x0             memsz: 0x0              align: 0x10
 [ 12 ] Type: 0xe552     Flags: 0x4      Offset: 0x2dd0          vaddr: 0x3dd0   paddr: 0x3dd0   filesz: 0x560           memsz: 0x560            align: 0x1
```


Now, it's time to take a deep dive into the inner workings of the `Elf64_Phdr` struct


### 1. p_type

Just like `sh_type`, this member tells the type of the segment. Whether the segment will be loaded in the memory or is it just used to store notes.

```
/*
https://elixir.bootlin.com/linux/latest/source/include/uapi/linux/elf.h#L25
*/

/* These constants are for the segment types stored in the image headers */
#define PT_NULL    0
#define PT_LOAD    1
#define PT_DYNAMIC 2
#define PT_INTERP  3
#define PT_NOTE    4
#define PT_SHLIB   5
#define PT_PHDR    6
#define PT_TLS     7               /* Thread local storage segment */
#define PT_LOOS    0x60000000      /* OS-specific */
#define PT_HIOS    0x6fffffff      /* OS-specific */
#define PT_LOPROC  0x70000000
#define PT_HIPROC  0x7fffffff
#define PT_GNU_EH_FRAME	(PT_LOOS + 0x474e550)
#define PT_GNU_STACK	(PT_LOOS + 0x474e551)
#define PT_GNU_RELRO	(PT_LOOS + 0x474e552)
#define PT_GNU_PROPERTY	(PT_LOOS + 0x474e553)
```


### 2. p_flags

This is quite similar to the the `(r)ead`, `(w)rite` and `e(x)ecute` permissions we are familiar with. This member specifies the permissions for the given segment.

Usually the segment containing the `.text` section will have `(r)ead` and `e(x)ecute` permissions.

```
/*
https://elixir.bootlin.com/linux/latest/source/include/uapi/linux/elf.h#L243
*/

/* These constants define the permissions on sections in the program
   header, p_flags. */

#define PF_R    0x4
#define PF_W    0x2
#define PF_X    0x1
```

### 3. p_offset
This holds the offset from the beginning of the file, where the first byte of the first section in this segment is located.

### 4. p_vaddr

This member holds the memory/virtual address for the segment.

### 5. p_paddr

This is same as `p_vaddr`, but holds the physical/on-disk address for the segment.


### 6. p_filesz

This holds the on-disk size (in bytes) of the segment.

### 7. p_memsz

This member holds the memory/virtual size (in bytes) of the segment.

### 8. p_align

This member holds the value to which the segments are aligned in memory and in the file.

Similar to `sh_addralign`, value of `0` and `1` are treated as "no alignment", while the positive powers of `2` are taken as the actual alignment values.



## Practicals

Let's start with checking if `strip` command makes any change to the program headers.

- Try to write a program to parse the program headers and display the information in better way.
- Try to write a program that gives the information about what sections are grouped together in a segment. `readelf` gives this information in below format

```
Program Headers:
  Type           Offset   VirtAddr           PhysAddr           FileSiz  MemSiz   Flg Align
  PHDR           0x000040 0x0000000000000040 0x0000000000000040 0x0002d8 0x0002d8 R   0x8
  INTERP         0x000318 0x0000000000000318 0x0000000000000318 0x00001c 0x00001c R   0x1
      [Requesting program interpreter: /lib64/ld-linux-x86-64.so.2 ]
  LOAD           0x000000 0x0000000000000000 0x0000000000000000 0x000630 0x000630 R   0x1000
  LOAD           0x001000 0x0000000000001000 0x0000000000001000 0x000189 0x000189 R E 0x1000
  LOAD           0x002000 0x0000000000002000 0x0000000000002000 0x0000b4 0x0000b4 R   0x1000
  LOAD           0x002dd0 0x0000000000003dd0 0x0000000000003dd0 0x000248 0x000250 RW  0x1000
  DYNAMIC        0x002de0 0x0000000000003de0 0x0000000000003de0 0x0001e0 0x0001e0 RW  0x8
  NOTE           0x000338 0x0000000000000338 0x0000000000000338 0x000040 0x000040 R   0x8
  NOTE           0x000378 0x0000000000000378 0x0000000000000378 0x000044 0x000044 R   0x4
  GNU_PROPERTY   0x000338 0x0000000000000338 0x0000000000000338 0x000040 0x000040 R   0x8
  GNU_EH_FRAME   0x002014 0x0000000000002014 0x0000000000002014 0x000024 0x000024 R   0x4
  GNU_STACK      0x000000 0x0000000000000000 0x0000000000000000 0x000000 0x000000 RW  0x10
  GNU_RELRO      0x002dd0 0x0000000000003dd0 0x0000000000003dd0 0x000230 0x000230 R   0x1

 Section to Segment mapping:
  Segment Sections...
   00
   01     .interp
   02     .interp .note.gnu.property .note.gnu.build-id .note.ABI-tag .gnu.hash .dynsym .dynstr .gnu.version .gnu.version_r .rela.dyn .rela.plt
   03     .init .plt .text .fini
   04     .rodata .eh_frame_hdr .eh_frame
   05     .init_array .fini_array .dynamic .got .got.plt .data .bss
   06     .dynamic
   07     .note.gnu.property
   08     .note.gnu.build-id .note.ABI-tag
   09     .note.gnu.property
   10     .eh_frame_hdr
   11
   12     .init_array .fini_array .dynamic .got
```

If you want to go extra mile and dig deep,
- Try overwriting the program interpreter with your custom loader program. Things will probably go wrong and then you can dig deep what's the root cause.
- Add a new section (`.text` type), create it's section header entry, then create it's program header entry such that it is loadable in memory. Then change the ELF entrypoint to the newly created section.


## Conclusion

Alright, buckle up, because we have just seen what segments are, how sections are grouped into segments, and how program headers act as a table to store information about segments which is helpful for runtime. Picture this -


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
  ├───────────────────────────┤  ◄───┐
  │                           │      │
  │                           │      │
  │      Section 1            │      │
  │                           │      │
  ├───────────────────────────┤      │ Segment 1
  │      Section 2            │      │
  ├───────────────────────────┤      │
  │                           │      │
  │      Section 3            │      │
  ├───────────────────────────┤  ◄───┤
  │                           │      │
  │                           │      │
  │                           │      │
  │                           │      │ Segment 2
  │                           │      │
  │      Section 4            │      │
  │                           │      │
  │                           │  ◄───┤
  │                           │      │ Segment 3
  │                           │      │
  ├───────────────────────────┤  ◄───┤
  │                           │      │
  │                           │      │
  │      Section 5            │      │
  │                           │      │ Segment 4
  │                           │      │
  │                           │      │
  ├───────────────────────────┤  ◄───┤
  │                           │      │
  │                           │      │
  │     Section 6             │      │ Segment 5
  │                           │      │
  │                           │      │
  ├───────────────────────────┤  ◄───┘
  │                           │
  │                           │
  │     Section Header        │
  │                           │
  │                           │
  └───────────────────────────┘

```
