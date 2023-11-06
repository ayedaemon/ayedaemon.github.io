---
title: Advanced Intrusion Detection Environment
date: 2020-12-19T14:25:28+05:30
tags:
  - IDS
  - Security
  - Linux
draft: false
comments: false
---

> Host-based intrusion detection system (HIDS) for checking the integrity of files.

<!--more-->


Advanced Intrusion Detection Environment (AIDE) is a **host-based intrusion detection system (HIDS)** for checking the integrity of files. It does this by creating a **baseline database** of files on an initial run, and then checks this database against the system on subsequent runs. File properties that can be checked against include inode, permissions, modification time, file contents, etc……….. [*more at archwiki*📚](https://wiki.archlinux.org/index.php/AIDE)

According to the definition, AIDE only checks for the **integrity of file** but **not for rootkits** and logs for other suspicious activities.

*But there are other HIDS tools that can do this for you. Like,* [*Splunk*](https://www.splunk.com/) *and* [*OSSEC*](https://www.ossec.net/)*.*

AIDE have provided a [pretty simple documentation](https://aide.github.io/doc/) to undertand and get familiar with it.

### How to install it?

```
# Check what repo will provide you aide tool.  

yum whatprovides aide  

# And then install it, if available.  

yum install aide -y
```

![aide-whatprovides-install](https://cdn-images-1.medium.com/max/800/1*GH5ZoirRAKBXJOdkSk4K2A.png)

### Next step ..??

Let’s check the files unpacked from the **aide package** we just installed.

![aide-rpm-ql](https://cdn-images-1.medium.com/max/800/1*WBBWo0dITSPIgjfNqPTrZA.png)

We found a configuration file — `/etc/aide.conf`.

```
# open the file with vim or your favourite text editor  

vim /etc/aide.conf  

# The file looked very huge so I checked its length.  

wc /etc/aide.conf  

# OUTPUT:  
# 312 765 7333 /etc/aide.conf
```

Fortunately they have given a man page for the configurations settings.

```
man 5 aide.conf
```
![aide.conf](https://cdn-images-1.medium.com/max/800/1*0HZv3rMcdAFMTbYiyPn_Xw.png)

This gives me a good news. There are only 3 types of line in the configuration file.

* There are the 1️⃣*configuration lines* which are used to **set configuration parameters** and **define/undefine variables**.
* There are 2️⃣*selection lines* that are used to indicate **which files are added** to the database.
* 3️⃣ *macro lines* **define or undefine variables** within the config file.
* Lines beginning with # are ignored as **comments**.#️⃣

![Really](https://cdn-images-1.medium.com/max/800/0*LRp9V0giXu-oxaae.gif)

You can now check the config file and things will make more sense to you. Also you can check the key-value pairs from [man page](https://linux.die.net/man/5/aide.conf).

### Enough for configuration… How to use it?

Go to the [man page](https://linux.die.net/man/1/aide) of aide.
```
# from terminal  

man aide
```

Again a reminder, and I quote.

> AIDE is an **intrusion detection system** for checking the **integrity of files**.

![man-aide](https://cdn-images-1.medium.com/max/800/1*gNw75QXgQiRPoxgopKvZSQ.png)

One thing to notice here is ***DIAGNOSTICS*** (*Scroll down to bottom on the man page*).

![diagnostics](https://cdn-images-1.medium.com/max/800/1*m3ygM3U9O_-eooMR-3NJcA.png)

Another is, that AIDE can be controlled using few basic commands.

![commands](https://cdn-images-1.medium.com/max/800/1*nYCpUl-Pv4oHGnjbTsYXew.png)

### Time for some fun now!!

![lets-play](https://cdn-images-1.medium.com/max/800/0*eV9LiAtIIoAbbflD.gif)

**Game-play**


* Create a folder and some files in it.
* Configure AIDE to add that folder in database.
* Have fun with the folder and files and check the AIDE logs for reports.

![create-folder](https://cdn-images-1.medium.com/max/800/1*4RBSuCrL2W9C1Y9MspWl2w.png)

Adding my new folder and files to aide.conf
```
#-------------- My-Settings ---------------  

myfilter = sha256  

/fun-with-aide myfilter
```

![add-to-conf](https://cdn-images-1.medium.com/max/800/1*aC940c2LfvDHrQu_D4nKoA.png)

This rule is a regular expression rule and will match the complete path of any file starting from /fun-with-aide, so this will include the files inside this folder.

Now some simple steps to follow:
```
* aide --init
* cp /var/lib/aide/aide.db.new.gz /var/lib/aide/aide.db.gz
* aide --check
```
![aide-check](https://cdn-images-1.medium.com/max/800/1*G7iCTRi0C6ttmWQe3-oxyA.png)

What if we tinker with the file /fun-with-aide/file1?

![tinker-file1](https://cdn-images-1.medium.com/max/800/1*wDbGmR7vps5vsqs3i8_Icg.png)

I have changed the content of the file1, due to which the sha256sum has also changed. This should be reported by **aide** in reports.

![aide-check](https://cdn-images-1.medium.com/max/800/1*z0Zq2wGpayGYo079V8bgUw.png)

This generates a report that tells about the changes. I’ll get a count of *added files*, *removed files* and *changed files*, along with the name of those files and some detailed information.

---

AIDE can be run manually if desired, but automation is the way nowadays.

![yeah-automation](https://cdn-images-1.medium.com/max/800/0*ip12LNV58018AQ8G.gif)

Check the below provided simple cron job script to automatically check for the changes. For more complex examples check [this](https://sources.gentoo.org/cgi-bin/viewvc.cgi/gentoo-x86/app-forensics/aide/files/aide.cron) and [this](https://rfxn.com/downloads/cron.aide).

```
# SOURCE: https://wiki.archlinux.org/index.php/AIDE  

#!/bin/bash -e  

# these should be the same as what's defined in /etc/aide.conf  
database=/var/lib/aide/aide.db.gz  
database\_out=/var/lib/aide/aide.db.new.gz  

if [ ! -f "$database" ]; then  
 echo "$database not found" >&2  
 exit 1  
fi  

aide -u || true  

mv $database $database.back  
mv $database\_out $database
```
---

**What about if attacker changed the database??**

When I checked the file type of the *aide.db.gz*... It came out to be a gzip compressed data, from Unix, max compression file.

This makes it very obvious to unzip this compressed file. I prefer using gunzip tool.

![operation-theatre](https://cdn-images-1.medium.com/max/800/1*rP1BY4TIP5OLCxQj4_GrGw.png)

Specification of the db is also mentioned in the file.

![db_spec](https://cdn-images-1.medium.com/max/800/1*FB264rJGlMj1Tr-4tPSWHg.png)

Visualizing the above in a tabular manner.

![](https://cdn-images-1.medium.com/max/800/1*3MXSU4ub4fgXolHfPB34Qw.png)

**You can add more filters and integrity checks to test other things as well.**

![being-tonystark](https://cdn-images-1.medium.com/max/800/0*v87DoqZLe0Cq2K-f.gif)

This whole db thing gives rise to a question. **What if the attacker modifies the db??**

Hmmm.. then he wins🤷‍♂️. You have to keep your db secure from attackers. For this, you should keep your database in **read-only mode**. So that it can be only read and no modifications can be done to this. Also you can keep the DB in a different location like in a **centralized server** or in a **removable media like pendrive**. Or you can have it your way.

You can read more about [Integrity Concepts](https://wiki.gentoo.org/wiki/Integrity/Concepts) here for better security guidelines.

### Conclusion

In the end, let’s understand how AIDE does what it does.

AIDE takes a “snapshot” of the state of the system, register hashes, modification times, and other data regarding the files defined by the administrator. This “snapshot” is used to build a database that is saved and may be stored on an external device for safekeeping.

When the administrator wants to run an integrity test, the administrator places the previously built database in an accessible place and commands AIDE to compare the database against the real status of the system. Should a change have happened to the computer between the snapshot creation and the test, AIDE will detect it and report it to the administrator. Alternatively, AIDE can be configured to run on a schedule and report changes daily using scheduling technologies such as cron.🔚

![end](https://cdn-images-1.medium.com/max/800/0*S_aFpI3KDiWfts2v.gif)
