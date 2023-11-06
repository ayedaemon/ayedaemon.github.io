---
title: "Eudyptula Task4"
date: 2022-06-17T16:14:27+05:30
draft: false
# showtoc: false
tags: [Eudyptula, linux]
series: [Eudyptula]
description: Task 4 for Eudyptula challenge
# math: true
# ShowBreadCrumbs: false
# searchHidden: true
# hideMeta: true
---


```txt{linenos=false}
This is Task 04 of the Eudyptula Challenge
------------------------------------------

Wonderful job in making it this far, I hope you have been having fun.
Oh, you're getting bored, just booting and installing kernels?  Well,
time for some pedantic things to make you feel that those kernel builds
are actually fun!

Part of the job of being a kernel developer is recognizing the proper
Linux kernel coding style.  The full description of this coding style
can be found in the kernel itself, in the Documentation/CodingStyle
file.  I'd recommend going and reading that right now, it's pretty
simple stuff, and something that you are going to need to know and
understand.  There is also a tool in the kernel source tree in the
scripts/ directory called checkpatch.pl that can be used to test for
adhering to the coding style rules, as kernel programmers are lazy and
prefer to let scripts do their work for them...

And why a coding standard at all?  That's because of your brain (yes,
yours, not mine, remember, I'm just some dumb shell scripts).  Once your
brain learns the patterns, the information contained really starts to
sink in better.  So it's important that everyone follow the same
standard so that the patterns become consistent.  In other words, you
want to make it really easy for other people to find the bugs in your
code, and not be confused and distracted by the fact that you happen to
prefer 5 spaces instead of tabs for indentation.  Of course you would
never prefer such a thing, I'd never accuse you of that, it was just an
example, please forgive my impertinence!

Anyway, the tasks for this round all deal with the Linux kernel coding
style.  Attached to this message are two kernel modules that do not
follow the proper Linux kernel coding style rules.  Please fix both of
them up, and send it back to me in such a way that does follow the
rules.

What, you recognize one of these modules?  Imagine that, perhaps I was
right to accuse you of the using a "wrong" coding style :)

Yes, the logic in the second module is crazy, and probably wrong, but
don't focus on that, just look at the patterns here, and fix up the
coding style, do not remove lines of code.

```



### Coding styles -- what and why?

We all have different styles and preferences in everything in life. Even while writing code, everyone loves to imprint their personalities in the code that brings originality and a sense of ownership and responsibility. 

Coding styles are set of rules or suggestions to write code. The idea behind developing these sets of rules for coding style is very simple - to make code readable by everybody or just to make your brain habitual to a specific style so that you can easily understand the code writen by another person.


The code style depends mainly on the language, few decisions are made depending on the context, and if you switch from one to another. Some of these decisions might be:

- Comments (how and when you use them)
- Tabs or spaces for indentation (the number of spaces is quite important)
- Appropriate naming of variables and functions.
- Code grouping an organization,
- Patterns to be used or avoided.

**But what happens when a whole team is working on one project?** 

Everyone develops a certain style and for individual work, this is great... but when you need more people on your team, understanding different styles could become a problem to everyone.

This has been on my mind a lot lately. For instance, during a code review, I often question whether I should bring specific ways of coding into the discussion or not. How does it affect the application; Is it readable, is it easy to maintain?

Or perhaps I should leave it alone, thinking to myself — *Don’t be picky, it’s just their preference, it’s not a matter of right or wrong.*

Linuc kernel developers has done an amazing job to build a coding style that is followed by mostly everybody who is getting involved in kernel development process.

#### How bad can it go?

If we do not have a properly defined coding style, we might end up with mostly working but very unreadable code.... This can be very unhealty for a mere human being.


Let's take a simple example (In C programming language) to understand the need of coding styles.

```c
#include <stdio.h>

int main() {
    printf("This is a very long line with some garbage text. Lorem ipsum dolor sit amet, consectetur adipiscing elit, sed do eiusmod tempor incididunt ut labore et dolore magna aliqua. Pulvinar neque laoreet suspendisse interdum consectetur libero. Turpis nunc eget lorem dolor sed viverra ipsum nunc. Semper feugiat nibh sed pulvinar proin gravida hendrerit lectus. Turpis egestas sed tempus urna et. Ornare lectus sit amet est placerat. Quam elementum pulvinar etiam non quam. Consequat semper viverra nam libero. Nisl condimentum id venenatis a condimentum vitae. Vitae proin sagittis nisl rhoncus mattis rhoncus urna neque viverra.");
    return 0;
}
```

Above code is very simple in terms of functionality - It prints a huge paragraph. The same code can also be written in the below style... 


```c
#include <stdio.h>

int 
main
() 
{
printf
(
    "This is a very long line with some garbage text. Lorem ipsum dolor sit amet, consectetur adipiscing elit, sed do eiusmod tempor incididunt ut labore et dolore magna aliqua. Pulvinar neque laoreet suspendisse interdum consectetur libero. Turpis nunc eget lorem dolor sed viverra ipsum nunc. Semper feugiat nibh sed pulvinar proin gravida hendrerit lectus. Turpis egestas sed tempus urna et. Ornare lectus sit amet est placerat. Quam elementum pulvinar etiam non quam. Consequat semper viverra nam libero. Nisl condimentum id venenatis a condimentum vitae. Vitae proin sagittis nisl rhoncus mattis rhoncus urna neque viverra."
)
;return 
0;
}
```

I know nobody will ever write code in this style..but it is still a possible option to write a working code. 

Or you might be familiar with the below code.... Spoiler:- This prints a rotating donut. Read [this amazing article](https://www.a1k0n.net/2011/07/20/donut-math.html) to understand the maths behind it.

```c
             k;double sin()
         ,cos();main(){float A=
       0,B=0,i,j,z[1760];char b[
     1760];printf("\x1b[2J");for(;;
  ){memset(b,32,1760);memset(z,0,7040)
  ;for(j=0;6.28>j;j+=0.07)for(i=0;6.28
 >i;i+=0.02){float c=sin(i),d=cos(j),e=
 sin(A),f=sin(j),g=cos(A),h=d+2,D=1/(c*
 h*e+f*g+5),l=cos      (i),m=cos(B),n=s\
in(B),t=c*h*g-f*        e;int x=40+30*D*
(l*h*m-t*n),y=            12+15*D*(l*h*n
+t*m),o=x+80*y,          N=8*((f*e-c*d*g
 )*m-c*d*e-f*g-l        *d*n);if(22>y&&
 y>0&&x>0&&80>x&&D>z[o]){z[o]=D;;;b[o]=
 ".,-~:;=!*#$@"[N>0?N:0];}}/*#****!!-*/
  printf("\x1b[H");for(k=0;1761>k;k++)
   putchar(k%80?b[k]:10);A+=0.04;B+=
     0.02;}}/*****####*******!!=;:~
       ~::==!!!**********!!!==::-
         .,~~;;;========;;;:~-.
             ..,--------,*/
```

Anyways, my point is that the code can become very unreadable and unmaintainable if no coding style guidelines are defined. Early in my journey, I engaged in all kinds of holy wars on code styles. I would read some article about why a particular convention was correct, while another was totally wrong. But I've finally come to a conclusion - **These things don't matter; Consistency and readability matters**.

The development process of linux kernel is very chaotic with thousands of developers working completely remotely on the same code base. How do they do it without messing it all up??

Out of many reasons, one is that all kernel developers try best to follow the code style guidelines laid out in the official kernel documentations [^1]. Here they provide all the necessary details and examples to put their point. And still, it is just a recommendation not forcing anybody to be very specific about it.
[^1]: https://www.kernel.org/doc/html/v4.10/process/coding-style.html

### Fixing style issue problem

In this task, we are provided with 2 different modules (Linux Kernel Modules)... and our job here is to fix the style coding as per the coding style.

You don't have to read the complete coding style to fix it, smart people have already built a tool that can check these issues and provide a warning to us in a clean way. Linux provides a `script/checkpatch.pl` [^4] script, for the same. This script can check the code for trivial style violations in patches and optionally corrects them. This tool can also be used for regular kernel source code files. Read more about this tool from official docs [^2].
[^2]: https://www.kernel.org/doc/html/latest/dev-tools/checkpatch.html
[^4]: https://elixir.bootlin.com/linux/latest/source/scripts/checkpatch.pl

This is not a very unique idea, today we have linters and other tools that can check the style issues and report it or fix it automatically in your source code....in almost all the program languages. You can add one such tool in you `pre-commit` hook or most IDEs have plugins for this, you can use those too.

Anyways, let's pick up a file and run the `checkpatch` tool against it. The code looked fine to me at first... it had a good filename at top and has got indentations. 

```c
/*
* helloworld.c
*/
#include <linux/init.h>
#include <linux/module.h>
#include <linux/kernel.h>

static int hello_init(void)
{
	  pr_debug("Hello World!\n");
	  return 0;
}

static void hello_exit(void)
{
	  pr_debug("See you later.\n");
}

module_init(hello_init);
module_exit(hello_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("ayedaemon");
MODULE_DESCRIPTION("Just a module");
```

But it still does not follow the guidelines. You can check that using the tool - `checkpatch.pl`.

```txt{linenos=false}
## COMMAND -  ../linux/scripts/checkpatch.pl -f helloworld.c

WARNING: Missing or malformed SPDX-License-Identifier tag in line 1
#1: FILE: helloworld.c:1:
+/*

WARNING: It's generally not useful to have the filename in the file
#2: FILE: helloworld.c:2:
+* helloworld.c

WARNING: Block comments should align the * on each line
#2: FILE: helloworld.c:2:
+/*
+* helloworld.c

total: 0 errors, 3 warnings, 24 lines checked
```

There are 3 warnings reported by this script... 

1. Add SPDX licence info in line 1
2. Filename at top is not useful
3. Block comments should have * aligned *(This will be discarded because according to point 2, we are going to remove that comment block)*


Now after fixing the code according to the provided suggestions, we get the below code:

```c
// SPDX-License-Identifier: GPL-2.0+

#include <linux/init.h>
#include <linux/module.h>
#include <linux/kernel.h>

static int hello_init(void)
{
	pr_debug("Hello World!\n");
	return 0;
}

static void hello_exit(void)
{
	pr_debug("See you later.\n");
}

module_init(hello_init);
module_exit(hello_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("ayedaemon");
MODULE_DESCRIPTION("Just a module");
```

In other module file, we have another code that looks good at first but the `checkpatch` reults indicate otherwise.

Code:-
```c
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/delay.h>
#include <linux/slab.h>

int do_work(int *my_int, int retval)
{
	int x;
	int y = *my_int;
	int z;

	for (x = 0; x < *my_int; ++x)
		udelay(10);

	if (y < 10)
		/*
		 * That was a long sleep, tell userspace about it
		 */
		pr_debug("We slept a long time!");
	z = x * y;
	return z;
}

int my_init(void)
{
	int x = 10;

	x = do_work(&x, x);
	return x;
}

void my_exit(void)
{
	return;
}

module_init(my_init);
module_exit(my_exit);
```

This module is somewhat different than what we previously saw. Apart from the regular `init` and `exit` functions, this module has an extra function that is called by `init` function. That's it. You don't have to worry about the code and other logic at this point.. just focus on coding styles.

```txt{linenos=false}
# COMMAND -  ../linux/scripts/checkpatch.pl -f coding_style.c
WARNING: Missing or malformed SPDX-License-Identifier tag in line 1
#1: FILE: coding_style.c:1:
+#include <linux/module.h>

WARNING: void function return statements are not generally useful
#35: FILE: coding_style.c:35:
+	return;
+}

total: 0 errors, 2 warnings, 38 lines checked
```

The script gives us 2 warnings for this module.

1. SPDX licence *(just as the previous one, we'll talk more about it later.)*
2. void function return statements. *(Void functions do not return anything; no return statements are required)*

For this case, I added the SPDX licence comment at line 1 and commented the return statement. If you want, you can remove it entirely from the code. For me, the new code looks like below:-

```C
// SPDX-License-Identifier: GPL-2.0+

#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/delay.h>
#include <linux/slab.h>

int do_work(int *my_int, int retval)
{
	int x;
	int y = *my_int;
	int z;

	for (x = 0; x < *my_int; ++x)
		udelay(10);

	if (y < 10)
		/*
		 * That was a long sleep, tell userspace about it
		 */
		pr_debug("We slept a long time!");
	z = x * y;
	return z;
}

int my_init(void)
{
	int x = 10;

	x = do_work(&x, x);
	return x;
}

void my_exit(void)
{
	// return;
}

module_init(my_init);
module_exit(my_exit);
```


Great!! you are now familiar with the coding style issues in the linux kernel code and has got the ability to fix such issues. If you want to be more automatic with this, you can also use the pre-commit hooks. Read more about git hooks from [githooks.com](https://githooks.com/) [^3]
[^3]: https://githooks.com/