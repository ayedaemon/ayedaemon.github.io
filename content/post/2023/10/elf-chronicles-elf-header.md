---
title: "ELF Chronicles: ELF file Header (1/?)"
date: 2023-10-18T13:34:57+05:30
draft: false
# showtoc: false
tags: ["C", "ELF", "RE"]
series:
  - "ELF Chronicles"
description: "Exploring ELF file headers"
# math: true
# ShowBreadCrumbs: false
# searchHidden: true
# hideMeta: true
---


## Hexdumps

In the fascinating world of computers, we're stuck conversing in binary, a rather dull language of just *ones* and *zeros*. But because we *mere* humans love things to be a tad more exciting and concise, we've come up with our own nifty number system - "`hexadecimal`" or "`hex`" for short. This system ditches the binary bore and adds a touch of flair with 16 snazzy symbols. It's got your usual digits from `0 to 9`, plus those fancy `A to F` letters to make data a bit more, well, *hexadecimal-chic*!


Now, let's take a gander at this binary enigma, a message that only the most extraordinary folks can decipher with ease:

```{linenos=false}
011010000110010101101100011011000110111100001010
```
For us ordinary humans, this is a bit like deciphering alien hieroglyphics. So, we follow a procedure to unravel the secrets hidden within.

Step one involves breaking down the binary data into byte-sized chunks, each containing 8 bits:

```{linenos=false}
01101000 01100101 01101100 01101100 01101111 00001010
```
Now, we embark on the magical journey of converting each chunk into its hexadecimal form. The legendary figures of the past might have used pen and paper, but in our tech-savvy era, we turn to tools like [CyberChef](https://cyberchef.org/#recipe=From_Binary('Space',8)To_Hex('Space',0)&input=MDExMDEwMDAgMDExMDAxMDEgMDExMDExMDAgMDExMDExMDAgMDExMDExMTEgMDAwMDEwMTA).

No matter your chosen method, the results remains the same:

```{linenos=false}
68 65 6c 6c 6f 0a
```

The binary code's cryptic riddle got a facelift, and voilà! We now have this friendly hexadecimal version. It's just what the doctor ordered for us humans to have a casual chat with the binary data, no sweat!


## From Code to Binary

Lets's go on a journey that turns elegant C code into a mysterious binary blob, a language of ones and zeros that only computers understand. (** *coughs compilation* **)

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


After compiling the above C code we get an ELF (Executable and Linkable Format) file. (Compilation command - `gcc hello_world.c -o hello_world`)

This generated file, at its core, is nothing more than a delightful binary blob. It's the computer's secret handshake, speaking directly in ones and zeros, no pleasantries. And the icing on the cake is that we *mere* humans, with our clever programming prowess, can craft tools to translate this binary jargon into friendly hexadecimal, or we can simply cozy up to good ol' `hexdump` and `xxd` for the job. Whichever suits your fancy, we've got options!

## ELF Header

Here's a snapshot of the first 64 bytes in the compiled binary file:

```{linenos=false}
# In binary representation
❯ xxd -b -l 64 ./hello_world
00000000: 01111111 01000101 01001100 01000110 00000010 00000001  .ELF..
00000006: 00000001 00000000 00000000 00000000 00000000 00000000  ......
0000000c: 00000000 00000000 00000000 00000000 00000011 00000000  ......
00000012: 00111110 00000000 00000001 00000000 00000000 00000000  >.....
00000018: 01010000 00010000 00000000 00000000 00000000 00000000  P.....
0000001e: 00000000 00000000 01000000 00000000 00000000 00000000  ..@...
00000024: 00000000 00000000 00000000 00000000 00101000 00110101  ....(5
0000002a: 00000000 00000000 00000000 00000000 00000000 00000000  ......
00000030: 00000000 00000000 00000000 00000000 01000000 00000000  ....@.
00000036: 00111000 00000000 00001101 00000000 01000000 00000000  8...@.
0000003c: 00011110 00000000 00011101 00000000                    ....

# In hex representation
❯ xxd -l 64 ./hello_world
00000000: 7f45 4c46 0201 0100 0000 0000 0000 0000  .ELF............
00000010: 0300 3e00 0100 0000 5010 0000 0000 0000  ..>.....P.......
00000020: 4000 0000 0000 0000 2835 0000 0000 0000  @.......(5......
00000030: 0000 0000 4000 3800 0d00 4000 1e00 1d00  ....@.8...@.....
```

Now, you may wonder, what on earth does this mean? Well, these intriguing bytes are like puzzle pieces, and depending on the machine type, they map to specific structures in the Linux kernel. Our quest, quite simply, is to unravel this digital enigma and shed light on the code's purpose.

```c{linenos=false}
/*
https://elixir.bootlin.com/linux/v6.5.7/source/include/uapi/linux/elf.h#L207
*/

#define EI_NIDENT	16

typedef struct elf32_hdr {
  unsigned char	e_ident[EI_NIDENT];
  Elf32_Half	e_type;
  Elf32_Half	e_machine;
  Elf32_Word	e_version;
  Elf32_Addr	e_entry;  /* Entry point */
  Elf32_Off	    e_phoff;
  Elf32_Off	    e_shoff;
  Elf32_Word	e_flags;
  Elf32_Half	e_ehsize;
  Elf32_Half	e_phentsize;
  Elf32_Half	e_phnum;
  Elf32_Half	e_shentsize;
  Elf32_Half	e_shnum;
  Elf32_Half	e_shstrndx;
} Elf32_Ehdr;

typedef struct elf64_hdr {
  unsigned char	e_ident[EI_NIDENT];	/* ELF "magic number" */
  Elf64_Half    e_type;
  Elf64_Half    e_machine;
  Elf64_Word    e_version;
  Elf64_Addr    e_entry;		/* Entry point virtual address */
  Elf64_Off     e_phoff;		/* Program header table file offset */
  Elf64_Off     e_shoff;		/* Section header table file offset */
  Elf64_Word    e_flags;
  Elf64_Half    e_ehsize;
  Elf64_Half    e_phentsize;
  Elf64_Half    e_phnum;
  Elf64_Half    e_shentsize;
  Elf64_Half    e_shnum;
  Elf64_Half    e_shstrndx;
} Elf64_Ehdr;
```

Since I'm on a 64 bit system, I'll use `Elf64_Ehdr` to show what each byte in the above data chunk represents.


```{linenos=false}
❯ xxd -l 64 ./hello_world
00000000: 7f45 4c46 0201 0100 0000 0000 0000 0000  .ELF............
00000010: 0300 3e00 0100 0000 5010 0000 0000 0000  ..>.....P.......
00000020: 4000 0000 0000 0000 2835 0000 0000 0000  @.......(5......
00000030: 0000 0000 4000 3800 0d00 4000 1e00 1d00  ....@.8...@.....


// After mapping the linux ELF struct to the above data

e_ident[16] = 7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00
e_type      = 03 00
e_machine   = 3e 00
e_version   = 01 00 00 00
e_entry     = 50 10 00 00 00 00 00 00
e_phoff     = 40 00 00 00 00 00 00 00
e_shoff     = 28 35 00 00 00 00 00 00
e_flags     = 00 00 00 00
e_ehsize    = 40 00
e_phentsize = 38 00
e_phnum     = 0d 00
e_shentsize = 40 00
e_shnum     = 1e 00
e_shstrndx  = 1d 00
```

Shall we dissect each of these mysterious members in the struct?

![](https://media.giphy.com/media/QaNQJZhjd2QrDUBNcg/giphy.gif#center)




### 1. **e_ident[EI_NIDENT]**

The first 16 bytes of the ELF header are collectively referred to as the "ident" or "identification" field. It includes a magic number and various identification information. Here is a table that tells more about what all identification information is present in it.

```{linenos=false}
e_ident[16] = 7f45 4c46 0201 0100 0000 0000 0000 0000
    EI_MAG0 = 7f
    EI_MAG1 = 45 (E)
    EI_MAG2 = 4c (L)
    EI_MAG3 = 46 (F)
    EI_CLASS = 02
    EI_DATA = 01
    EI_VERSION = 01
    EI_OSABI = 00
    EI_ABIVERSION = 00
    EI_PAD = 00 0000 0000 0000
```

Ah, you might wonder, "How on earth do I know this?" Well, my friend, it's a detective game we play, and our magnifying glass is the kernel source code.


```{linenos=false}
/*
https://elixir.bootlin.com/linux/v6.5.7/source/include/uapi/linux/elf.h#L334
*/

#define	EI_MAG0		0		/* e_ident[] indexes */
#define	EI_MAG1		1
#define	EI_MAG2		2
#define	EI_MAG3		3
#define	EI_CLASS	4       /* 1=32Bit; 2=64Bit */
#define	EI_DATA		5       /* Endianness ==> 1=Little; 2=Big */
#define	EI_VERSION	6       /* ELF header version */
#define	EI_OSABI	7       /* OS ABI ==> 0=None(same as SysV); 3=Linux */
#define	EI_PAD		8       /* Starting of padding - currently unused */
```


**==>** This information tells me that my ELF binary is a `64-Bit` (EI_CLASS = 02), `Little` endian (EI_DATA = 01) binary.








### 2. **e_type**

This member tells what type of ELF file it is.

```{linenos=false}
/*
https://elixir.bootlin.com/linux/v6.5.7/source/include/uapi/linux/elf.h#L69
*/

#define ET_NONE   0         // No file type
#define ET_REL    1
#define ET_EXEC   2
#define ET_DYN    3
#define ET_CORE   4
#define ET_LOPROC 0xff00    // Processor-specific
#define ET_HIPROC 0xffff    // Processor-specific
```

Since my binary is little endian, `e_type = 03 00` should be read as `e_type = 00 03`. That tells me that I've a `ET_DYN` type of file.








### 3. **e_machine**

This member tells us about the target architecture for the file. In linux kernel uapi, there is [a complete header file dedicated for target machines](https://elixir.bootlin.com/linux/v6.5.7/source/include/uapi/linux/elf-em.h).

For my binary file, machine type is `3e` (e_machine = *3e 00*; Should be read as *00 3e*).

*(Integer representation of `3e` is `62`)*

```{linenos=false}
/*
https://elixir.bootlin.com/linux/v6.5.7/source/include/uapi/linux/elf-em.h#L31
*/

#define EM_X86_64	62	/* AMD x86-64 */
```

### 4. **e_version**

This member specifies the version of the ELF file. This is different from the `EI_VERSION` which tells only about the ELF header version.

For my binary file, version is `1` (remember, to convert the value to little endian)

These are the versions defined in linux kernel uapi header

```{linenos=false}
#define EV_NONE		0		/* e_version, EI_VERSION */
#define EV_CURRENT	1
#define EV_NUM		2
```



### 5. **e_entry**

This member is quite interesting. This tells about the virtual/memory address where program execution begins. This is the starting point of the program.

You might think, "Aha, this must always point to the `main()` function!" Well, here's a plot twist for you!

For my binary file, the entry point is `1050` (e_entry = 50 10 00 00 00 00 00 00).


According to our trusty `objdump`, this value does not point to the `main` function but points to the `_start` function. *(..which in turn executes the `main` function. Here is [an article](https://ayedaemon.github.io/post/2022/01/debugging-c-code/#the-whole-picture) that explains this.)*

```{linenos=false}
❯ objdump  -D --disassembler-options=intel hello_world | grep -i "1050"

0000000000001050 <_start>:
```



### 6. **e_phoff**

This is the program header offset. The starting point in the ELF file where program headers can be found.

### 7. **e_shoff**

Just like `e_phoff`, this member stores the offset of the section headers of the ELF file.

### 8. **e_flags**

This member provides processor-specific flags associated with the file.

### 9. **e_ehsize**

This member tells the size of the the ELF header. For my binary, value of this member is `40` (64 in decimal). Now you take a guess why I started analyzing first 64 bytes of the file.

### 10. **e_phentsize**

This is the size of each entry in program header.

### 11. **e_phnum**

This is the count of entries in program header


### 12. **e_shentsize**

This is the size of each entry in section header.

### 13. **e_shnum**

This is the count of entries in section header

### 14. **e_shstrndx**

Now, this little guy is what we call the "Section string index". This points to the index in section headers which holds all of the strings.


*(We'll talk more about section headers and program headers in later articles.)*


## Practicals

### How to edit a binary file?

If you think it through, you just need a program that can read/write binary data and convert that data to hex for us to view. You can build your own tool to do this or you can use other tools that can already do this.

I would like to propose my favorite - `vim` + `xxd`

Here are the steps to it.

- Open the file in vim in binary mode (use `-b` flag)

```{linenos=false}
vim -b argv_printer
```

- Pass the data to `xxd` (you can also use the additional flags that xxd supports)
  - Press `:` to go into commmand mode
  - then type `%!xxd -c 1` to pass the binary data through this command.

```{linenos=false}
:%!xxd -c 1
```

- Edit the hex values you want (just like you would edit any other text file, press `i` and go on)
- Reverse the hex to binary
  - Go to command mode again by pressing `:`
  - then type `%!xxd -r`
```{linenos=false}
:%!xxd -r
```

- Now save and quit the vim editor
  - If you don't know steps for that consider learning vim first
  - or, use another hex editor


### Change the ELF magic number

- Open the file with vim and edit the `EI_MAG` part.

```{linenos=false}
# Before
  00000000: 7f  .
  00000001: 45  E
  00000002: 4c  L
  00000003: 46  F

# After
  00000000: 7f  .
  00000001: 48  E
  00000002: 45  L
  00000003: 58  F
```

*Note that I've only changed the hex values and not the ascii values for it.*

- revert the hex to binary data (`:%!xxd -r`)
- write and quit vim (I'm still not telling you the command)
- analyze it

```{linenos=false}
❯ ./hello_world
zsh: exec format error: ./hello_world


❯ readelf --file-header --wide hello_world
readelf: Error: Not an ELF file - it has the wrong magic bytes at the start
```


The reason for this behaviour is written in kernel code.

```c{linenos=false}
/*
https://elixir.bootlin.com/linux/v6.5.7/source/include/uapi/linux/elf.h#L348
*/
#define	ELFMAG		"\177ELF"
#define	SELFMAG		4

/*
https://elixir.bootlin.com/linux/v6.5.7/source/fs/binfmt_elf.c#L848
*/
retval = -ENOEXEC;
if (memcmp(elf_ex->e_ident, ELFMAG, SELFMAG) != 0)
		goto out;
```



### Change the executable class (64 bit -> 32 bit)


- Open the file with vim and edit the `EI_CLASS` part.

```{linenos=false}
# Before
  00000004: 02  .

# After
  00000004: 01  .
```

- revert the hex to binary data (`:%!xxd -r`)
- write and quit vim (I'm still not telling you the command)
- analyze it

```{linenos=false}
# Runs perfectly fine
❯ ./hello_world
Hello World1
Hello World2
Hello World3


# file command tells another tale
❯ file hello_world
hello_world: ELF 32-bit LSB pie executable, x86-64, version 1 (SYSV), no program header, no section header
```

This is clearly a parsing problem. There are no checks on the kernel for the `EI_CLASS` (or I should say I could not find any, if you find one, please let me know.)


### ...more (DIY, kind of)

There are few more interesting things you can play around with
- `EI_OSABI`
- `e_machine`
- `e_entry`


## Conclusion

ELF headers emerge as the silent orchestrators of the executable files... The backstage bosses of the show. In this article, we cracked open their secrets (with not-so-real-world tricks) and diving into their nitty-gritty using hexdumps. Think of this as the cool architect of the software world, shaping how things work under the hood.

Mastering these headers is like getting a backstage pass to rock the binary world - tweaking, fixing, and making stuff dance to your tune. So next time you run an executable on *unix machines, remember, ELF header are the groove makers behind the scenes!

---

## Useful links

1. (ELF Specification 1.1) https://flint.cs.yale.edu/cs422/doc/ELF_Format.pdf
