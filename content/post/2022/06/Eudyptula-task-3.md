---
title: "Eudyptula Task3"
date: 2022-06-16T16:14:27+05:30
draft: false
# showtoc: false
tags: [Eudyptula, linux]
series: [Eudyptula]
description: Task 3 for Eudyptula challenge
# math: true
# ShowBreadCrumbs: false
# searchHidden: true
# hideMeta: true
---

```txt{linenos=false}
This is Task 03 of the Eudyptula Challenge
------------------------------------------

Now that you have your custom kernel up and running, it's time to modify
it!

The tasks for this round is:
  - take the kernel git tree from Task 02 and modify the Makefile to
    and modify the EXTRAVERSION field.  Do this in a way that the
    running kernel (after modifying the Makefile, rebuilding, and
    rebooting) has the characters "-eudyptula" in the version string.
  - show proof of booting this kernel.  Extra cookies for you by
    providing creative examples, especially if done in intrepretive
    dance at your local pub.
  - Send a patch that shows the Makefile modified.  Do this in a manner
    that would be acceptable for merging in the kernel source tree.
    (Hint, read the file Documentation/SubmittingPatches and follow the
    steps there.)
```

Linux kernel source code is very huge and compiling such source code files can be tiring, especially when you have to include several source files from different directories and type the compiling command every time you need to compile. **`Makefiles`** are the solution to automate and simplify this task.

### What is a Makefile??

Makefile is a specially formated file which contains all the steps to compile your program in the form of rules. These rules are also sometimes referred as recipes (...but they both are totally different things🤷‍♂️) and these rules are executed by a utility called `make`. You might have used **make** to compile a program from source code. Most open source projects use it to compile a final executable binary, which can then be installed typically using `make install`.


#### Understand with examples

Let's start with very simple and classic example - "Hello World". To begin with, create a new directory called `my_make_dir` containing a `Makefile` with below contents in it.

```Makefile
# Comment - this says hello
say_hello:
		echo "Hello World"
```

If you run `make say_hello` inside the directory, it'll give below output:

```text{linenos=false}
echo "Hello World"
Hello World
```

We just created our first Makefile rule and triggered it to run. Here `say_hello` behaves like a function name, this is also called **target** and `echo "Hello World"` is a **recipie**.  In a single Makefile there can be multiple targets with multiple sets of recepies.

Let's take a look at another example now,

```Makefile
# Comment - this says hello
say_hello:
		echo "Hello World"


say_bye: say_hello
		echo "I'm going now!"
		echo "Bye Bye world"
```

In above example, we have 2 targets, namely, `say_hello` and `say_bye`.  In the target - `say_bye`, we have set `say_hello` as dependency or prerequisite. Pre-requisites are like another targets in the Makefile which should run before the intended target. So when we will invoke `say_bye` command, it'll first trigger the prerequisite `say_hello` and then `say_bye` will be triggered.

```txt{linenos=false}
# COMMAND -  make say_bye

echo "Hello World"
Hello World
echo "I'm going now!"
I'm going now!
echo "Bye Bye world"
Bye Bye world
```

Another key point about make utility is that whenever it does not take any targets to trigger, it'll trigger the top most target present in the Makefile. In this case, it'll trigger `say_hello` as it is in the first target in the Makefile.


```txt{linenos=false}
# COMMAND -  make

echo "Hello World"
Hello World
```


#### More practical examples

Now we know some basic terminology around Makefiles and how it works. Let's take a look at some more practical example that can be related to a real-world task.

```Makefile
all: say_hello generate

say_hello:
	@echo "Hello World"

generate:
	@echo "Creating empty text files..."
	touch file-{1..10}.txt

clean:
	@echo "Cleaning up..."
	rm *.txt

say_bye: clean
	@echo "Bye World"
```

In above example,  we have got many targets, some with pre-requisites and some without those. Let's pick each one of them and go one by one:

1. `all` : This is the first target in the file and if we do not pass any target to `make` utility, it'll trigger this target by default. This has got no recipes - just a simple rule without recipes. But this rule has some pre-requisites that will run before anything (in the given order - First `say_hello` and `generate`.
2. `say_hello` : This is another target that can be triggered as a single rule or as a pre-requisite for `all` target. Anyways, it'll simply print the following string - `Hello World`.
3. `generate` : This target is responsible to generate 10 empty files ranging from 1-10. Just like `say_hello`, it can also be triggered as a single standalone target or as a pre-requisite for `all` target.
4. `Clean` : This target cleans all the .txt files from the folder. Idea is to delete all the files created by `generate` target.
5. `say_bye` : This target prints "Bye World", but has `clean` target as a pre-requisite so it'll first clean and then it'll run it's own recipes.

*Note:- any line after # (hash) character will be treated as comments by make utility*

Now we understand what the Makefile consists of, let's see how we are going to use it for our case.

1. To get a "Hello World" msg and create all the files we can simply type `make`. This will trigger the first target `all`, that in turn will execute `say_hello` and `generate`.
2. Once you are done with everything and what to clean-up the mess, you can just type `make bye_world`. This target will first clean and then provide you with a goodbye message.

That's it. Pretty easy, isn't it? Now let's bang our head with another example that is more close to the real world application.

For this, we need to write our program (I'll use C language for that, language is never a barrier for make command). Open up a text editor and write your first `HelloWorld.c` program.

```c
#include <stdio.h>

int main() {
    printf("Hello World\n");
    return 0;
}
```

The above program will simply print "Hello World" when compiled and executed. We can write make recipes for this now.

```Makefile
# Compile the source code
HelloWorld.o: HelloWorld.c
	gcc  HelloWorld.c   -o HelloWorld.o

# Execute the binary
HelloWorld: HelloWorld.o
	./HelloWorld.o
```

The above `Makefile` has 2 rules with `HelloWorld.o` and `HelloWorld` as targets. Both targets have some recipes and some pre-requisites to it. Remember, the pre-requisite executes first and then the actual recipe for the target... That's how these rules work.

So if I trigger `HelloWorld` target, then it'll trigger the `HelloWorld.o` target as pre-requisite, which will compile the `HelloWorld.c` source code; Once the `HelloWorld.o` target is completed, it'll execute the generated `./HelloWorld.o` binary... This can be now further reduced to just invoking single `make` command, by making sure that `HelloWorld` target runs first by default.

You'll have to change the Makefile to get desired behaviour,

```Makefile
# Creates a variable (this points to the compiler)
CC=gcc

# First target in the file, default.
all: HelloWorld

# Executes the source code
HelloWorld: HelloWorld.o
	./HelloWorld.o

# Compile the source code
HelloWorld.o: HelloWorld.c
	${CC}  HelloWorld.c   -o HelloWorld.o
```

Above Makefile is now good to compile and run our source code. It has got a variable that stores the compiler name, and it is used in `HelloWorld.o` target with `${CC}` syntax. I believe you understand the rest.

![](https://media.giphy.com/media/nvkcS2Qc6pvId8S11b/giphy.gif#center)

Now you can invoke `make` and it'll compile the source and execute the binary in a single go. One key thing to observe here is, that when we run make multiple times without changing the source code, it is not recompiling the code. This saves us a lot of time. And this is exactly why make is so de-facto automation tool for such usecases. (not "de-facto", but you get it, right?)

Now, what about cleaning?? We gotta clean our mess. Although we don't have huge mess, but mess is mess and one gotta take care of his own mess. So we are going to write another rule in our Makefile.

```Makefile
# Deletes the executable binary file
clean:
	rm -rf HelloWorld.o
```

Now, anybody who will be using this, just needs to remember 2 commands:-

- `make` : to actually compile and run the binary file.
- `make clean` : To remove the compiled binary file.


### Makefile for kernel

Linux Kernel also uses makefiles (Plural; not singular) to make the building process relatively very easy for anybody. They just have to type 2 commands and a fresh new custom kernel will be ready for them. Although, this might take a lot of time depending upon what hardware you are running on and what files are you gonna compile.

Take a look at the top few lines of the `Makefile` in the top most directory of source tree.

```Makefile
# SPDX-License-Identifier: GPL-2.0
VERSION = 5
PATCHLEVEL = 19
SUBLEVEL = 0
EXTRAVERSION = -ExtraVersionText
NAME = Superb Owl
```

This tells about a lot of things related to version and name given to that version. We can see our `EXTRAVERSION` which I've changed in the file (for Task 3 - Eudyptula Challenge).

Reading further down, we have so many sections, separated with comment blocks. These comment blocks are simple and grep-able, give them a try. Once we are comfortable with the idea of Makefiles and linux kernel, we can now connect all the dots and understand how things are linked together.

 - We started with a `.config` file, that contains all the required kernel configurations. This file is then read by the top-level `Makefile` in linux kernel directory.
 - This Makefile is responsible for building 2 major products: `vmlinux` & `modules`. To achive this, **make** goes recursively into the sub-directories of kernel source tree and builds and compiles everything. The list of directories is determined using the `.config` file.... because that's what we want to compile.

According to the official kernel docs, people have four different relationships with the kernel Makefiles.

- Some will simply build kernel using commands like `make menuconfig` and `make`
- Some will work with device drivers or other kernel features and will have to deal with kbuild makefiles for each subsystem they are working on.
- A few will be working with architecture specific code and will be responsible for arch makefiles.
- And then, there are kbuild developers.... people who work on the kernel build system itself.

There are many things that a kernel developer needs to understand when he/she is working on kernel features. One of those many things is to understand kernel build makefiles. If you want, read more about important segments of a kernel makefile from [official documentations here](https://www.kernel.org/doc/html/latest/kbuild/makefiles.html#the-kbuild-files).[^1]
[^1]: https://www.kernel.org/doc/html/latest/kbuild/makefiles.html#the-kbuild-files


Now we know what makefile is, how it works and little bit about kernel makefile. Now the last part of the task - to make a patch of modified Makefile. This is very easy with git... you know *git* right?

![](https://media.giphy.com/media/hs1wBxNGuR7z2LyzHT/giphy.gif#center)

Just do `git diff Makefile` in linux kernel source directory. Executing this command should give you output like below.

```diff
diff --git a/Makefile b/Makefile
index 1a6678d81..25e909b50 100644
--- a/Makefile
+++ b/Makefile
@@ -2,7 +2,7 @@
 VERSION = 5
 PATCHLEVEL = 19
 SUBLEVEL = 0
-EXTRAVERSION = -rc2
+EXTRAVERSION = -ExtraVersionText
 NAME = Superb Owl

 # *DOCUMENTATION*
```

But this is not a patch... it is just a diff, showing what things changed. To make a actual patch you need to do more than this.

```bash
## After modifying the Makefile
git add Makefile
git commit -m "Modified Makefile"
git format-patch -1 HEAD -- Makefile

## OUTPUT - 0001-modified-makefile.patch
```


```txt{linenos=false}
# COMMAND -  cat 0001-modified-makefile.patch

From 45176125f95a5606ab4334f334634e19492f4928 Mon Sep 17 00:00:00 2001
From: ayedaemon <ris3234@gmail.com>
Date: Sat, 18 Jun 2022 10:54:16 +0530
Subject: [PATCH] modified makefile

---
 Makefile | 2 +-
 1 file changed, 1 insertion(+), 1 deletion(-)

diff --git a/Makefile b/Makefile
index b815ea3..7d97ece 100644
--- a/Makefile
+++ b/Makefile
@@ -1,6 +1,6 @@
 VERSION = 5
 PATCHLEVEL = 19
 SUBLEVEL = 0
-EXTRAVERSION = -rc2
+EXTRAVERSION = -ExtraVersionText
 NAME = Superb Owl

--
2.36.1
```

Now this gives you a patch file in an email format. That's convinient if you just want generate a patch and send it to someone using CLI mail client tools. Your work as a bug fixer/ feature developer/ etc is done once you have submitted the patch.

#### Submit the patch!! But where??

Linux kernel developers have also made automation scripts for you that actually finds the maintainers for a file.

```bash
./scripts/get_maintainer.pl -f Makefile
```

Output:-

```
Masahiro Yamada <masahiroy@kernel.org> (maintainer:KERNEL BUILD + files below scripts/ (unless mai...)
Michal Marek <michal.lkml@markovi.net> (maintainer:KERNEL BUILD + files below scripts/ (unless mai...)
Nick Desaulniers <ndesaulniers@google.com> (reviewer:KERNEL BUILD + files below scripts/ (unless mai...)
Nick Terrell <terrelln@fb.com> (maintainer:ZSTD)
Alexei Starovoitov <ast@kernel.org> (supporter:BPF (Safe dynamic programs and tools))
Daniel Borkmann <daniel@iogearbox.net> (supporter:BPF (Safe dynamic programs and tools))
Andrii Nakryiko <andrii@kernel.org> (supporter:BPF (Safe dynamic programs and tools))
Martin KaFai Lau <kafai@fb.com> (reviewer:BPF (Safe dynamic programs and tools))
Song Liu <songliubraving@fb.com> (reviewer:BPF (Safe dynamic programs and tools))
Yonghong Song <yhs@fb.com> (reviewer:BPF (Safe dynamic programs and tools))
John Fastabend <john.fastabend@gmail.com> (reviewer:BPF (Safe dynamic programs and tools))
KP Singh <kpsingh@kernel.org> (reviewer:BPF (Safe dynamic programs and tools))
Nathan Chancellor <nathan@kernel.org> (supporter:CLANG/LLVM BUILD SUPPORT)
Tom Rix <trix@redhat.com> (reviewer:CLANG/LLVM BUILD SUPPORT)
linux-kbuild@vger.kernel.org (open list:KERNEL BUILD + files below scripts/ (unless mai...)
linux-kernel@vger.kernel.org (open list)
netdev@vger.kernel.org (open list:BPF (Safe dynamic programs and tools))
bpf@vger.kernel.org (open list:BPF (Safe dynamic programs and tools))
llvm@lists.linux.dev (open list:CLANG/LLVM BUILD SUPPORT)
```

The above list gives you the recipients list for you patch. Before submitting your patch, please read [the official kernel documentations](https://www.kernel.org/doc/html/latest/process/submitting-patches.html)[^2] to know mode about the patch submissions.
[^2]: https://www.kernel.org/doc/html/latest/process/submitting-patches.html

![](https://media.giphy.com/media/fvmbkNjNwBlHCG3Yin/giphy.gif#center)


#### how do they apply my patch??

Maintainers (or the person you just submitted your patch) will check your patch and if everything is good, they'll apply your patch to their codebase.

`git` provides an easy way to do that using the patch file... just type below command and the patch will be applied.

```bash 
git apply 0001-modified-makefile.patch
```


### Conclusion

Linux kernel is very huge project which involves tons of people. This looks very chaotic and scary, but it works!! Among many other tools/scripts, **Git** and **Makefiles** are the 2 important tools this chaotic process relies upon. One should have good understanding about these tools to take part in the development process of kernel.

