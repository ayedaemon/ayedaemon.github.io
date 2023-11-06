---
title: "Eudyptula Task 6"
date: 2022-09-18T13:57:01+05:30
draft: false
# showtoc: false
tags: [Eudyptula, linux]
series: [Eudyptula]
description: Task 6 for Eudyptula challenge
# math: true
# ShowBreadCrumbs: false
# searchHidden: true
# hideMeta: true
---


```txt{linenos=false}
This is Task 06 of the Eudyptula Challenge
------------------------------------------

Nice job with the module loading macros, those are tricky, but a very
valuable skill to know about, especially when running across them in
real kernel code.

Speaking of real kernel code, let's write some!

The task this time is this:
  - Take the kernel module you wrote for task 01, and modify it to be a
    misc char device driver.  The misc interface is a very simple way to
    be able to create a character device, without having to worry about
    all of the sysfs and character device registration mess.  And what a
    mess it is, so stick to the simple interfaces wherever possible.
  - The misc device should be created with a dynamic minor number, no
    need running off and trying to reserve a real minor number for your
    test module, that would be crazy.
  - The misc device should implement the read and write functions.
  - The misc device node should show up in /dev/eudyptula.
  - When the character device node is read from, your assigned id is
    returned to the caller.
  - When the character device node is written to, the data sent to the
    kernel needs to be checked.  If it matches your assigned id, then
    return a correct write return value.  If the value does not match
    your assigned id, return the "invalid value" error value.
  - The misc device should be registered when your module is loaded, and
    unregistered when it is unloaded.
  - Provide some "proof" this all works properly.
```


### Device drivers??

When a user adds a new part to a computer system, such a printer, the computer doesn't immediately understand how to connect with it and identify it. This requires some sort of translator that can mediate between the component and our operating system/Computer. These translators are called **device drivers**. The operating system and other software are typically instructed on how to interface with another piece of hardware by device drivers, which are typically extremely small pieces of software. 

For example, In some laptops there is a dedicated *CAPSLOCK* LED, which toggles to indicate the state of the key itself. It seems like a really easy and straightforward concept. A key and a led are there; when the key is pressed, the led toggles; when the key is pressed again, the led toggles once more. However, There is lot more going on than what meets the eye.

When the *CAPSLOCK* key is pressed, userspace application that connects with the relevant mediator/translator and transmits the message to it. The translator then manages this message, analyses it, and then executes some operations on the associated hardware device (LED). 


```goat

             ┌───────────────┐
             │               │
             │   Program     │              USERSPACE
             │               │
             └───────┬───────┘
                     │
                     │
   ──────────────────┼────────────────────────
                     │
                     │
            ┌────────▼────────┐
            │                 │
            │  Device Driver  │              KERNEL
            │                 │
            └────────┬────────┘
                     │
                     │
   ──────────────────┼────────────────────────
                     │
                ┌────▼─────┐
                │   LED    │                HARDWARE
                └──────────┘

```


Now we know what are device drivers and how do they fit in the bigger picture. But that's not all! 

In linux, these device drivers are usually implemented as kernel modules, that provides an interface between the actual hardare device and the userspace "[files](https://www.kernel.org/doc/html/latest/filesystems/vfs.html)". On the basis of speed, volume and way of organizing the data to be transfered from userspace to the device and vice versa, device drivers are categorized under 2 types. *(Note:- There is one more type of device called network device, but for this article it's better to not start discussing about those)*

1. Character devices (Slow and manages small amount of data; used for keyboards, mouse, etc)
2. Block devices (Fast and can manage bulk data with ease and efficiency; used mainly for storage devices)

A typical linux `ls -l` command gives us a lot of information about the kind of device each file is.

```bash{linenos=false}
ls -l /dev

## Output (snipped)
# lrwxrwxrwx  1 root root          3    Sep 16 19:56 cdrom -> sr0
# brw-rw----  1 root disk        8,0    Sep 16 19:56 sda
# brw-rw----  1 root disk        8,1    Sep 16 19:56 sda1
# brw-rw----  1 root disk        8,2    Sep 16 19:56 sda2
# brw-rw----+ 1 root optical    11,0    Sep 16 19:56 sr0
# crw--w----  1 root tty         4,0    Sep 16 19:56 tty0
# crw-rw-rw-  1 root root        1,5    Sep 16 19:56 zero

```

First character of each line from the above output gives the type of the file it is, for example:

- `l` indicates that the file is a link file.
- `b` is the indicator for block device.
- `c` is for the character device.


Another important thing this gives out are unique identifiers associated with each device. These identifiers consists of **two comma separated numbers** - `major number` and `minor number`. 

Major number tells about the driver associated with the device. In the output above, `sda`, `sda1` and `sda2` all are managed by driver `8`. The kernel uses the major number at open time to dispatch execution to the appropriate driver. While the minor number is used by driver (specified by major number) to differentiate among multiple devices handled by the driver.

If you wish to read more about this, here is a [good article about major and minor numbers.](https://www.oreilly.com/library/view/linux-device-drivers/0596000081/ch03s02.html)[^ldd_book]

[^ldd_book]: https://www.oreilly.com/library/view/linux-device-drivers/0596000081/ch03s02.html

At this point we can take a guess that these numbers are not random numbers, but they have a meaning to it. [Here](https://www.kernel.org/doc/html/latest/admin-guide/devices.html) [^device_kernel] is the official registry of allocated devices numbers which we will have to keep in mind before writing a device driver.

[^device_kernel]: https://www.kernel.org/doc/html/latest/admin-guide/devices.html



### What is a misc char device driver??

Well, it's quite clear, isn't it? A Misc driver is a driver that is used for miscellaneous devices. 🤭🤭

They mostly behave like a char drivers, but they are unique in that we don't need to worry about all the complicated number registration issues. We can simply write our driver module and assign it a static minor number or ask kernel to provide a dynamic minor number. All the misc devices have common major number `10`. And just like char device, it supports all the file operation calls like open, read, write, close and IOCTL.

This is quite useful when we want to write a basic driver for a simple functionality and save ourselves from the mess of allocating and registering a major number.

#### Your first misc char device driver

In linux kernel source, `struct miscdevice` is defined in [`linux/miscdevice.h` file](https://elixir.bootlin.com/linux/latest/source/include/linux/miscdevice.h#L79)


```c

struct miscdevice  {
	int minor;
	const char *name;
	const struct file_operations *fops;
	struct list_head list;
	struct device *parent;
	struct device *this_device;
	const struct attribute_group **groups;
	const char *nodename;
	umode_t mode;
};

```

For this task, we will need only 3 members of the above struct.

1. `int minor`: To allocate the minor number, either static or dynamic.
2. `const char *name`: To give the name to the device.
3. `const struct file_operations *fops` : To allow custom file operations like read and write.


This will allow us to write a basic module that will work as misc device driver that can be loaded and unloaded from the kernel. Create a file with name `misc_char_device_driver.c` and paste the below code in that.

```c
#include <linux/miscdevice.h>
#include <linux/module.h>
#include <linux/init.h>
#include <linux/kernel.h>

// message formatting - optional
#define pr_fmt(fmt) KBUILD_MODNAME ": " fmt

// Device name
#define MISC_DEVICE_NAME "eudyptula"


// Misc Device structure
static struct miscdevice my_misc_device = {
  .minor = MISC_DYNAMIC_MINOR,
  .name = MISC_DEVICE_NAME,
};


// Entry point
static int hello_world_init(void)
{
  int ret = misc_register(&my_misc_device);

  pr_info("Hello from module; Return %d\n", ret);
  if (ret < 0)
          return -EFAULT;
  return 0;
}


// Exit point
static void hello_world_exit(void)
{
  misc_deregister(&my_misc_device);
  pr_info("Exiting from module\n");
}


module_init(hello_world_init);
module_exit(hello_world_exit);
MODULE_LICENSE("GPL");
MODULE_AUTHOR("ayedaemon");
MODULE_DESCRIPTION("Eudyptula task6");
```

Compile and load module using below `Makefile`


```makefile
KDIR := /lib/modules/$(shell uname -r)/build

all: clean build install

build:
    $(MAKE) -C $(KDIR) M=$(PWD) modules

clean: uninstall
    $(MAKE) -C $(KDIR) M=$(PWD) clean

install:
    - sudo insmod misc_char_device_driver.ko
    sudo lsmod | grep misc_char_device_driver

uninstall:
    - sudo rmmod misc_char_device_driver

```

After compiling the above module and loading it, this will give me a character device in the `/dev/` directory.


```text{linenos=false}
crw------- 1 root root 10, 122 Sep 17 13:21 /dev/eudyptula
```

![](https://media.giphy.com/media/o75ajIFH0QnQC3nCeD/giphy.gif#center)

We can see that the file is a character type because of the initial `c` indicator. Also the major number is `10`, which is the common major number for all misc devices. The minor number in our case is random but if you want feel free to allocate a static one.


When loading and unloading this module, it'll also create some logs because of the `pr_info` function calls. These logs can be viewed via `dmesg | grep misc_char_device_driver` command.


```text{linenos=false}
[17543.585039] misc_char_device_driver: Hello from module; Return 0
[17583.624421] misc_char_device_driver: Exiting from module
```


#### Adding file operations to driver

Now we have a working character device that just exists but it does not support any file operations at this point. We can add the required file operations using the `const struct file_operations *fops` member of `struct miscdevice`. In linux kernel, `struct file_operations` is defined at [`linux/fs.h` file](https://elixir.bootlin.com/linux/latest/source/include/linux/fs.h#L1964). There are many file operations that are supported but we only need `read` and `write` for now.


Our new code will look something like this

```c

// SPDX-License-Identifier: GPL-2.0+

#include <linux/miscdevice.h>
#include <linux/fs.h>
#include <linux/module.h>
#include <linux/init.h>
#include <linux/kernel.h>

// message formatting
#define pr_fmt(fmt) KBUILD_MODNAME ": " fmt

// Device name
#define MISC_DEVICE_NAME "eudyptula"


// Custom read operation
static ssize_t misc_read(struct file *filp, char __user *buff, size_t cnt, loff_t *offt)
{
	pr_info("Read operation performed\n");
	return cnt;
}


// Custom write operation
static ssize_t misc_write(struct file *filp, const char __user *buff, size_t cnt, loff_t *offt)
{
	pr_info("Write operation performed\n");
  return cnt;
}


// Operations structure
const struct file_operations misc_fops = {
        .read = misc_read,
        .write = misc_write,
};


// Misc Device structure
static struct miscdevice my_misc_device = {
        .minor = MISC_DYNAMIC_MINOR,
        .name = MISC_DEVICE_NAME,
        .fops = &misc_fops,
};


// Entry point
static int hello_world_init(void)
{
        int ret = misc_register(&my_misc_device);

        pr_info("Hello from module; Return %d\n", ret);
        if (ret < 0)
                return -EFAULT;
        return 0;
}


// Exit point
static void hello_world_exit(void)
{
        misc_deregister(&my_misc_device);
        pr_info("Exiting from module\n");
}



module_init(hello_world_init);
module_exit(hello_world_exit);
MODULE_LICENSE("GPL");
MODULE_AUTHOR("ayedaemon");
MODULE_DESCRIPTION("Eudyptula task6");
```


After compiling the above module and loading it, this will provide additional functionality of read and write on the device. This can be tested by reading and writing to the `/dev/eudyptula` device now.



```bash
## Read operation
dd if=/dev/eudyptula of=/dev/null count=1

## Output in `dmesg`
# [23375.547714] misc_char_device_driver: Read operation performed


## Write operation
dd if=/dev/zero of=/dev/eudyptula count=1

## Output in `dmesg`
# [23353.947901] misc_char_device_driver: Write operation performed
```

#### Userspace <--[data]--> kernel module

Now it's time for the ultimate move, we have a misc character device driver that supports read and write operations to it, but it actually doesn't send any data from kernel to userland and vice-versa. This is because there is no shared memory where we can simply pass the variables or pointers to the location and do whatever we intend to do it.

Things are a bit different when have to transfer data between a kernel layer and userspace. Complete explaination is out of the scope for this article, but the short version is that the transfer is done with the help of 2 buffers, one on the kernel and other on the userspace. The userspace programs fill data in their buffer and address to that buffer is passed to the kernel. The kernel then uses that buffer to copy data to it's own buffer and with that we are done. [Here is a stackoverflow thread](https://stackoverflow.com/questions/8265657/how-does-copy-from-user-from-the-linux-kernel-work-internally) on the same topic.

![](https://media.giphy.com/media/fd1TSJqq3b4GI/giphy.gif#center)

But we need not to worry about all this complexity, for us it'll be as hard as calling the [`copy_from_user()`](https://www.kernel.org/doc/htmldocs/kernel-api/API---copy-from-user.html) [^copy_from_user] and [`copy_to_user()`](https://www.kernel.org/doc/htmldocs/kernel-api/API---copy-to-user.html) [^copy_to_user] function from our kernel module.

[^copy_from_user]: https://www.kernel.org/doc/htmldocs/kernel-api/API---copy-from-user.html

[^copy_to_user]: https://www.kernel.org/doc/htmldocs/kernel-api/API---copy-to-user.html


With this, our new `read` and `write` functions will look something like below:-


```c
// Custom read operation
static ssize_t misc_read(struct file *filp, char __user *buff, size_t cnt, loff_t *offt)
{
	char *my_id = "ayedaemon\n";
	int my_id_len = strlen(my_id)+1;

	pr_info("[begin] offt=%ld\n", *offt);
	if (*offt != 0)
		return 0;

	if ((cnt < my_id_len) ||                        // Check the size
		(copy_to_user(buff, my_id, my_id_len)))       // Copy to buffer
		return -EINVAL;

	*offt += cnt;
	pr_info("[ end ] offt=%ld\n", *offt);
	return cnt;
}


// Custom write operation
static ssize_t misc_write(struct file *filp, const char __user *buff, size_t cnt, loff_t *offt)
{
	char *my_id = "ayedaemon";
	int my_id_len = strlen(my_id);
	char temp[my_id_len+1]; // size = 10; including the null byte

	if ((cnt != my_id_len+1) ||                                 // Check input size (mainly to prevent overflows)
		(copy_from_user(temp, buff, my_id_len)) ||              // Copy 9 bytes from userland
		(strncmp(temp, my_id, my_id_len)))                      // finally, compare 9 bytes
	  return -EINVAL;
	else
	  return cnt;
}

```


In the above code, `read` function first checks the size of the buffer and then copies the `my_id` value to userspace buffer. And in `write` function, we check input size, then copies the data from userspace to `temp` buffer and then compares the data to be same. There are other things in the code but those are for extra checks, just to make sure the module does not crash.


We can now test the code with our makefile and everything should be as we are expecting it to be.


```bash
## Read the value from device
cat /dev/eudyptula

## Output
# ayedaemon

#-=-=-=-=-=-=-=-=-=-=-=-=-=-=

## Write "ayedaemon" to the device
echo "ayedaemon" > /dev/eudyptula

## Output - No output

#-=-=-=-=-=-=-=-=-=-=-=-=-=-=

## Write anything apart from "ayedaemon"
echo "something" > /dev/eudyptula

## Output - Gives error
# bash: echo: write error: Invalid argument

```

![](https://media.giphy.com/media/hZj44bR9FVI3K/giphy.gif#center)


Now we have achieved the goal of this task, let's check the formatting of the code so that it follows the proper coding conventions that linux kernels developers have set.


```bash
./linux/scripts/checkpatch.pl -f misc_char_device_driver.c

## Output

# total: 0 errors, 0 warnings, 99 lines checked
# misc_char_device_driver.c has no obvious style problems and is ready for submission.
```

### Conclusion

There are mainly 2 types of device drivers - character and block. But there are also network device drivers which are completely different from the ones we discussed here. This article tries to provide basic introduction to misc character devices and details about how to write your own driver. All the code can be found in the [github repo here](https://github.com/ayedaemon/eudyptula) [^githubrepo] for you to playaround.

[^githubrepo]: https://github.com/ayedaemon/eudyptula