---
title: "Eudyptula Task2"
date: 2022-06-01T15:14:27+05:30
draft: false
# showtoc: false
tags: [Eudyptula, linux]
series: [Eudyptula]
description: Task 2 for Eudyptula challenge
# math: true
# ShowBreadCrumbs: false
# searchHidden: true
# hideMeta: true
---

```txt{linenos=false}
This is Task 02 of the Eudyptula Challenge
------------------------------------------

Now that you have written your first kernel module, it's time to take
off the training wheels and move on to building a custom kernel.  No
more distro kernels for you, for this task you must run your own kernel.
And use git!  Exciting isn't it!  No, oh, ok...

The tasks for this round is:
  - download Linus's latest git tree from git.kernel.org (you have to
    figure out which one is his, it's not that hard, just remember what
    his last name is and you should be fine.)
  - build it, install it, and boot it.  You can use whatever kernel
    configuration options you wish to use, but you must enable
    CONFIG_LOCALVERSION_AUTO=y.
  - show proof of booting this kernel.  Bonus points for you if you do
    it on a "real" machine, and not a virtual machine (virtual machines
    are acceptable, but come on, real kernel developers don't mess
    around with virtual machines, they are too slow.  Oh yeah, we aren't
    real kernel developers just yet.  Well, I'm not anyway, I'm just a
    script...)  Again, proof of running this kernel is up to you, I'm
    sure you can do well.

Hint, you should look into the 'make localmodconfig' option, and base
your kernel configuration on a working distro kernel configuration.
Don't sit there and answer all 1625 different kernel configuration
options by hand, even I, a foolish script, know better than to do that!

After doing this, don't throw away that kernel and git tree and
configuration file.  You'll be using it for later tasks, a working
kernel configuration file is a precious thing, all kernel developers
have one they have grown and tended to over the years.  This is the
start of a long journey with yours, don't discard it like was a broken
umbrella, it deserves better than that.
```

# What, why?

Kernel is the main component of any operating system and is also referred as  the "Heart of the Operating System". It is at the core of all the layers present in OS and can have complete access to all the hardware (CPU, disk, RAM, etc). Therefore, it runs on very high privileges. Basically it handles most of the hardware related tasks (Allocate memory, CPU scheduling, etc) and most of the process related tasks (Copying file from/to disk, Uploading/Downloading, opening browser to read this blog, etc)

**Wait, what? Does it have control to everything we do on our computers?**

A big YES and small no. It is responsible to send data across multiple resources in your system and it can intercept everything there. But it depends if it can understand what it sees.

Kernel has mainly 4 tasks:

1. Keep track of the memory - who is using it and how much; And where.
2. Decides who uses CPU, when and for how long
3. Takes data from processes and passes sensible code to hardware for processing it and vise-versa.
4. Receives requests via system calls (API calls; but not Web API calls) from processes. This is used to do low level stuff and build amazing tools like docker.

Talking to kernel is difficult and can be dangerous if not used properly. Most of the times, user does not need to talk to kernel directly, and have got few layers of abstraction on top of it - Device drivers, system libraries, CLI shells, GUI shells (Graphical thing which comes up, when you start your system), etc.

This gives rise to the idea of 2 spaces - user space and kernel space. Kernel space is the memory segment that is used only by kernel and users stay out of it. Another is user space memory segment, where user can do all what he wants.

The rough mind map would look something like below

```txt{linenos=false}
[ [ [ [Hardware] --> Kernel ] --> OS ]  --> Process(browser)]

# If process fails in OS, damage is small and might be recovered by kernel.
# If kernel crashes, Your system goes down.
# If hardware fails, you cry!!
```

This is the complete bundle which makes up your system.  Now what I want to take out of this whole jibber-jabber is that a **kernel** is a **_piece of software_** that works with hardware and other user-friendly softwares to solve your problems or play games and have fun.

Since it is a piece of software, we can download it and replace old versions with new versions (manually or via a script/program/whatever). Another option for tech savvy people is to custom compile it. There could be many reasons to compile a linux kernel by yourself, few possible reasons are:-

- You want to know how it is done.
- You might want to brag about it and feel superior and very tech savvy.
- You want to face "I use arch BTW" community. (FYI, I use arch BTW!!)
- You want optimal performance on specific hardware and architecture.
- You might want to disable/enable some kernel features.
- You might want to add support for extra hardware.
- You are solving eudyptula challenge, just like me :)

Regardless of why, knowing how to compile a linux kernel is very useful and cool.

![](https://media.giphy.com/media/dCLyraFCJhaLUsG3dX/giphy.gif#center)

### Getting the source code for kernel.

Getting source code for kernel is very easy. You just need to go to [kernel.org](https://kernel.org/) and download the required files. I'll be compiling [Linus Torvalds's git tree](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/) source code on `archlinux/archlinux` vagrant box.[^1]

```bash
# Use git to download the linux kernel source code. (just the latest commit => --depth=1)
git clone --depth=1 git://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git
```

Linux kernel is very huge software and might take up a minute to download (depends upon your connectivity). This is one of the reasons for why most of the extended functionalities are provided via loadable kernel modules. If you don't know about kernel modules the read [Eudyptula Task1](https://ayedaemon.github.io/post/eudyptula/eudyptula-task-1/) blog.

Inside the cloned directory, we have multiple files and sub-directories. Each sub-directory is for specified purpose like `arch` contains files for different system architecture and `security` contains files for `selinux` , `apparmor` and other security related files. In short, linux kernel is developed by thousands of developers in collaboration and not everybody knows about each file present in the source code and yet they have an understanding of where they have to make changes to achieve their goal. Very neat management!!

### Compiling Linux kernel

Just like every configurable software, linux kernel also provide configuration support via `.config` file. We can either use other kernel's config file or write a config file by ourselves.

#### Creating own config file from scratch

Creating own config file from scratch can be a bad idea for someone who is doing it for very first time. But if you still want to do it, I'm not gonna stop you. You can make the use of `Makefile` by typing `make config` from inside the kernel source code directory and then you'll have to simply answer `yes or no` for all the configurable options that kernel supports. [Read more here](https://www.linuxtopia.org/online_books/linux_kernel/kernel_configuration/ch05.html#id2564226)

#### Using existing config file

You can copy the config file of your existing kernel and use it as a base config to make further changes. This is a very efficient method if there are only few changes you need to make. Most of the kernel developers have their own config files which they have fine-tuned in so many years. To know about how to get config file for your linux system [read this stackoverflow thread](https://superuser.com/questions/287371/obtain-kernel-config-from-currently-running-linux-system). For my vagrant system, I can check my running kernel's config file using below command

```bash
ls /proc/config.gz

zcat /proc/config.gz  | grep  ".*CONFIG_" | wc -l
# Output = 9128
```

There are total `9128` configurable options here and it is very impractical to make all the proper changes with a text editor in one go, so  instead, we will do it with some [TUI script](https://en.wikipedia.org/wiki/Text-based_user_interface). Below script will copy the config file to working directory and start the TUI for you. Navigate to `linux` source code directory and run the below script.

```bash
# Copy config file and take a backup for later review
zcat /proc/config.gz > .config
cp .config ../old.config

# Install requirements to run `make menuconfig`
sudo pacman -S --noconfirm --needed\
	pkg-config ncurses \
	gcc \
	flex \
	bison

# Update config file.
make menuconfig
```

Linux kernel has a lot of make options and the best way to check supporting make options is via `make help` command. After executing `make menuconfig`, a TUI will open in shell which will help you to update, save and load the new configuration. This command uses the `.config` file from current directory to pre-fill the old config options, this makes it very easy for us to just focus on what we want to change.

For my config file, I simply enabled the `CONFIG_PRINTK_CALLER`   and `CONFIG_LOCALVERSION_AUTO` features of the kernel... and then saved the file with a filename - `new.config` (I want to keep it backed up for future tasks). We can compare the changed values from the old `.config` and newer `new.config` and see the difference.

```bash
diff .config ../old.config  | grep -i -E 'localver|printk'

#   < CONFIG_LOCALVERSION="ayedaemon"
#   > CONFIG_LOCALVERSION=""
#   < CONFIG_PRINTK_CALLER=y
#   > # CONFIG_PRINTK_CALLER is not set
```

Now we have very few steps left to be done. We need to **_compile the kernel_**, then **_install modules_** and finally, **_install kerrnel_**. Run below commands to get this done.

```
# Update .config with newer config file
mv -v new.config .config

# Backup the newer config file
cp -v .config ../new.config

# Install some more dependencies
sudo pacman -S --noconfirm --needed \
	bc \
	cpio   # If on arch based distro or using my vagrantfile

# Compile kernel (might require your input) - use -j4 to make it build faster
make -j4

# Install modules (takes some time) - user -j4 to make it build faster
sudo make modules_install -j4
```

If you are use LILO bootloader, then the kernel make file will do the job for you with this command - `sudo make install`. But if you are using GRUB, then you will have to make some manual steps by running below commands.

```
# Copy kernel image to /boot
sudo cp arch/x86_64/boot/bzImage /boot/vmlinuz-ayedaemonlinux

# Copy system.map to /boot
sudo cp System.map /boot/System-ayedaemonlinux.map

# Copy config file to /boot (just to be safe)
sudo cp .config /boot/ayedaemonlinux.kernel.config
```
Let's take a minute to see what we just did. After we configured the linux kernel using `make` and `.config` file, we compiled the kernel with our configuration requirements and then installed all the modules required. Once this is done, we got 2 important files we need :-
1. `vmlinuz` => Is the actual kernel file. Yes it is the kernel you were waiting for so long. If you fancy, do 	`file /boot/vmlinuz-ayedaemonlinux` and check the results.
2. `System.map` => This is the map file which stores the kernel symbol table information. [Read more about it here](https://rlworkman.net/system.map/)

Anyways, we need these files in our `/boot/` directory so that our boot-loader can load our compiled kernel. But our boot-loader is dumb, it can not simply detect the files from `/boot/` and show us options on the boot-loader screen, we will have to do that as well. You might also need to generate a `initrd` file depending upon what configurations you are using on your system. If you are following the steps from this blog, then you need **initrd** for sure. **Initrd** is the program that helps your kernel to load and boot up properly by providing the modules support that are not built into the kernel at compile time. In this blog, we have not compiled all the modules in the kernel that our kernel might need at boot time, so we will create a `initrd` file and then we can tell our boot loader about our custom kernel.

Use `mkinitcpio` command to generate a initrd file and then update bootloader config using `grub-mkconfig` command. If you want this kernel to be default, then you'll have to make proper changes to the boot config file. Read more about it from [arch wiki](https://wiki.archlinux.org/title/GRUB/Tips_and_tricks#Changing_the_default_menu_entry) or [stackoverflow](https://stackoverflow.com/questions/44422745/change-default-kernel-version-in-grub). If you are someone who prefers easy workarounds, you can also select the new kernel from the grub menu at boot time; Just make sure that `GRUB_TIMEOUT` variable (from `/etc/default/grub`) is not set to zero.

```
# generate initramfs
sudo mkinitcpio -k 5.18.0ayedaemon-g8ab2afa23bd1 -g /boot/initramfs-ayedaemonlinux.img

# update grub config - add entry to boot menu
sudo grub-mkconfig -o /boot/grub/grub.cfg

# Setup grub boot order if you want to - else use the lazy way
```

Output:-

```text{linenos=false}
#  sudo grub-mkconfig -o /boot/grub/grub.cfg

Generating grub configuration file ...
Found linux image: /boot/vmlinuz-linux
Found initrd image: /boot/initramfs-linux.img
Found fallback initrd image(s) in /boot:  initramfs-linux-fallback.img
Found linux image: /boot/vmlinuz-ayedaemonlinux
Found initrd image: /boot/initramfs-ayedaemonlinux.img
Warning: os-prober will not be executed to detect other bootable partitions.
Systems on them will not be added to the GRUB boot configuration.
Check GRUB_DISABLE_OS_PROBER documentation entry.
done
```
From the output of last command, we can see that my `ayedaemonlinux` was detected by grub and it also updated the `/boot/grub/grub.cfg` file with current detections. Now, lets reboot and hope everything works as expected. Select the custom kernel from grub menu if needed and boot into it. If successfull, you can check the kernel version and other information with `uname` command.

```text{linenos=false}
# Before Reboot --> uname -a
Linux archlinux 5.18.1-arch1-1 #1 SMP PREEMPT_DYNAMIC Mon, 30 May 2022 17:53:11 +0000 x86_64 GNU/Linux

# After Reboot --> uname -a
Linux archlinux 5.18.0ayedaemon-g8ab2afa23bd1 #1 SMP PREEMPT_DYNAMIC Wed Jun 1 16:54:51 UTC 2022 x86_64 GNU/Linux

```
We just compiled our very own first kernel and since we have not changed much of the kernel parameters and no user-space programs are affected with this. But we get our name on the kernel tag!!

![](https://media.giphy.com/media/jurcfxao8M3yzHmCjS/giphy.gif#center)




[^1]: Arch is a rolling distro and the packages can be easily upgraded to latest versions available. No `<package-name> too old` kind of errors.
