---
title: "Eudyptula Task 7"
date: 2023-05-01T02:32:12+05:30
draft: false
# showtoc: false
tags: [Eudyptula, linux]
series: [Eudyptula]
description: Task 7 for Eudyptula challenge
# math: true
# ShowBreadCrumbs: false
# searchHidden: true
# hideMeta: true
---



```{linenos=false}
This is Task 07 of the Eudyptula Challenge
------------------------------------------

Great work with that misc device driver.  Isn't that a nice and simple
way to write a character driver?

Just when you think this challenge is all about writing kernel code,
this task is a throwback to your second one.  Yes, that's right,
building kernels.  Turns out that's what most developers end up doing,
tons and tons of rebuilds, not writing new code.  Sad, but it is a good
skill to know.

The tasks this round are:
  - Download the linux-next kernel for today.  Or tomorrow, just use
    the latest one.  It changes every day so there is no specific one
    you need to pick.  Build it.  Boot it.  Provide proof that you built
    and booted it.

What is the linux-next kernel?  Ah, that's part of the challenge.

For a hint, you should read the excellent documentation about how the
Linux kernel is developed in Documentation/development-process/ in the
kernel source itself.  It's a great read, and should tell you all you
never wanted to know about what Linux kernel developers do and how they
do it.
```


# What is Linux??

Before jumping on Linux-next...let's start with an overview of the Linux kernel and its significance within the Linux operating system.

Linux is an open-source operating system kernel that was initially developed by Linus Torvalds in 1991. The kernel serves as the core component of the Linux operating system, providing essential functionalities and acting as an intermediary between the hardware and the software layers.

The Linux kernel plays a crucial role in managing system resources, handling hardware devices, and providing a foundation for software applications to run efficiently. Some of its key responsibilities include process management, memory management, device driver handling, file system management, and networking.

One of the defining characteristics of Linux is its open-source nature. This means that the source code of the kernel is freely available to the public, allowing individuals and communities to examine, modify, and distribute it according to their needs. This openness has fostered a vibrant ecosystem of developers who collaborate to improve and enhance the kernel's capabilities.

Linux is renowned for its stability, security, and scalability. Its robust design and efficient resource management make it suitable for a wide range of computing devices, from small embedded systems and smartphones to large-scale servers and supercomputers. Moreover, Linux serves as the foundation for numerous Linux distributions, which are complete operating systems that package the Linux kernel with additional software and user-friendly interfaces.

By harnessing the power of open-source collaboration, Linux has grown into a highly versatile and widely adopted operating system, powering various domains such as enterprise computing, cloud infrastructure, scientific research, mobile devices, and more. Its flexibility, reliability, and vast community support make it an attractive choice for individuals, businesses, and organizations seeking a powerful and customizable operating system.

Now that we have a basic understanding of Linux and its kernel, we can delve into the concept of Linux-next and its significance within the development process.

## Big picture of linux development

The development of the Linux kernel in the open-source world is a remarkable (somewhat scary for me) and dynamic process that involves collaboration among thousands of developers worldwide. Here's a big picture overview of the Linux kernel development process:

1. **Collaboration and Community**: The Linux kernel development thrives on collaboration and community engagement. It is led by Linus Torvalds, the original creator of Linux, along with a core group of maintainers who oversee different subsystems of the kernel. The development process follows a meritocratic model where contributions are reviewed and integrated based on their technical merits.

2. **Patch Submission**: Developers from diverse backgrounds and organizations contribute to the Linux kernel. They propose changes, enhancements, bug fixes, and new features in the form of patches. These patches are submitted to the relevant subsystem maintainers or mailing lists for review.

3. **Review and Feedback**: The submitted patches undergo a thorough review process, where experienced developers provide feedback, suggestions, and technical guidance. The review process ensures that the proposed changes align with the kernel's standards, maintain compatibility, and adhere to best practices.

4. **Iterative Development**: Developers iterate on their patches based on the feedback received during the review process. They make necessary modifications, address concerns, and refine their code to meet the kernel's quality standards.

5. **Testing and Integration**: Once the patches are considered ready, they are tested extensively. Various testing frameworks, such as the Linux Test Project (LTP) and KernelCI, are used to ensure that the changes do not introduce regressions and maintain stability. The patches are then integrated into the "mainline" development branch.

6. **Mainline Integration**: The mainline development branch is where the accepted patches are merged into the official Linux kernel source code. Linus Torvalds oversees this process and has the final authority to accept or reject patches based on technical considerations. The mainline branch represents the most up-to-date version of the Linux kernel and serves as the basis for future releases.

7. **Stable Releases and Long-Term Support**: The Linux kernel follows a time-based release model, with new stable versions being released approximately every two to three months. These releases incorporate the accumulated changes from the mainline branch, undergo further testing, and receive bug fixes. Additionally, long-term support (LTS) versions are maintained for an extended period to ensure stability and compatibility for enterprise and embedded systems.

8. **Ecosystem and Distribution**: The Linux kernel forms the foundation for numerous Linux distributions. These distributions package the kernel with additional software, libraries, and user interfaces to create complete operating systems suitable for various use cases. The distributions play a crucial role in making Linux accessible to a wide range of users, providing installation, customization, and support options.


The Linux kernel development follows a patch cycle that involves several release candidate (RC) versions before a stable release is made. Here's an overview of the Linux patch cycle and RC versions:

1. **Development Cycle**: The development cycle begins after a stable release is made. During this cycle, new features, enhancements, bug fixes, and improvements are introduced into the Linux kernel.

2. **Patch Submission**: Developers submit patches for review and inclusion in the next kernel release. These patches can come from individual contributors, companies, or organizations.

3. **Mainline Integration**: The submitted patches go through a review process, where they are examined for technical correctness, adherence to coding standards, and compatibility with existing code. Accepted patches are integrated into the mainline development branch of the Linux kernel.

4. **Merge Window**: The merge window is a specific period at the beginning of the development cycle when major changes and new features are merged into the mainline development branch. During this time, the Linux kernel developers are more open to accepting substantial modifications.

5. **Release Candidates (RC)**: After the merge window closes, the Linux kernel development enters the release candidate phase. Release candidates are pre-release versions of the kernel that undergo testing and further refinement before the final stable release. Each release candidate is identified by the tag `-rc<number>`.

   - RC1: The first release candidate marks the beginning of the testing phase for the upcoming release. It includes the merged changes from the merge window.
   - RC2, RC3, and so on: Successive release candidates incorporate additional bug fixes, improvements, and patches that are considered stable enough to be tested.

6. **Testing and Bug Fixing**: During the release candidate phase, extensive testing is performed by developers, testers, and the wider community. Any bugs, regressions, or issues discovered during this testing period are addressed and fixed in subsequent release candidates.

7. **Stable Release**: Once the release candidates have undergone sufficient testing and the kernel is deemed stable, the final stable release is made. The stable release incorporates all the accepted changes and bug fixes from the release candidates.

8. **Long-Term Support (LTS) Releases**: In addition to the regular stable releases, certain versions of the Linux kernel are designated as Long-Term Support (LTS) releases. These LTS versions receive extended maintenance and bug fix support for a specified period to ensure stability and compatibility for enterprise and embedded systems.


Here is how the 5.4 development cycle went:

|  Date  |  Release  |
|-------------|---------------------|
| September 15, 2019 | 5.3 stable release | 
| September 30, 2019 | 5.4-rc1, merge window closes | 
| October 6, 2019 | 5.4-rc2 | 
| October 13, 2019 | 5.4-rc3 |
| October 20, 2019 | 5.4-rc4 |
| October 27, 2019 | 5.4-rc5 |
| November 3, 2019 | 5.4-rc6 |
| November 10, 2019 | 5.4-rc7 |
| November 17, 2019 | 5.4-rc8|
| November 24, 2019 | 5.4 stable release |

*Table taken from kernel.org/doc*

## Next trees

"Linux Next" refers to a specific development branch of the Linux kernel. The Linux Next branch serves as a staging area for upcoming changes and new features that are planned for inclusion in future versions of the Linux kernel. It acts as a testing ground where patches from different developers and subsystems are integrated and tested together.

The purpose of the Linux Next branch is to catch any potential conflicts or issues that may arise when combining different changes before they are merged into the mainline Linux kernel. By testing these changes in advance, it helps ensure the overall stability and quality of the Linux kernel.

The Linux Next branch is maintained by the Linux Next project, which is led by Stephen Rothwell and is supported by several Linux kernel developers. It provides a convenient way for developers to collaborate, test, and integrate their changes before they are submitted for inclusion in the mainline kernel.



There is a great [Youtube video](https://www.youtube.com/watch?v=vyenmLqJQjs) [^gkh_video] where Greg Kroah-Hartman explains the whole development workflow. **A MUST WATCH!!**

[^gkh_video]: https://www.youtube.com/watch?v=vyenmLqJQjs



# Working with linux-next

Now that we understand how the Linux development process works, we can see where the `-next` trees fall into the overall process. Let's look at how these trees can be used to contribute to kernel development.

## Initial setup

- Clone the linux tree

```
git clone https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git
```

- Add a `linux-next` remote.

```
cd linux
git remote add linux-next https://git.kernel.org/pub/scm/linux/kernel/git/next/linux-next.git
```

- Fetch the changes and tags from `linux-next`.

```
git fetch linux-next
git fetch --tags linux-next
```

## Regular tracking

Linux `-next` trees are built every day... so you need to update it everyday before your work to make sure that you are not working on an older code base.

```
git checkout master
git remote update
```

- Check newer tags
```
git tag -l "next-*" | tail
```

- Checkout a new branch from the branch you want to work 
```
git checkout -b my_local_branch next-20230427
```


Now the next steps are very simple and straight-forward.

- Make changes to the code base.
- Test it with `./scripts/checkpatch.pl` script for any issues which you need to solve before submitting it as a patch.
- Check `git status` and `git diff` before making a commit.
- Make a patch using `git format-patch` command.
- Now you have a well tested and formatted patch, all you need to do is find the maintainer using `./scripts/get_maintainer.pl` script and then send the patch to all those people using `git send-mail`.

***Note:** All of the above-discussed tools and techniques are completely optional; this is just what I would do.*


# Resources
- [Linux development subscription lists (mails)](http://vger.kernel.org/vger-lists.html)
- [All linux trees (git)](https://git.kernel.org/)
- [Linux-next man page](https://www.kernel.org/doc/man-pages/linux-next.html)
- [Linux kernel development process (official doc)](https://www.kernel.org/doc/html/latest/process/2.Process.html)
- [Youtube: Linux Kernel Development, Greg Kroah-Hartman - Git Merge 2016](https://www.youtube.com/watch?v=vyenmLqJQjs)
