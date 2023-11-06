---
title: "Elf Chronicles: String Tables"
date: 2023-10-29T15:12:36+05:30
draft: true
# showtoc: false
tags:
    - "C"
    - "ELF"
    - "RE"
series:
    - "ELF Chronicles"
description: "Exploring ELF string tables"
# math: true
# ShowBreadCrumbs: false
# searchHidden: true
# hideMeta: true
---


In the article about section headers, you got [an introduction to string tables](https://ayedaemon.github.io/post/2023/10/elf-chronicles-section-headers/#1-sh_name). In this article, we will delve deeper into the topic.


## String table

So, here's the deal: when you've got a bunch of characters, and you end them with a null character, that whole thing is what we call a "string." (*At least, that's what I've learned, and I'm sticking with it for now.*)


Now, when it comes to a string table, it's pretty simple. It's just a bunch of these strings all lined up, one after the other. The only twist is that the first string is always null (just a null char - `\0`). Now you can put all that data in a section and create a section header for it with **type** - `SHT_STRTAB`(which is just `0x3` in fancy lingo). And voila, you've got yourself a proper string table, with a section header entry for it.


If you want to picture it, think of it like this - a string table is like a list of strings, where the first one is always an empty string.


```
## Every 00 is a null char (in hex)
# For  Section=.shstrtab (Offset: 0x348b, Size: 278)

❯ xxd -s 0x348b -l 278 hello

0000348b: 002e 7379 6d74 6162 002e 7374 7274 6162  ..symtab..strtab
0000349b: 002e 7368 7374 7274 6162 002e 696e 7465  ..shstrtab..inte
000034ab: 7270 002e 6e6f 7465 2e67 6e75 2e70 726f  rp..note.gnu.pro
000034bb: 7065 7274 7900 2e6e 6f74 652e 676e 752e  perty..note.gnu.
000034cb: 6275 696c 642d 6964 002e 6e6f 7465 2e41  build-id..note.A
000034db: 4249 2d74 6167 002e 676e 752e 6861 7368  BI-tag..gnu.hash
000034eb: 002e 6479 6e73 796d 002e 6479 6e73 7472  ..dynsym..dynstr
000034fb: 002e 676e 752e 7665 7273 696f 6e00 2e67  ..gnu.version..g
0000350b: 6e75 2e76 6572 7369 6f6e 5f72 002e 7265  nu.version_r..re
0000351b: 6c61 2e64 796e 002e 7265 6c61 2e70 6c74  la.dyn..rela.plt
0000352b: 002e 696e 6974 002e 7465 7874 002e 6669  ..init..text..fi
0000353b: 6e69 002e 726f 6461 7461 002e 6568 5f66  ni..rodata..eh_f
0000354b: 7261 6d65 5f68 6472 002e 6568 5f66 7261  rame_hdr..eh_fra
0000355b: 6d65 002e 696e 6974 5f61 7272 6179 002e  me..init_array..
0000356b: 6669 6e69 5f61 7272 6179 002e 6479 6e61  fini_array..dyna
0000357b: 6d69 6300 2e67 6f74 002e 676f 742e 706c  mic..got..got.pl
0000358b: 7400 2e64 6174 6100 2e62 7373 002e 636f  t..data..bss..co
0000359b: 6d6d 656e 7400                           mment.
```


It should be pretty easy to write a parser for this, if not, ask your friend to do it for you

```
# Format: [ Offset (in section) ] String value

[    0 ]
[    1 ] .symtab
[    9 ] .strtab
[   17 ] .shstrtab
[   27 ] .interp
[   35 ] .note.gnu.property
[   54 ] .note.gnu.build-id
[   73 ] .note.ABI-tag
[   87 ] .gnu.hash
[   97 ] .dynsym
[  105 ] .dynstr
[  113 ] .gnu.version
[  126 ] .gnu.version_r
[  141 ] .rela.dyn
[  151 ] .rela.plt
[  161 ] .init
[  167 ] .text
[  173 ] .fini
[  179 ] .rodata
[  187 ] .eh_frame_hdr
[  201 ] .eh_frame
[  211 ] .init_array
[  223 ] .fini_array
[  235 ] .dynamic
[  244 ] .got
[  249 ] .got.plt
[  258 ] .data
[  264 ] .bss
[  269 ] .comment
```


Now, to proceed, let's take a look at the C program that I'll be using in this article.

```c
/*
file: hello.c
*/
#include <stdio.h>

int global1;
char global2 = 'x';
static int global3 = 9;

static void print_globals(void) {
    printf("global1 = %d (%p) | global2 = %c (%p) | global3 = %d (%p)\n",
        global1, &global1,
        global2, &global2,
        global3, &global3
    );
}

int main(){

    int local1;
    char local2 = 'y';
    static int local3 = 6;

    printf("Main: %p\n", &main);

    print_globals();

    printf("local1 = %d (%p) | local2 = %c (%p) | local3 = %d (%p)\n",
        local1, &local1,
        local2, &local2,
        local3, &local3
    );
    return 0;
}

```

I assume you can compile it and create the ELF binary. After the ELF binary is ready, analyze it to extract the list of all sections with a type of `0x3` (feeling fancy - `SHT_STRTAB`).

Using my pretty parser, I found three entries. Feel free to use `readelf`, `hexdump`, `xxd`, or any tool you prefer – the output should be same, regardless of your choice.

```
[ 07 ] Section Name: .dynstr        Type: 0x3       Flags: 0x2      Addr: 0x4a0     Offset: 0x4a0           Size: 170       Link: 0         Info: 0x0       Addralign: 0x1          Entsize: 0
[ 28 ] Section Name: .strtab        Type: 0x3       Flags: 0x0      Addr: 0x0       Offset: 0x3318          Size: 371       Link: 0         Info: 0x0       Addralign: 0x1          Entsize: 0
[ 29 ] Section Name: .shstrtab      Type: 0x3       Flags: 0x0      Addr: 0x0       Offset: 0x348b          Size: 278       Link: 0         Info: 0x0       Addralign: 0x1          Entsize: 0
```


String tables consist exclusively of strings. This data doesn't serve much purpose unless those strings are needed by other sections. So, let's examine them closely, one by one.


### 1. `.shstrtab`

Recall the good old days when you'd inspect ELF file headers with a command like `readelf --file-header hello.`


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
  Entry point address:               0x1050
  Start of program headers:          64 (bytes into file)
  Start of section headers:          13736 (bytes into file)
  Flags:                             0x0
  Size of this header:               64 (bytes)
  Size of program headers:           56 (bytes)
  Number of program headers:         13
  Size of section headers:           64 (bytes)
  Number of section headers:         30
  Section header string table index: 29
```


... and you would get the `section header string table index` value from it. This value tells you the index of the "section header string table" (abbreviated as `shstrtab`). This table holds the names of all the sections, and each section has a `sh_name` member that stores the offset of its name from this table.

To retrieve the name of any section, you'd need to extract the offset value from the sh_name member of that section. Then, you can navigate to that offset within the shstrtab section, and that's where you'll find the name of the section!

Question: Why are the section names stored in a separate dedicated section, rather than directly within each section's `sh_name` member??

Answer: While I can't say for certain, it's possible that this design choice was made to accommodate variable-length section names. Storing the names in a separate section allows flexibility in the length of section names and avoids any size constraints related to the `sh_name` member.

When parsed, the data of this section appears like this: an offset and the string stored at that offset.

```
[ 29 ] Section Name: .shstrtab      Type: 0x3       Flags: 0x0      Addr: 0x0       Offset: 0x348b          Size: 278       Link: 0         Info: 0x0       Addralign: 0x1          Entsize: 0
     [    0 ]
     [    1 ] .symtab
     [    9 ] .strtab
     [   17 ] .shstrtab
     [   27 ] .interp
     [   35 ] .note.gnu.property
     [   54 ] .note.gnu.build-id
     [   73 ] .note.ABI-tag
     [   87 ] .gnu.hash
     [   97 ] .dynsym
     [  105 ] .dynstr
     [  113 ] .gnu.version
     [  126 ] .gnu.version_r
     [  141 ] .rela.dyn
     [  151 ] .rela.plt
     [  161 ] .init
     [  167 ] .text
     [  173 ] .fini
     [  179 ] .rodata
     [  187 ] .eh_frame_hdr
     [  201 ] .eh_frame
     [  211 ] .init_array
     [  223 ] .fini_array
     [  235 ] .dynamic
     [  244 ] .got
     [  249 ] .got.plt
     [  258 ] .data
     [  264 ] .bss
     [  269 ] .comment
```


### 2. `.strtab`

This section contains strings (:P), mostly the ones representing names linked to symbol table entries (we'll dive deeper into symbol tables later). But at a quick glance, you can spot some of the names for variables and function names we used in our C program, such as `global3`, `print_globals`, `main` and so on.

Keep in mind that this section does not hold strings which are used by programs like the ones used with `printf` function.

```
[ 28 ] Section Name: .strtab        Type: 0x3       Flags: 0x0      Addr: 0x0       Offset: 0x3318          Size: 371       Link: 0         Info: 0x0       Addralign: 0x1          Entsize: 0
     [    0 ]
     [    1 ] hello.c
     [    9 ] global3
     [   17 ] print_globals
     [   31 ] local3.0
     [   40 ] _DYNAMIC
     [   49 ] __GNU_EH_FRAME_HDR
     [   68 ] _GLOBAL_OFFSET_TABLE_
     [   90 ] __libc_start_main@GLIBC_2.34
     [  119 ] _ITM_deregisterTMCloneTable
     [  147 ] _edata
     [  154 ] _fini
     [  160 ] __stack_chk_fail@GLIBC_2.4
     [  187 ] printf@GLIBC_2.2.5
     [  206 ] global1
     [  214 ] __data_start
     [  227 ] __gmon_start__
     [  242 ] __dso_handle
     [  255 ] _IO_stdin_used
     [  270 ] _end
     [  275 ] __bss_start
     [  287 ] main
     [  292 ] __TMC_END__
     [  304 ] _ITM_registerTMCloneTable
     [  330 ] __cxa_finalize@GLIBC_2.2.5
     [  357 ] _init
     [  363 ] global2
```

### 3. `.dynstr`

Similar to `strtab`, this section contains strings for symbol table entries, but these symbols come into play during runtime, often as part of dynamic linking. Because this section is used for dynamic linking, this needs to be loaded into memory for runtime use. You can confirm that with the `sh_flags` value for this section (should be `0x2` (or fancy, `SHF_ALLOC`))

For your satisfaction, here is the the output of `readelf --segments hello`, which indicates that this section is a part of the first `LOAD` segment

```
❯ readelf --segments --wide hello

Elf file type is DYN (Position-Independent Executable file)
Entry point 0x1050
There are 13 program headers, starting at offset 64

Program Headers:
  Type           Offset   VirtAddr           PhysAddr           FileSiz  MemSiz   Flg Align
  0 PHDR           0x000040 0x0000000000000040 0x0000000000000040 0x0002d8 0x0002d8 R   0x8
  1 INTERP         0x000318 0x0000000000000318 0x0000000000000318 0x00001c 0x00001c R   0x1
      [Requesting program interpreter: /lib64/ld-linux-x86-64.so.2]
  2 LOAD           0x000000 0x0000000000000000 0x0000000000000000 0x000690 0x000690 R   0x1000
  3 LOAD           0x001000 0x0000000000001000 0x0000000000001000 0x00024d 0x00024d R E 0x1000
  4 LOAD           0x002000 0x0000000000002000 0x0000000000002000 0x000154 0x000154 R   0x1000
  5 LOAD           0x002dd0 0x0000000000003dd0 0x0000000000003dd0 0x00025c 0x000268 RW  0x1000
  6 DYNAMIC        0x002de0 0x0000000000003de0 0x0000000000003de0 0x0001e0 0x0001e0 RW  0x8
  7 NOTE           0x000338 0x0000000000000338 0x0000000000000338 0x000040 0x000040 R   0x8
  8 NOTE           0x000378 0x0000000000000378 0x0000000000000378 0x000044 0x000044 R   0x4
  9 GNU_PROPERTY   0x000338 0x0000000000000338 0x0000000000000338 0x000040 0x000040 R   0x8
 10 GNU_EH_FRAME   0x002088 0x0000000000002088 0x0000000000002088 0x00002c 0x00002c R   0x4
 11 GNU_STACK      0x000000 0x0000000000000000 0x0000000000000000 0x000000 0x000000 RW  0x10
 12 GNU_RELRO      0x002dd0 0x0000000000003dd0 0x0000000000003dd0 0x000230 0x000230 R   0x1

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


But the section structure is still same as any other string table, so my cool parser parsed it.

```
 [ 07 ] Section Name: .dynstr       Type: 0x3       Flags: 0x2      Addr: 0x4a0     Offset: 0x4a0           Size: 170       Link: 0         Info: 0x0       Addralign: 0x1          Entsize: 0
     [    0 ]
     [    1 ] __cxa_finalize
     [   16 ] __libc_start_main
     [   34 ] __stack_chk_fail
     [   51 ] printf
     [   58 ] libc.so.6
     [   68 ] GLIBC_2.2.5
     [   80 ] GLIBC_2.4
     [   90 ] GLIBC_2.34
     [  101 ] _ITM_deregisterTMCloneTable
     [  129 ] __gmon_start__
     [  144 ] _ITM_registerTMCloneTable
```


## Conclusion

string tables in ELF files serve as repositories for various strings for section names, symbol names, and other dynamic linking data. The separation of string data into dedicated sections like "strtab" and "dynstr" allows for flexibility in string length and ensures that these essential strings are readily available during program execution.

Before closing this, I want you to run `strip` command against the ELF binary used in this article... Whatever happens will raise some good new questions for you to dig deeper (Some of those questions will be answered as we go forward with this series)
