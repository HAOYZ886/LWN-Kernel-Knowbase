---
title: The UAPI header file split
url: https://lwn.net/Articles/507794/
date: "July 25, 2012"
category: "Include files; User-space API"
author: "By Michael Kerrisk July 25, 2012"
---

By **Michael Kerrisk**  
July 25, 2012

Patches that add new software features often gain the biggest headlines in free software projects. However, once a project reaches a certain size, refactoring work that improves the overall maintainability of the code is arguably at least as important. While such work does not improve the lives of users, it certainly improves the lives of developers, by easing later work that _does_ add new features. 

With around 15 million lines of code (including 17,161 `.c` files and 14,222 `.h` files) in the recent 3.5 release, the Linux kernel falls firmly into the category of projects large enough that periodic refactoring is a necessary and important task. Sometimes, however, the sheer size of the code base means that refactoring becomes a formidable task--one that verges on being impossible if attempted manually. At that point, an enterprising kernel hacker may well turn to writing code that refactors the kernel code. David Howell's UAPI patch series, which has been proposed for inclusion during the last few kernel merge windows, was created using such an approach. 

The UAPI patchset was motivated by David's [observation](<https://lwn.net/Articles/507832/>) that when modifying the kernel code: 

I occasionally run into a problem where I can't write an inline function in a header file because I need to access something from another header that includes this one. Due to this, I end up writing it as a #define instead. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

He went on to elaborate that this problem of "inclusion recursion" in header files typically occurs with inline functions: 

Quite often it's a case of an inline function in header A wanting a struct [or constant or whatever] from header B, but header B already has an inline function that wants a struct from header A. 

As is the way of such things, a small itch can lead one to thinking about more general problems, and how to solve them, and David has devised a grand nine-step plan of changes to achieve his goals, of which the current patch set is just the first step. However, this step is, in terms of code churn, a big one. 

What David wants to do is to split out the user-space API content of the kernel header files in the `include` and `arch/xxxxxx/include` directories, placing that content into corresponding headers created in new `uapi/` subdirectories that reside under each of the original directories. As well as being a step toward solving his original problem and performing a number of other useful code cleanups, David notes that disintegrating the header files has many other benefits. It simplifies and reduces the size of the kernel-only headers. More importantly, splitting out the user-space APIs into separate headers has the desirable consequence that it ""simplifies the complex interdependencies between headers that are [currently] partly exported to userspace"". 

There is one other benefit of the UAPI split that may be of particular interest to the wider Linux ecosystem. By placing all of the user-space API-related definitions into files dedicated solely to that task, it becomes easier to track changes to the APIs that the kernel presents to user space. In the first instance, these changes can be discovered by scanning the git logs for changes in files under the `uapi/` subdirectories. Easing the task of tracking user-space APIs would help many other parts of the ecosystem, for example, C library maintainers, scripting language projects that maintain language bindings for the user-space API, testing projects such as [LTP](<http://ltp.sourceforge.net/>), documentation projects such as [man-pages](<http://www.kernel.org/doc/man-pages/>), and perhaps even LWN editors preparing summaries of changes in the merge window that occurs at the start of each kernel release cycle. 

The task of disintegrating each of the header files into two pieces is in principle straightforward. In the general case, each header file has the following form: 

```
/* Header comments (copyright, etc.) */
    
        #ifndef _XXXXXX_H     /* Guard macro preventing double inclusion */
        #define _XXXXXX_H
    
        [User-space definitions]
    
        #ifdef __KERNEL__
    
        [Kernel-space definitions]
    
        #endif /* __KERNEL__ */
    
        [User-space definitions]
      
        #endif /* End prevent double inclusion */
```

Each of the above parts may or may not be present in individual header files, and there may be multiple blocks governed by `#ifdef __KERNEL__` preprocessor directives. 

The part of this file that is of most interest is the code that falls inside the outermost `#ifndef` block that prevents double inclusion of the header file. Everything inside that block that is _not_ nested within a block governed by a `#ifdef __KERNEL__` block should move to the corresponding `uapi/` header file. The content inside the `#ifdef __KERNEL__` block remains in the original header file, but the `#ifdef __KERNEL__` and its accompanying `#endif` are removed. 

A copy of the header comments remains in the original header file, and is duplicated in the new `uapi/` header file. In addition, a `#include` directive needs to be added to the original header file so that it includes the new `uapi/` header file, and of course a suitable `git commit` message needs to be supplied for the change. 

The goal is to modify the original header file to look like this: 

```
/* Header comments (copyright, etc.) */
    
        #ifndef _XXXXXX_H     /* Guard macro preventing double inclusion */
        #define _XXXXXX_H
    
        #include <include/uapi/path/to/header.h>
    
        [Kernel-space definitions]
    
        #endif /* End prevent double inclusion */
```

The corresponding `uapi/` header file will look like this: 

```
/* Header comments (copyright, etc.) */
    
        #ifndef _UAPI__XXXXXX_H     /* Guard macro preventing double inclusion */
        #define _UAPI__XXXXXX_H
    
        [User-space definitions]
    
        #endif /* End prevent double inclusion */
```

Of course, there are various details to handle in order to correctly automate this task. First of all, sometimes the script should produce only one result file. If there is no `#ifdef __KERNEL__` block in the original header, the original header file is in effect renamed to the `uapi/` file. Where the header file is disintegrated into two files, there are many other details that need to be handled. For example, if there are `#include` directives that are retained at the top of the original header file, then the `#include` for the generated `uapi/` file should be placed _after_ those `#include` directives (in case the included `uapi/` file has dependencies on them). Furthermore, there may be pieces of the original header that are explicitly _not_ intended for kernel space (i.e., they are for user-space only)--for example, pieces governed by `#ifndef __KERNEL__`. Those pieces should migrate to the `uapi/` file, retaining the guarding `#ifndef __KERNEL__`. 

David's scripts handle all of the aforementioned details, and many others as well, including making corresponding changes to `.c` source files and various kernel build files. Naturally, no scripting can correctly handle all possible cases in human-generated files, so part of the current patch set includes pre-patches that add markers to "coach" the scripts to do the right thing in those cases. 

Writing scripts to automate this sort of task becomes a major programming project in its own right, and the shell and Perl scripts ([.tar.xz archive](<http://people.redhat.com/~dhowells/uapi.tar.xz>)) to accomplish the task run total more than 1800 lines. (Using scripting to generate the patch set has the notable benefit that the patch set can be automatically refreshed as the relevant kernel source files are changed by other kernel developers. Given that the UAPI patches touch a large number of files, this is an important consideration.) 

Knowing the size of those scripts, and the effort that must have been required to write them, gives us a clue that the scale of the actual changes to the kernel code must be large. And indeed they are. In its current incarnation, the UAPI patch series consists of 74 commits, of which 65 are scripted (the scripted changes produce commits to the kernel source tree on a per-directory basis). Altogether, the patches touch more than 3500 files, and the `diff` of the changes amounts to over 300,000 lines. 

The scale of these changes brings David to his next problem: how to get the changes accepted by Linus. The problem is that it's impossible to manually review source code changes of this magnitude. Even a partial review would require considerable effort, and would not provide ironclad guarantees about the remaining unreviewed changes. In the absence of such reviews, when Linus received David's [request](<http://thread.gmane.org/gmane.linux.kernel/1273854>) to pull these patches in the 3.5 merge window, he employed a time-honored strategy: the request was ignored. 

Although David first started working on these changes around [a year ago](<http://thread.gmane.org/gmane.linux.kbuild.devel/7123>), Linus has not to date directly commented on them. However, back in January Linus [accepted](<http://git.kernel.org/?p=linux/kernel/git/torvalds/linux-2.6.git;a=commit;h=dca88ad6915b65f6e037f8c3e632fcd92a70bd88>) some [preparatory patches](<http://thread.gmane.org/gmane.linux.kernel.cross-arch/11863>) for the UAPI work, which suggests that he's at least aware of the proposal and possibly willing to entertain it. Other kernel developers have expressed support for the UAPI split ([1](<http://lists.linux-foundation.org/pipermail/ksummit-2012-discuss/2012-June/000470.html>) and [2](<http://marc.info/?l=linux-arch&m=131068042616067&w=2>)). However, probably because of the magnitude of the changes, getting actual reviews and `Acked-by:` tags has to date proved to be a challenge. Given the impossibility of a complete manual review of the changes, the best hope would seem to be to have other developers review the conceptual approach employed by David's scripts, possibly review the scripts themselves, perform a review of a sample of the changed kernel source files, and perform kernel builds on as many different architectures as possible. (Aspiring kernel hackers might note that much of the review task on this quite important piece of kernel work does _not_ require deep understanding of the workings of the kernel.) 

Getting sufficient review of any set of kernel patches, let alone a set this large, is a perennial difficulty. Things at least took a step forward with David's request to Linus to have the patches accepted for the currently open 3.6 merge window, when Arnd Bergmann [provided his `Acked-by:`](<https://lwn.net/Articles/508075/>) for the entire patch series. Whether that will prove enough, or whether Linus will want to see formal agreement from additional developers before accepting the patches is an open question. If it proves insufficient for this merge window, then perhaps a rethink will be required next time around about how to have such a large change accepted into the mainline kernel.
