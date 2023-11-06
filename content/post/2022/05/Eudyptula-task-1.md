---
title: "Eudyptula Task1"
date: 2022-05-25T15:14:27+05:30
draft: false
# showtoc: false
tags: [Eudyptula, linux]
series: [Eudyptula]
description: Task 1 for Eudyptula challenge
# math: true
# ShowBreadCrumbs: false
# searchHidden: true
# hideMeta: true
---

### What is this?

> The [Eudyptula Challenge](http://eudyptula-challenge.org/) is a series of programming exercises for the Linux kernel, that start from a very basic “Hello world” kernel module, moving on up in complexity to getting patches accepted into the main Linux kernel source tree.

Unfortunately, this project is not accepting any new applicants right now. So I decided to gather tasks details from other online sources and complete them locally.

### Task-1

```txt{linenos=false}
This is Task 01 of the Eudyptula Challenge
------------------------------------------

Write a Linux kernel module, and stand-alone Makefile, that when loaded
prints to the kernel debug log level, "Hello World!"  Be sure to make
the module be able to be unloaded as well.

The Makefile should build the kernel module against the source for the
currently running kernel, or, use an environment variable to specify
what kernel tree to build it against.
```

Linux provides a powerful and expansive API for applications, but sometimes that’s not enough. Interacting with a piece of hardware or conducting operations that require access to privileged information in the system can require a kernel module. In this task we have to write a kernel module that basically prints "Hello World!".

### What is a Kernel Module?

A Linux kernel module is a piece of compiled binary code that is inserted directly into the Linux kernel, running at ring 0, the lowest and least protected ring of execution in the x86–64 processor. Code here runs completely unchecked but operates at incredible speed and has access to everything in the system.

A loadable kernel module (LKM) is a mechanism for adding code to, or removing code from, the Linux kernel at run time. They are ideal for device drivers, enabling the kernel to communicate with the hardware without it having to know how the hardware works. The alternative to LKMs would be to build the code for each and every driver into the Linux kernel.

Without this modular capability, the Linux kernel would be very large, as it would have to support every driver that would ever be needed for the system to work properly. You would also have to rebuild the kernel every time you wanted to add new hardware or update a device driver.

Kernel modules run in kernel space and applications run in user space, and both kernel space and user space have their own unique memory address spaces that do not overlap. This approach ensures that applications running in user space have a consistent view of the hardware, regardless of the hardware platform. The kernel services are then made available to the user space in a controlled way through the use of system calls. The kernel also prevents individual user-space applications from conflicting with each other or from accessing restricted resources through the use of protection levels (e.g., superuser versus regular user permissions).

### Prepare system for building LKMs

The system must be prepared to build kernel code, and to do this you must have the Linux headers installed on your device. On a typical Linux desktop machine you can use your package manager to locate the correct package to install. For example, under 64-bit Centos7 you can use the below code. Sometimes the package manager provides multiple version of headers, then you must install the headers for the exact version of your kernel build.

```bash
# Update system
sudo yum update -y

# Install headers
sudo yum install -y kernel-devel kernel-headers

# Check headers
ls /usr/src/kernels/$(uname -r)
```

### Write first module - Hello World

The LKM code is very different from the regular user-space C program. Typical computer programs are reasonably straightforward. A loader allocates the memory for the program, then loads the program and other shared libraries into memory. Instruction Execution begins at some entrypoint (typically `main()` in C/C++ programs). On exit, OS identifies any memory leaks and frees lost memory to pool.

The LKMs are not applications - For a start there is no `main()`  and no `printf()` functions!!. They also do not have any automatic cleanup. Interestingly, they also do not have any floating-point support. In LKMs, the kernel module have atleast 2 entrypoint like functions; These functions executes at loading or unloading of the LKM.

The above can be a lot to digest all at once but it is important that they are addressed. Now, we can wrap our minds around the below code and understand how it works.

To start with, we need a `HelloWorld.c` file with  2 function definitions - `hello_world_init()` and `hello_world_exit()`. We then register first function to be executed when the LKM is loaded in the memory and the later is registered to be executed at unloading of the LKM. There are few extra functions that configure the metadata for the created module.

```C
#include <linux/init.h>
#include <linux/module.h>
#include <linux/kernel.h>


MODULE_LICENSE("GPL");
MODULE_AUTHOR("ayedaemon");
MODULE_DESCRIPTION("Eudyptula task1");


static int hello_world_init(void)
{
  printk(KERN_DEBUG "Hello World!\n");
  return 0;
}


static void hello_world_exit(void)
{
  printk(KERN_DEBUG "Bye Bye World!\n");
}


module_init(hello_world_init);
module_exit(hello_world_exit);

```

In kernel space, we do not have access to `printf()` functions, instead we have a very similar in usage function called `printk()`[^1], and you can call it from anywhere withing the LKM code. Read more about `printk()` from [here](https://www.kernel.org/doc/html/latest/core-api/printk-basics.html).[^2]

Now that we’ve constructed the simplest possible module, let’s understand the important parts in detail:

- The “includes” cover the required header files necessary for Linux kernel development.

- `MODULE_LICENSE` can be set to a variety of values depending on the license[^3] of the module. Other following 2 lines are also a part of module metadata.

- At the end of the file, we call module_init and module_exit to tell the kernel which functions are or loading and unloading functions. This gives us the freedom to name the functions whatever we like.


### ... make Makefile

A Makefile is required to build the kernel module — in fact, it is a special **kbuild** Makefile. Below is the `Makefile` used to build the above LKM code.

```Makefile
obj-m += HelloWorld.o
KDIR := /lib/modules/$(shell uname -r)/build

all:
	$(MAKE) -C $(KDIR) M=$(PWD) modules


clean:
	$(MAKE) -C $(KDIR) M=$(PWD) clean
```

First line of this Makefile is called *goal definition* and it defines the module to be built.  The rest of the Makefile is a regular makefile. Here, `-C` option switches the directory to the kernel directory before performing any make tasks. The `M=$(PWD)` variable assignment tells the make command where the actual project files exist, which helps make to return back to the project directory from kernel directory.

All going well, the process to build the kernel module should be straightforward, provided that you have installed the Linux headers as described earlier. The steps are as follows:

```txt
[vagrant@centos7 task-1]$ ls
HelloWorld.c  Makefile  README.md

[vagrant@centos7 task-1]$ make
make -C /lib/modules/3.10.0-1160.62.1.el7.x86_64/build M=/vagrant_data/task-1 modules
make[1]: Entering directory `/usr/src/kernels/3.10.0-1160.62.1.el7.x86_64'
  CC [M]  /vagrant_data/task-1/HelloWorld.o
  Building modules, stage 2.
  MODPOST 1 modules
  CC      /vagrant_data/task-1/HelloWorld.mod.o
  LD [M]  /vagrant_data/task-1/HelloWorld.ko
make[1]: Leaving directory `/usr/src/kernels/3.10.0-1160.62.1.el7.x86_64'
```

Once the module is successfully buit, we can test it by loading the module using `insmod` command.

```txt
[vagrant@centos7 task-1]$ ls -l *.ko
-rw-r--r--. 1 vagrant vagrant 101880 May 25 18:03 HelloWorld.ko

[vagrant@centos7 task-1]$ sudo insmod HelloWorld.ko
[vagrant@centos7 task-1]$ dmesg | tail -1
[35803.038855] Hello World!

[vagrant@centos7 task-1]$ lsmod | head -2
Module                  Size  Used by
HelloWorld             12496  0

```

The metadata information coded in the LKM can be checked with `modinfo` command.

```txt
[vagrant@centos7 task-1]$ modinfo HelloWorld.ko
filename:       /vagrant_data/task-1/HelloWorld.ko
description:    Eudyptula task1
author:         ayedaemon
license:        GPL
retpoline:      Y
rhelversion:    7.9
srcversion:     7969E1C9B651C03B53BA6B2
depends:
vermagic:       3.10.0-1160.62.1.el7.x86_64 SMP mod_unload modversions
```

At last, the module can be unloaded easily with `rmmod` command.

```txt
[vagrant@centos7 task-1]$ sudo rmmod HelloWorld.ko
[vagrant@centos7 task-1]$ dmesg | tail -2
[35803.038855] Hello World!
[35983.753824] Bye Bye World!
```

### Conclusion

Hopefully you have built your first loadable kernel module (LKM). Despite the simplicity of the functionality of this module there was a lot of material to cover — by the end of this article: you should have a broad idea of how loadable kernel modules work; you should have your system configured to build, load and unload such modules; and, you should be able to define custom parameters for your LKMs.

Just remember that you are completely on your own in kernel land. There are no backstops or second chances for your code. If you’re quoting a project for a client, be sure to double, if not triple, the anticipated debugging time. Kernel code has to be as perfect as possible to ensure the integrity and reliability of the systems that will run it.



[^1]: If there is no `\n` character at the end of the `printk()` string, then the next `printk()`  string will also be printed in dmesg. I was able to see both `Hello World!` and `Bye Bye World` at the same time when I was either loading or unloading the module.

[^2]: https://www.kernel.org/doc/html/latest/core-api/printk-basics.html

[^3]: Linux kernel licensing rules - https://www.kernel.org/doc/html/latest/process/license-rules.html
