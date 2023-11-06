---
title: "Fun with Master Boot Record"
date: 2022-09-28T01:10:18+05:30
draft: false
# showtoc: false
tags: [linux, mbr, bootloader]
# series: []
description: Analysis of bootloader and partition table; And some fun with Master Boot Record
# math: true
# ShowBreadCrumbs: false
# searchHidden: true
# hideMeta: true
---

Malware has been used numerous times by attackers to destroy a computer's Master Boot Record, rendering it inoperable. By erasing the MBR, the machine is unable to load the operating system. There is no easy way to rewrite the Master Boot Record into place without an operating system, and the machine becomes completely useless and unrecoverable. In addition, many ransomwares infect the master boot record by overwriting it with malicious code. The system is then automatically restarted to allow the infection to take place. When the system restarts, the user is locked out, and the ransomware displays a note demanding payment. Simple money!

To understand how all of this is possible, and how an attacker can achieve it, we must first understand the MBR and the process of its execution. 

## The boot process

The booting procedure of a system has become simpler over time, but this does not always imply that it is any easier. Every computer, big or small, goes through a start-up procedure known as the "Boot" process. Because different types of hardware operate in different ways, the boot procedure is heavily influenced by the type of CPU architecture and other hardware components.

To avoid confusion, I won't go into great detail about each stage of the booting process. However, a typical linux booting procedure involves the following phases at a higher level: 

1. **Power Up**
	
	This is the step where you press the power button. This triggers the [BIOS](https://en.wikipedia.org/wiki/BIOS) [^wiki_bios] from motherboard's flash memory to start executing it's functions.

    [^wiki_bios]: https://en.wikipedia.org/wiki/BIOS

2. **Power On Self Test**
	
	After BIOS is up and running, it initiates a quick self test to know if all the required hardware components are in working condition.

3. **Find a boot device**
	
	This step finds all the bootable devices from the earlier detected hard drives. The way this works is by checking the MBR (Master Boot Record) for each detected devices. MBR is refered to the first 512 bytes of any bootable device.

4. **Load the MBR**
	
	MBR is the first 512 bytes. These 512 bytes contains a bootloader, partition table and the magic number. This is loaded into ram and is responsible to read data from drives and start the operating system.
	
5. **Load GRUB**
	
	This is a boot loader program which works in 2 stages. First stage is a small machine code binary on MBR. Its sole job is to locate the second stage boot loader and load it in memory. Once the second stage boot loader is in the memory, it presents the user with a graphical screen showing the different operating systems to choose from.
	
6. **Kernel**

	The above OS selection decides what kernel and optional initramfs is to be loaded into memory. The kernel then initializes and configures the computer's memory and configures the various hardware attached to the system, including all the I/O subsystems. After some more operations, the kernel is completely loaded into memory and is operational. It's time to set up the user environment.

7. **init**

	This is the first userspace program that is started by kernel. Now this starts and manages all the userspace processes like your web browser, file manager, web servers, etc.


## MBR and other little things

Now that we are aware of how the boot procedure works, we can go on to the article's main objective, the [Master Boot Record](https://en.wikipedia.org/wiki/Master_boot_record). (but not this [Master Boot Record](https://www.youtube.com/c/MasterBootRecord)) 

If you're not already aware, this is how a typical hard drive appears from the outside.

![source: reddit](https://external-preview.redd.it/MNhQwTRxl_vXaEKZikrhbSz0A_vSK0zMzRFH5g7ZvOM.jpg?auto=webp&s=7ac34d2a3701d51967d02fab59eea9bb2db7f1da#center)

There are numerous components inside this small semi-metallic box that aid in its proper operation.

![source: researchgate.net](https://www.researchgate.net/profile/Mostafa-Sabbaghi/publication/317472565/figure/fig2/AS:613957429645317@1523390266281/Intact-HDD-with-the-cover-removed-left-and-exploded-image-of-a-HDD-showing-individual.png#center)

But we don't need to know about all of these components; instead, we'll concentrate on the disc-like structure in the centre. This is known as a **platter**. A platter is a single recording disc. A hard disc drive may have one or more platters.

![source: safemode.com](https://safemode.com.au/wp-content/uploads/2020/01/hard-drive-parts-min.png#center)

Each platter is divided into several circular tracks, and each track is further divided into several sectors. Each sector on a hard disc drive typically stores 512 bytes of user-accessible data. 

![source: quora](https://qph.fs.quoracdn.net/main-qimg-0763e88136c511a9f93c49ce3e84360f#center)

The first 512 bytes (or first sector) of a hard drive is where the MBR is located. And since everything in Linux is a "file", if we want to extract MBR data, all we have to do is to read the first 512 bytes of our bootable hard disk file and then write that content to another local file for further analysis. In most of the linux platforms, we can do this by [`dd`](https://www.man7.org/linux/man-pages/man1/dd.1.html) [^man_dd] command.

[^man_dd]: https://www.man7.org/linux/man-pages/man1/dd.1.html

```
dd if=/dev/sda of=mbr.sample bs=512 count=1
```

The above command will read a 512-byte block (once) from `/dev/sda` and save it in the `mbr.sample` file. Then we can the check the type of this file using `file` command. 

```bash 
file mbr.sample

## Output
# mbr.sample: x86 boot sector; partition 1: ID=0x83, active, starthead 32, startsector 2048, 2097152 sectors; partition 2: ID=0x8e, starthead 170, startsector 2099200, 41191424 sectors, code offset 0x63
```

An x86 boot sector is recognised in this file. Interestingly, it also lists the start head, start sector, total number of sectors, offset, and IDs of all the partitions. This was sufficient reason for me to dig up the file's hexdump and understand how `file` command is able to gather all this information.

```bash
hexdump -C mbr.sample
```

Output:-

```txt
00000000  eb 63 90 10 8e d0 bc 00  b0 b8 00 00 8e d8 8e c0  |.c..............|
00000010  fb be 00 7c bf 00 06 b9  00 02 f3 a4 ea 21 06 00  |...|.........!..|
00000020  00 be be 07 38 04 75 0b  83 c6 10 81 fe fe 07 75  |....8.u........u|
00000030  f3 eb 16 b4 02 b0 01 bb  00 7c b2 80 8a 74 01 8b  |.........|...t..|
00000040  4c 02 cd 13 ea 00 7c 00  00 eb fe 00 00 00 00 00  |L.....|.........|
00000050  00 00 00 00 00 00 00 00  00 00 00 80 01 00 00 00  |................|
00000060  00 00 00 00 ff fa 90 90  f6 c2 80 74 05 f6 c2 70  |...........t...p|
00000070  74 02 b2 80 ea 79 7c 00  00 31 c0 8e d8 8e d0 bc  |t....y|..1......|
00000080  00 20 fb a0 64 7c 3c ff  74 02 88 c2 52 be 05 7c  |. ..d|<.t...R..||
00000090  b4 41 bb aa 55 cd 13 5a  52 72 3d 81 fb 55 aa 75  |.A..U..ZRr=..U.u|
000000a0  37 83 e1 01 74 32 31 c0  89 44 04 40 88 44 ff 89  |7...t21..D.@.D..|
000000b0  44 02 c7 04 10 00 66 8b  1e 5c 7c 66 89 5c 08 66  |D.....f..\|f.\.f|
000000c0  8b 1e 60 7c 66 89 5c 0c  c7 44 06 00 70 b4 42 cd  |..`|f.\..D..p.B.|
000000d0  13 72 05 bb 00 70 eb 76  b4 08 cd 13 73 0d 5a 84  |.r...p.v....s.Z.|
000000e0  d2 0f 83 de 00 be 85 7d  e9 82 00 66 0f b6 c6 88  |.......}...f....|
000000f0  64 ff 40 66 89 44 04 0f  b6 d1 c1 e2 02 88 e8 88  |d.@f.D..........|
00000100  f4 40 89 44 08 0f b6 c2  c0 e8 02 66 89 04 66 a1  |.@.D.......f..f.|
00000110  60 7c 66 09 c0 75 4e 66  a1 5c 7c 66 31 d2 66 f7  |`|f..uNf.\|f1.f.|
00000120  34 88 d1 31 d2 66 f7 74  04 3b 44 08 7d 37 fe c1  |4..1.f.t.;D.}7..|
00000130  88 c5 30 c0 c1 e8 02 08  c1 88 d0 5a 88 c6 bb 00  |..0........Z....|
00000140  70 8e c3 31 db b8 01 02  cd 13 72 1e 8c c3 60 1e  |p..1......r...`.|
00000150  b9 00 01 8e db 31 f6 bf  00 80 8e c6 fc f3 a5 1f  |.....1..........|
00000160  61 ff 26 5a 7c be 80 7d  eb 03 be 8f 7d e8 34 00  |a.&Z|..}....}.4.|
00000170  be 94 7d e8 2e 00 cd 18  eb fe 47 52 55 42 20 00  |..}.......GRUB .|
00000180  47 65 6f 6d 00 48 61 72  64 20 44 69 73 6b 00 52  |Geom.Hard Disk.R|
00000190  65 61 64 00 20 45 72 72  6f 72 0d 0a 00 bb 01 00  |ead. Error......|
000001a0  b4 0e cd 10 ac 3c 00 75  f4 c3 00 00 00 00 00 00  |.....<.u........|
000001b0  00 00 00 00 00 00 00 00  70 7e 04 00 00 00 80 20  |........p~..... |
000001c0  21 00 83 aa 28 82 00 08  00 00 00 00 20 00 00 aa  |!...(....... ...|
000001d0  29 82 8e fe ff ff 00 08  20 00 00 88 74 02 00 00  |)....... ...t...|
000001e0  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
000001f0  00 00 00 00 00 00 00 00  00 00 00 00 00 00 55 aa  |..............U.|
00000200
```

![](https://media.giphy.com/media/3o6MbbwX2g2GA4MUus/giphy.gif#center)


### Dissecting the Master Boot Record

Without any knowledge of layout, simply looking at the `hexdump` output is not particularly helpful. Therefore, it is now necessary to understand the MBR layout. 

MBR consists of 3 parts - **bootloader**, **partition table**, and **magic number**.

The magic number is found in the final two bytes, as opposed to the regular userspace files. It is `55AA` in the file, but be mindful of the processor's endianness. Since my CPU is little endian, the leftmost bytes are read first. As a result, `AA55` will become the magic number.

After that is subtracted, we are left with `512-2 = 510` bytes. Out of these, the bootloader is stored in the first `446` bytes, and the partition tables are stored in the remaining `64` bytes. To evaluate these components separately, let's extract them into distinct files using the same old `dd` command.

```bash
## Bootlaoder
dd if=mbr.sample of=mbr.bootloader bs=1 count=446

## Partition table (skip first 446 bytes)
dd if=mbr.sample of=mbr.partition_table bs=1 count=64 skip=446

## magic (skip first 510 bytes)
dd if=mbr.sample of=mbr.magic bs=1 count=2 skip=510

## Check file types
file *
## Output
# mbr.bootloader:      data
# mbr.magic:           BIOS (ia32) ROM Ext.
# mbr.partition_table: 8086 relocatable (Microsoft)
# mbr.sample:          x86 boot sector; partition 1: ID=0x83, active, starthead 32, startsector 2048, 2097152 sectors; partition 2: ID=0x8e, starthead 170, startsector 2099200, 41191424 sectors, code offset 0x63
```

It's fantastic that the file command can recognise each of these MBR components separately. Now with separate files, we can carefully examine the partition table and determine what data it can give us. 


```bash
hexdump -C -v  mbr.partition_table
```
Output:-
```txt
00000000  80 20 21 00 83 aa 28 82  00 08 00 00 00 00 20 00  |. !...(....... .|
00000010  00 aa 29 82 8e fe ff ff  00 08 20 00 00 88 74 02  |..)....... ...t.|
00000020  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
00000030  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
```

The MBR system only supports 4 primary partitions since partition tables actually only contain 4 records. We must divide a primary partition into smaller partitions and keep a separate partition table inside of that primary partition if we want to construct more than four partitions. The term "Expanded partitions" is in fact used to describe these extended partitions. We can see from the result above that there are a total of 64 bytes, giving us a total of `64/4 = 16` bytes for each record. Let's understand the layout of these 16 bytes and then we can analyze the partition table data using `hexdump`.

|Size (in bytes) | Purpose|
|----------------|--------|
|	1   | Boot indicator (0x80 for active and 0x00 for inactive) |
|  1 	|	partition start: head
|  1 	|	partition start: sector
|  1 	|	partition start: cylinder
|  1 	|	Partition ID
|  1 	|	partition end: head
|  1 	|	partition end: sector
|  1 	|	partition end: cylinder 
|  4 	|	Number of sectors before the beginning of this partition (sectors_before)
|  4 	|	Number of sectors in this partition (number_of_sectors)


There are 16 bytes in all of that. Now we know where the information that the `file` command was displaying previously comes from. 

Based on the information we have now, we can figure out few things on our own... 

```txt
00000000  80 20 21 00 83 aa 28 82  00 08 00 00 00 00 20 00  |. !...(....... .|
00000010  00 aa 29 82 8e fe ff ff  00 08 20 00 00 88 74 02  |..)....... ...t.|
00000020  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
00000030  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
```

For example, this disk only has 2 partitions because the final 2 records are all zeros. Due to the '0x80' byte in the first records, the first partition is bootable. And the ID for that partition is `83`. While the second partition is non bootable partition, and the ID of that partition is `82`. If you want, you can even calculate the size of each partition with the help of other information present in these records.

We are now down to the first `446` bytes, which include the bootloader.
The bootloader is simply a software that reads and loads other applications from the bootable partition.GRUB typically loads the second stage of itself from disk, however this is not a condition.There are bootloaders that load the kernel directly into memory or, even better, some of them are [full-fledged application that just works](https://forum.osdev.org/viewtopic.php?f=2&t=18763). [^osdev_512]

[^osdev_512]: https://forum.osdev.org/viewtopic.php?f=2&t=18763

*Note:- Although I won't be discussing it today, you can use the `ndisasm` disassembler to disassemble the bootloader image.This will require for some knowledge of the interrupts and memory management in the BIOS, which is outside the scope of this blog.*

That settles it; now that we are aware of what is contained within an MBR, why don't we attempt to construct one? 

![](https://media.giphy.com/media/R459x856IfF6w/giphy.gif#center)

## Creating your own bootloader

To start, we'll make a simple raw binary file and put `AA55` in it. Keep in mind that this is the **magic number** that belongs in an MBR. 

```asm
dw 0xAA55
```

Save this file as `custom bootloader.asm`. After compiling it with the `nasm` compiler, the results should look like this. 

```bash
## Compile custom_bootloader.asm
nasm -fbin custom_bootloader.asm -o custom_bootloader.bin


## Check the file type
file rhel_magic_number custom_bootloader.bin

## Output
# rhel_magic_number:     ISO-8859 text, with no line terminators
# custom_bootloader.bin: ISO-8859 text, with no line terminators


## Check hexdump
hexdump -C custom_bootloader.bin

## Ouput
# 00000000  55 aa                                             |U.|
# 00000002
```


We now have a 2 byte file containing the magic number. However, because the MBR is 512 bytes long, we must fill 510 more bytes. For the time being, let's just fill it with zeros and see if it's a valid MBR file. 

```asm
times 510 db 0
dw 0xAA55
```

The above code will write `0` 510 times and then write `AA55`.

```bash
## Compile custom_bootloader.asm
nasm -fbin custom_bootloader.asm -o custom_bootloader.bin


## Check the file type
file rhel_mbr custom_bootloader.bin

## Ouput
# rhel_mbr:              DOS/MBR boot sector
# custom_bootloader.bin: DOS/MBR boot sector


## Check hexdump
hexdump -C custom_bootloader.bin

## Ouput
# 00000000  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
# *
# 000001f0  00 00 00 00 00 00 00 00  00 00 00 00 00 00 55 aa  |..............U.|
# 00000200
```

This is, as expected, a valid MBR file with no information about the partition table or the bootloader. Can we, however, use this to boot the system?

Let's make an attempt.

I first tried without any mbr data to see what errors I would get when it fails. 
```bash
qemu-system-x86_64
```

And, as I was expecting, it said "no bootable device." 

![](https://i.imgur.com/wMCYSoo.png#center)

Let's run the test again, but this time with the MBR file we made. 

![](https://i.imgur.com/DJS9nN2.png#center)

It did not give me the error this time. That must imply that our MBR is functional. Since it lacks bootloader code, it does nothing. However, it is not returning the same previous error.

We can now add new instructions to our assembly file. However, we must keep in mind that we do not exceed the file's 512-byte limit. That means we'll have to take care of the zeros we're padding with. Because this is a very simple problem, there are special characters that can assist us in calculating the memory address of the beginning of the file and the current address in the file. 

```asm
times 510-($-$$) db 0       ;$ - Start addr; $$ - current addr
dw 0xAA55
```

We can calculate the exact number of zeros required for padding using these special characters. Let's compile it and put it to the test. 

```
nasm -fbin custom_bootloader.asm -o custom_bootloader.bin

qemu-system-x86_64 custom_bootloader.bin
```

This produces the same results as before, and the output file size remains 512 bytes. Let's add some more instructions to help us write some text on the screen.

Unlike userspace and kernelspace programs, we do not have any helper functions that can take a string and automatically print it to the screen. We'll have to tell the BIOS to do what we want here. And the only way I'm aware of is through **interrupts**. It is the same facility that operating systems and application programmes use to access BIOS functions. 

[Here](https://en.wikipedia.org/wiki/BIOS_interrupt_call#Interrupt_table) is a list of common BIOS interrupts. Not all BIOS (especially older ones) support all of these interrupts. The basic idea of using interrupts is we place proper values in some specific registers, and then trigger the interrupt. The interrupt routine will then fetch the values from those registers and based on that, it'll perform some action.

Anyway, using the above table, I determined that we needed to use interrupt vector `10h` (or 0x10) with interrupt vector `03h` (or 0x03) in `AH` register. Consider it as invoking the 10h function with the parameter value 03h. This returns the cursor's current position and shape. 

```
mov ah, 0x03;
int 10h

times 510-($-$$) db 0
dw 0xAA55
```

We can see that some initial bytes are written to the binary file after compiling and inspecting the hexdump.... And, thanks to `$` and `$$`, the file size remains 512 bytes. 

```bash
# Compiling the binary
nasm -fbin custom_bootloader.asm -o custom_bootloader.bin

## Checking hexdump
hexdump -C custom_bootloader.bin

## Output
# 00000000  b8 03 00 cd 10 00 00 00  00 00 00 00 00 00 00 00  |................|
# 00000010  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
# *
# 000001f0  00 00 00 00 00 00 00 00  00 00 00 00 00 00 55 aa  |..............U.|
# 00000200
```

This switches the BIOS to TTY mode, allowing me to print characters using the same interrupt `10h` but with a different value in the `ah` register. 

```
mov ah, 0x03
int 10h

mov ah, 0xE
mov al, 'H'
int 0x10

mov al, 'E'
int 0x10

mov al, 'Y'
int 0x10

times 510-($-$$) db 0
dw 0xAA55
```


When we compile and run this with qemu, we get the message "HEY" printed on the screen.


![](https://i.imgur.com/9YAcZ03.png#center)

Now that we know how to write characters on the screen, let's make a string and loop through it until the end, printing each character on the screen one by one using the same interrupt combination. 

```
; Setup TTY mode
mov ax, 0x03
int 10h


mov si, msg      ; si register now points to msg
mov ah, 0Eh      ; Use write function from 10h interrupt

.loop:
    lodsb          ; load first char from msg and point to next char
    or al, al      ; Check if end of string
    jz halt        ; if end of string, jump to halt
    int 10h        ; else, print char via interrupt
    jmp .loop      ; loop

halt:

msg: db "Hack the world", 0

times 510-($-$$) db 0
dw 0xAA55
```


Unfortunately, testing the above code does not produce the desired results, but instead produces some garbage values. 

![](https://i.imgur.com/LUbZu5D.png#center)

Further investigation revealed that our bootloader in memory is not properly aligned. This led me down another rabbit hole, this time about how the contents of the computer's physical memory look when the BIOS jumps to my bootloader code. Here is a [dedicated page](https://wiki.osdev.org/Memory_Map_(x86)) on the same topic here which covers a lot of details about it.

For us, we need to add a few more instructions to our code to properly align it. Finally, our code will look like this.

```
bits 16      ; BIOS works in 16 bit mode
org 0x7c00   ; MBR is loaded at 0x7c00 memory location

mov ax, 0x03
int 10h


mov si, msg
mov ah, 0Eh

.loop:
    lodsb
    or al, al
    jz halt
    int 10h
    jmp .loop

halt:
cli     ; disable further interrupts
hlt     ; halt

msg: db "Hack the world!!", 0

times 510-($-$$) db 0
dw 0xAA55
```

This time we get the desired result after compiling and testing the above. 
![](https://i.imgur.com/6eV40TA.png#center)


We successfully created a bootloader that prints some message on the screen. 


## Conclusions

We know that a MBR sector is comprised of 3 parts:
- bootloader (446 bytes)
- partition table (64 bytes)
- magic number (2 bytes)

And each component can be extracted separately and treated as a regular binary file. This means that we can create backups of only partition tables if necessary. Alternatively, we can **replace the bootloader code** with another code without affecting the partition table.(*Obviously for fun; like a friendly joke, nothing malicious*) 😈 😈

![](https://imgs.xkcd.com/comics/fight.png#center)

We know our above "Hack the World!!" code does not use all 510 bytes, so why not shrink it a little to fit in 446 bytes? This way we can protect the original partition table.


```asm
bits 16      
org 0x7c00   

mov ax, 0x03
int 10h


mov si, msg
mov ah, 0Eh

.loop:
    lodsb
    or al, al
    jz halt
    int 10h
    jmp .loop

halt:
cli     
hlt    

msg: db "Hack the world!!", 0

times 446-($-$$) db 0         ; Just change 510 to 446 :)
dw 0xAA55
```

This will generate the raw data file containing the bootloader program, which we can quickly test in a virtual machine.

```ruby
Vagrant.configure("2") do |config|
  config.vm.box = "archlinux/archlinux"
  config.vm.box_check_update = true

  config.vm.provider "virtualbox" do |vb|
    vb.gui = true
    vb.memory = "512"
  end

  config.vm.provision "shell", inline: <<-SHELL
    # Backup the original bootloader
    dd \
      if=/dev/sda \
      of=/vagrant/backedup_bootloader.bin \
      bs=1 \
      count=446
	 
    # Copy the fun bootloader to first 446 bytes of sda
    dd \
      if=/vagrant/custom_bootloader.bin \
      of=/dev/sda \
      bs=1 \
      count=446
	 	
    # Reboot the system to see the effect
    reboot
  SHELL
end
```

The [`Vagrantfile`](https://www.vagrantup.com/intro) above will launch a quick test VM. We just need to sit back and relax. 

![](https://media.giphy.com/media/8ynPshGnD8WnS/giphy.gif#center)

After successful bootup and reboot, It displayed the expected message.

![](https://i.imgur.com/6eV40TA.png#center)