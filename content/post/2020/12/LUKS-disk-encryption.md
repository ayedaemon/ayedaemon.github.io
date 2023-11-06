---
title: LUKS Disk Encryption
date: 2020-12-19T13:15:55+05:30
tags:
  - linux
  - cryptograpy
  - security
draft: false
comments: false
---

> Linux Unified Key Encryption — Disk Encryption

<!--more-->

**cryptsetup** — *manage plain dm-crypt and LUKS encrypted volumes*

```
cryptsetup <OPTIONS> <action> <action-specific-options> <device> <dmname>
```

An encrypted blockdevice is protected by a key. A key is either:


* a passphrase, or
* a keyfile

### What the..?

Ok.. If you are new to encryption world, then it’s time to get a bit familiar data encryption.

There are 2 methods to encrypt your data:


* **Filesystem stacked level encryption** : Form of disk encryption where individual files or directories are encrypted by the file system itself. [read more here](https://en.wikipedia.org/wiki/Filesystem-level_encryption)
* **Block device level encryption** : The entire partition or disk, in which the file system resides, is encrypted.

![](https://cdn-images-1.medium.com/max/800/0*bg4VTXG8Lp6jq9aC.gif)

Before things go really technical and scary, let me show you how your data is stored in a harddisk.

![](https://cdn-images-1.medium.com/max/800/1*2cy1Ut_NVLQof_vTZWoLAQ.png)

Above diagram shows how your data is stored in a harddisk.


* You create files (I am calling it data chunks) and insert your data in it.
* These files are stored in a very systematic and managed system called **File System**.
* Partitions are formatted to carry a file system on it.
* Harddisks are divided into Partitions. ([Wanna know why? — ask Leo!](https://askleo.com/should_i_partition_my_hard_disk/))

Now when you know how your data is exactly stored in a harddisk. Let’s see how a **Block device level encryption** works.

![](https://cdn-images-1.medium.com/max/800/1*2easqwhcymbcCSp6fHSZ6g.png)

Here, a new layer is added in the usual thing.


* We attach a harddisk to our system.
* Create partitions on it.
* Encrypt the complete partition (make it password protected) 🔐
* Create filesystem (NTFS, EXT4, XFS, etc) on the encrypted partition.
* Write/save your data chunks.

![](https://cdn-images-1.medium.com/max/800/0*rcPPu_6o2rOgW2bS.gif)

---

### Just Do It now ✔️

### Installing required tools

I am using a RHEL based OS which uses yum/dnf package managers.

```
yum install cryptsetup -y

or

dnf install -y cryptsetup
```

### Creating the partition

`lsblk` - check the device name for the harddisk (sdb)

![](https://cdn-images-1.medium.com/max/800/1*4c6x1UUNyksMWLCc-Yqyaw.png)

`fdisk` - partitioning tool

![](https://cdn-images-1.medium.com/max/800/1*k_dPP7Hb4tGu7ARPeVLvOg.png)
![](https://cdn-images-1.medium.com/max/800/1*LEOYgsQ-9juROPCikHX42g.png)

### formating with luks

`cryptsetup -y -v luksFormat /dev/sdb1` - encrypt the partition

![](https://cdn-images-1.medium.com/max/800/1*SETeIxieb0fOBPCuX7aHNg.png)

`lsblk -f` - check the encrypted partition

![](https://cdn-images-1.medium.com/max/800/1*pBN1T2AcoQyo5B8FucJV2g.png)

`cryptsetup -v luksOpen /dev/sdb1 myencrypt` - map the encrypted partition to 'myencrypt'.

![](https://cdn-images-1.medium.com/max/800/1*zcUCmHHyDkU9YpBL5Zg5ZQ.png)

`lsblk -f` - check it

![](https://cdn-images-1.medium.com/max/800/1*5SKWOXd-lSX468LhgJNiZQ.png)

### creating a file system

`mkfs.xfs /dev/mapper/myencrypt` - create a file system on top of the encrypted partition.

`lsblk -f` - Check the layering and filesystem associated.

![](https://cdn-images-1.medium.com/max/800/1*ch_byQVfsZexQP7O7O7dag.png)

### creating a mountpoint

```mkdir -p /mnt/my_encrypted_backup mount -v /dev/mapper/myencrypt /mnt/my_encrypted_backup/```

*If you face such issues - SELinux lables blah blah blah*

![](https://cdn-images-1.medium.com/max/800/1*GtzHhCp6CPR8YWBTOz-yqQ.png)

*Type this on magic terminal —* *restorecon -vvRF /mnt/my_encrypted_backup/* *- This will restore the SELinux context back to defaults for the destination directory.*

### Checking luks dumps

```
cryptsetup luksDump /dev/sdb1
```

![](https://cdn-images-1.medium.com/max/800/1*tWyJ8XLcwJuyRG-aUl8BZA.png)

### Adding new key

```
mkdir /etc/luks-keys/; dd if=/dev/random of=/etc/luks-keys/mybackup\_key bs=32 count=1
```

![](https://cdn-images-1.medium.com/max/800/1*fE51gnVtgYGkSOeASgrzfQ.png)

```
cryptsetup luksAddKey /dev/sdb1 /etc/luks-keys/mybackup\_key
```

![](https://cdn-images-1.medium.com/max/800/1*FBkoDsoqSHZ9AWw73ICg2Q.png)

Checking the dumps again

![](https://cdn-images-1.medium.com/max/800/1*7pWmoHcrHZmLXtGISkrCZQ.png)

Now here are 2 slots available.


* one with the initial key I entered at the time of setting it up.
* another, just in the above step.

#### At this particular moment, there are few questions in my mind.

![](https://cdn-images-1.medium.com/max/800/0*-Pngs8QCzMnlwKVv.gif)

You should know them too.


1. If you want to unmount and remove the harddisk. You’ll have to follow the steps:
```
umount /mountpoints/sdb cryptsetup luksClose myencrypt
```
![](https://cdn-images-1.medium.com/max/800/1*6JplORU1iMFii5fwR4Reqg.png)
![](https://cdn-images-1.medium.com/max/800/1*TZ8IkBpyRwkufkvyPkIT2g.png)

2. If you want to open the luks partition with keyfile instead of the passphrase.
```
cryptsetup -v luksOpen /dev/sdb1 myencrypt --key-file=/etc/luks-keys/mybackup\_key
```
3. What if someone changes the content of the keyfile?

Creating a new key

![](https://cdn-images-1.medium.com/max/800/1*SuPXbF7MvbIufvP8qAikTw.png)

Add the key to the slots

![](https://cdn-images-1.medium.com/max/800/1*gJ41bDkh1J5UyNGIQjZJww.png)

Use key

![](https://cdn-images-1.medium.com/max/800/1*bykTB3_3rLa3w5AODyKA-w.png)

So the content inside the keyfile do matter; You can’t change it and expect things to work just fine for you.



### Time for some Automation

Get the UUID of the encrypted partition

![](https://cdn-images-1.medium.com/max/800/1*wasZr_6cKCRIibLohoPJvQ.png)

And make the below entry in `/etc/crypttab` file. (Check the UUID for your device - Don't copy mine!!)

```
myencrypt    UUID=48a20857-6f26-4352-89d5-e778f2d98950     /etc/luks-keys/mybackup\_key    luks
```

The above line is a combination of 4 fields:


* name of the mapped device.
* uuid of the encrypted partition
* keyfile to unlock the partiotion
* type of encryption used — luks

And then make below entry in `/etc/fstab` file.
```
/dev/mapper/myencrypt /mountpoints/sdb xfs defaults 0 0
```
*Want to learn more about* [*crypttab*](https://linux.die.net/man/5/crypttab) *and* [*fstab*](https://linux.die.net/man/5/fstab)

Last step to verify if the above steps worked fine or not.


* Remount and verify (using mount command with 'a' and 'v' flags for clarity)
* Reboot the system and check if everything works after reboot. (Trust me, things betray sometimes after reboot)

---

Want to [read more](https://wiki.archlinux.org/index.php/Dm-crypt/Device_encryption) about dm-crypt or device encryption?

---

![](https://miro.medium.com/max/480/0*IZmFYignS8jVnoSE.gif)
