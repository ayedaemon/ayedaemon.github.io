---
title: "Elf Chronicles: String Tables (4/?)"
date: 2023-10-29T15:12:36+05:30
draft: false
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


In the article about [section headers](https://ayedaemon.github.io/post/2023/10/elf-chronicles-section-headers), you got [an introduction to string tables](https://ayedaemon.github.io/post/2023/10/elf-chronicles-section-headers/#1-sh_name). In this article, we will delve deeper into the topic.


## ...prologue

We'll start with the same program we used in the previous article about section headers.

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

Compile this and then analyze the ELF executable file using `readelf` (Not everytime we'll go with `xxd`).

```
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


With the help of this, you can get the `section header table` of the file.


```
#################### Explaination ###########################
#
# xxd \
#   -s <start_of_section_headers> \               # Start of section headers:   13608 (bytes into file)
#   -l <total_size_of_all_section_headers> \      # size_of_one_section_header(64) * total_count_of_section_headers(30)
#   -c <bytes_to_print_in_a_single_line>   \      # Just to get a section header entry in a single line
#   <ELF_file> \                                  #  ... duhh!
#   | nl -v0 -                                    # I wanted to get the line numbers starting from 0. WHY 0?? - because that's where the array indexing starts
#############################################################

❯ xxd \
    -s 13608 \
    -l $(( 64*30 )) \
    -c 64 \
    hello_world \
    | nl -v0 -


 0  00003528: 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000 0000  ................................................................
 1  00003568: 1b00 0000 0100 0000 0200 0000 0000 0000 1803 0000 0000 0000 1803 0000 0000 0000 1c00 0000 0000 0000 0000 0000 0000 0000 0100 0000 0000 0000 0000 0000 0000 0000  ................................................................
 2  000035a8: 2300 0000 0700 0000 0200 0000 0000 0000 3803 0000 0000 0000 3803 0000 0000 0000 4000 0000 0000 0000 0000 0000 0000 0000 0800 0000 0000 0000 0000 0000 0000 0000  #...............8.......8.......@...............................
 3  000035e8: 3600 0000 0700 0000 0200 0000 0000 0000 7803 0000 0000 0000 7803 0000 0000 0000 2400 0000 0000 0000 0000 0000 0000 0000 0400 0000 0000 0000 0000 0000 0000 0000  6...............x.......x.......$...............................
 4  00003628: 4900 0000 0700 0000 0200 0000 0000 0000 9c03 0000 0000 0000 9c03 0000 0000 0000 2000 0000 0000 0000 0000 0000 0000 0000 0400 0000 0000 0000 0000 0000 0000 0000  I............................... ...............................
 5  00003668: 5700 0000 f6ff ff6f 0200 0000 0000 0000 c003 0000 0000 0000 c003 0000 0000 0000 1c00 0000 0000 0000 0600 0000 0000 0000 0800 0000 0000 0000 0000 0000 0000 0000  W......o........................................................
 6  000036a8: 6100 0000 0b00 0000 0200 0000 0000 0000 e003 0000 0000 0000 e003 0000 0000 0000 c000 0000 0000 0000 0700 0000 0100 0000 0800 0000 0000 0000 1800 0000 0000 0000  a...............................................................
 7  000036e8: 6900 0000 0300 0000 0200 0000 0000 0000 a004 0000 0000 0000 a004 0000 0000 0000 a800 0000 0000 0000 0000 0000 0000 0000 0100 0000 0000 0000 0000 0000 0000 0000  i...............................................................
 8  00003728: 7100 0000 ffff ff6f 0200 0000 0000 0000 4805 0000 0000 0000 4805 0000 0000 0000 1000 0000 0000 0000 0600 0000 0000 0000 0200 0000 0000 0000 0200 0000 0000 0000  q......o........H.......H.......................................
 9  00003768: 7e00 0000 feff ff6f 0200 0000 0000 0000 5805 0000 0000 0000 5805 0000 0000 0000 4000 0000 0000 0000 0700 0000 0100 0000 0800 0000 0000 0000 0000 0000 0000 0000  ~......o........X.......X.......@...............................
10  000037a8: 8d00 0000 0400 0000 0200 0000 0000 0000 9805 0000 0000 0000 9805 0000 0000 0000 c000 0000 0000 0000 0600 0000 0000 0000 0800 0000 0000 0000 1800 0000 0000 0000  ................................................................
11  000037e8: 9700 0000 0400 0000 4200 0000 0000 0000 5806 0000 0000 0000 5806 0000 0000 0000 3000 0000 0000 0000 0600 0000 1700 0000 0800 0000 0000 0000 1800 0000 0000 0000  ........B.......X.......X.......0...............................
12  00003828: a100 0000 0100 0000 0600 0000 0000 0000 0010 0000 0000 0000 0010 0000 0000 0000 1b00 0000 0000 0000 0000 0000 0000 0000 0400 0000 0000 0000 0000 0000 0000 0000  ................................................................
13  00003868: 9c00 0000 0100 0000 0600 0000 0000 0000 2010 0000 0000 0000 2010 0000 0000 0000 3000 0000 0000 0000 0000 0000 0000 0000 1000 0000 0000 0000 1000 0000 0000 0000  ................ ....... .......0...............................
14  000038a8: a700 0000 0100 0000 0600 0000 0000 0000 5010 0000 0000 0000 5010 0000 0000 0000 7101 0000 0000 0000 0000 0000 0000 0000 1000 0000 0000 0000 0000 0000 0000 0000  ................P.......P.......q...............................
15  000038e8: ad00 0000 0100 0000 0600 0000 0000 0000 c411 0000 0000 0000 c411 0000 0000 0000 0d00 0000 0000 0000 0000 0000 0000 0000 0400 0000 0000 0000 0000 0000 0000 0000  ................................................................
16  00003928: b300 0000 0100 0000 0200 0000 0000 0000 0020 0000 0000 0000 0020 0000 0000 0000 1100 0000 0000 0000 0000 0000 0000 0000 0400 0000 0000 0000 0000 0000 0000 0000  ................. ....... ......................................
17  00003968: bb00 0000 0100 0000 0200 0000 0000 0000 1420 0000 0000 0000 1420 0000 0000 0000 2400 0000 0000 0000 0000 0000 0000 0000 0400 0000 0000 0000 0000 0000 0000 0000  ................. ....... ......$...............................
18  000039a8: c900 0000 0100 0000 0200 0000 0000 0000 3820 0000 0000 0000 3820 0000 0000 0000 7c00 0000 0000 0000 0000 0000 0000 0000 0800 0000 0000 0000 0000 0000 0000 0000  ................8 ......8 ......|...............................
19  000039e8: d300 0000 0e00 0000 0300 0000 0000 0000 d03d 0000 0000 0000 d02d 0000 0000 0000 0800 0000 0000 0000 0000 0000 0000 0000 0800 0000 0000 0000 0800 0000 0000 0000  .................=.......-......................................
20  00003a28: df00 0000 0f00 0000 0300 0000 0000 0000 d83d 0000 0000 0000 d82d 0000 0000 0000 0800 0000 0000 0000 0000 0000 0000 0000 0800 0000 0000 0000 0800 0000 0000 0000  .................=.......-......................................
21  00003a68: eb00 0000 0600 0000 0300 0000 0000 0000 e03d 0000 0000 0000 e02d 0000 0000 0000 e001 0000 0000 0000 0700 0000 0000 0000 0800 0000 0000 0000 1000 0000 0000 0000  .................=.......-......................................
22  00003aa8: f400 0000 0100 0000 0300 0000 0000 0000 c03f 0000 0000 0000 c02f 0000 0000 0000 2800 0000 0000 0000 0000 0000 0000 0000 0800 0000 0000 0000 0800 0000 0000 0000  .................?......./......(...............................
23  00003ae8: f900 0000 0100 0000 0300 0000 0000 0000 e83f 0000 0000 0000 e82f 0000 0000 0000 2800 0000 0000 0000 0000 0000 0000 0000 0800 0000 0000 0000 0800 0000 0000 0000  .................?......./......(...............................
24  00003b28: 0201 0000 0100 0000 0300 0000 0000 0000 1040 0000 0000 0000 1030 0000 0000 0000 1d00 0000 0000 0000 0000 0000 0000 0000 0800 0000 0000 0000 0000 0000 0000 0000  .................@.......0......................................
25  00003b68: 0801 0000 0800 0000 0300 0000 0000 0000 2d40 0000 0000 0000 2d30 0000 0000 0000 0300 0000 0000 0000 0000 0000 0000 0000 0100 0000 0000 0000 0000 0000 0000 0000  ................-@......-0......................................
26  00003ba8: 0d01 0000 0100 0000 3000 0000 0000 0000 0000 0000 0000 0000 2d30 0000 0000 0000 1b00 0000 0000 0000 0000 0000 0000 0000 0100 0000 0000 0000 0100 0000 0000 0000  ........0...............-0......................................
27  00003be8: 0100 0000 0200 0000 0000 0000 0000 0000 0000 0000 0000 0000 4830 0000 0000 0000 7002 0000 0000 0000 1c00 0000 0600 0000 0800 0000 0000 0000 1800 0000 0000 0000  ........................H0......p...............................
28  00003c28: 0900 0000 0300 0000 0000 0000 0000 0000 0000 0000 0000 0000 b832 0000 0000 0000 5301 0000 0000 0000 0000 0000 0000 0000 0100 0000 0000 0000 0000 0000 0000 0000  .........................2......S...............................
29  00003c68: 1100 0000 0300 0000 0000 0000 0000 0000 0000 0000 0000 0000 0b34 0000 0000 0000 1601 0000 0000 0000 0000 0000 0000 0000 0100 0000 0000 0000 0000 0000 0000 0000  .........................4......................................
```

Now look back at the `readelf` output for this line

```
Section header string table index: 29
```


This gives the index for the string table which contains the names of all of the sections... Remember, `sh_name` member of section headers did not contained the actual name for the section but a index to section table. This is that section table.

On further analyzing this section table entry, we can identify everything about this section.


```
index |  offset   |  sh_name  |  sh_type  |        sh_flags     |        sh_addr      |       sh_offset     |       sh_size       |  sh_link  |  sh_info  |     sh_addralign    |       sh_entsize    |
29    | 00003c68: | 1100 0000 | 0300 0000 | 0000 0000 0000 0000 | 0000 0000 0000 0000 | 0b34 0000 0000 0000 | 1601 0000 0000 0000 | 0000 0000 | 0000 0000 | 0100 0000 0000 0000 | 0000 0000 0000 0000 |
```

Right now, interesting thing for us is the data that resides in this section. To get that, we need `sh_offset` and `sh_size`. *(Keep in mind that these values are in little endian form)*


```
❯ xxd \
    -s 0x340b \     # short for 0x000000000000340b (sh_offset)
    -l 0x116 \      # short for 0x0000000000000116 (sh_size)
    hello_world


0000340b: 002e 7379 6d74 6162 002e 7374 7274 6162  ..symtab..strtab
0000341b: 002e 7368 7374 7274 6162 002e 696e 7465  ..shstrtab..inte
0000342b: 7270 002e 6e6f 7465 2e67 6e75 2e70 726f  rp..note.gnu.pro
0000343b: 7065 7274 7900 2e6e 6f74 652e 676e 752e  perty..note.gnu.
0000344b: 6275 696c 642d 6964 002e 6e6f 7465 2e41  build-id..note.A
0000345b: 4249 2d74 6167 002e 676e 752e 6861 7368  BI-tag..gnu.hash
0000346b: 002e 6479 6e73 796d 002e 6479 6e73 7472  ..dynsym..dynstr
0000347b: 002e 676e 752e 7665 7273 696f 6e00 2e67  ..gnu.version..g
0000348b: 6e75 2e76 6572 7369 6f6e 5f72 002e 7265  nu.version_r..re
0000349b: 6c61 2e64 796e 002e 7265 6c61 2e70 6c74  la.dyn..rela.plt
000034ab: 002e 696e 6974 002e 7465 7874 002e 6669  ..init..text..fi
000034bb: 6e69 002e 726f 6461 7461 002e 6568 5f66  ni..rodata..eh_f
000034cb: 7261 6d65 5f68 6472 002e 6568 5f66 7261  rame_hdr..eh_fra
000034db: 6d65 002e 696e 6974 5f61 7272 6179 002e  me..init_array..
000034eb: 6669 6e69 5f61 7272 6179 002e 6479 6e61  fini_array..dyna
000034fb: 6d69 6300 2e67 6f74 002e 676f 742e 706c  mic..got..got.pl
0000350b: 7400 2e64 6174 6100 2e62 7373 002e 636f  t..data..bss..co
0000351b: 6d6d 656e 7400                           mment.

```


ASCII representation of this section's data chunk confirms that this must be **the** string table. (the one which contains the names of the sections). Now atleast we know how to walk through the headers and locate a string table section. This gives us a green signal to go deeper and learn more about string tables.

## String table

So, here's the deal: when you've got a bunch of characters, and you end them with a null character, that whole thing is what we call a "string." (*At least, that's what I've learned, and I'm sticking with it for now.*)


Now, when it comes to a string table, it's pretty simple. It's just a bunch of these strings all lined up, one after the other. The only twist is that the first string is always null (just a null char - `\0` - a null string). Now you can put all that data in a section and create a section header for it with **type** - `SHT_STRTAB`(which is just `0x3` in fancy lingo). And voila, you've got yourself a proper string table, with a section header entry for it.


If you want to picture it, think of it like this - a string table is like a list of strings, where the first one is always an empty string.


```
## Every 00 is a null char (in hex)
# For  Section=.shstrtab (Offset: 0x348b, Size: 278 = 0x116 in hex)

❯ xxd -s 0x340b -l 278 hello

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


It should be pretty easy to write a parser for this, if not, ask your friend to do it for you. (hint: not me)



Now, to proceed, let's take a look at the C program that I'll be using for further examples

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

I assume you can compile it and create the ELF binary. After the ELF binary is ready, analyze it to extract the list of all sections with a type of `0x3` (feeling fancy - `SHT_STRTAB`). Feel free to use `readelf`, `hexdump`, `xxd`, or any tool you prefer – the output should be same, regardless of your choice.

Using my pretty parser, I found three entries.

```
[ 07 ] Section Name: .dynstr        Type: 0x3       Flags: 0x2      Addr: 0x4a0     Offset: 0x4a0           Size: 170       Link: 0         Info: 0x0       Addralign: 0x1          Entsize: 0
[ 28 ] Section Name: .strtab        Type: 0x3       Flags: 0x0      Addr: 0x0       Offset: 0x3318          Size: 371       Link: 0         Info: 0x0       Addralign: 0x1          Entsize: 0
[ 29 ] Section Name: .shstrtab      Type: 0x3       Flags: 0x0      Addr: 0x0       Offset: 0x348b          Size: 278       Link: 0         Info: 0x0       Addralign: 0x1          Entsize: 0
```

Let's examine them closely, one by one.


**NOTE**: *String tables consist exclusively of strings. This data doesn't serve much purpose unless those strings are needed by other sections.*


### 1. `.shstrtab`

This is *the* string table (the one which stores the names of all of the sections) - *well we already talked about it so no point of repeating it, right?*

So, Why are the section names stored in a separate dedicated section, rather than directly within each section's `sh_name` member??

**Answer**: While I can't say for certain, it's possible that this design choice was made to accommodate variable-length section names. Storing the names in a separate section allows flexibility in the length of section names and avoids any size constraints related to the `sh_name` member.

When I parse this with my parser, the data of this section appears like this -- an offset in the section and the string stored at that offset.

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

This section contains strings (:P), mostly the ones representing names linked to symbol table entries (we'll talk about symbol tables later). But at a quick glance, you can spot some of the names for `variables` and `functions` we used in our C program, such as `global3`, `print_globals`, `main` and so on.

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
