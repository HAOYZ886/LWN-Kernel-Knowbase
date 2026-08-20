---
title: "Software suspend: and then there were two"
url: https://lwn.net/Articles/94071/
date: "July 21, 2004"
category: Software suspend; pmdisk
author: 
---

Software suspend has long been one of the problem areas in the Linux kernel. Despite multiple available implementations, truly reliable, out-of-the-box support for suspending a system to disk and (crucially) restoring it again is still missing. The return of one long-missing developer may help to improve things, however. 

Pat Mochel is the author of much of the power management and device model code in the 2.6 kernel. At one point in his efforts, his communications with software suspend ("swsusp") maintainer Pavel Machek broke down. In response, Patrick created his own fork of the software suspend code, which he called "pmdisk." The pmdisk code went into the kernel, and a small amount of work was done on it, but then Pat got busy with other things and vanished from the kernel development community. Nobody else was working on pmdisk, so the effort simply stalled. Pavel has discussed its removal from the kernel more than once, but that has not ever happened. 

Just in time for the Kernel Summit, Pat returned with [a 25-part patch set](<https://lwn.net/Articles/94063/>). Pat now believes that he made a mistake by forking the software suspend code, and is trying to make up. So his patch set removes pmdisk from the 2.6 kernel - but not before merging its best parts into the existing swsusp code base. With this patch set, swsusp is significantly cleaned up and more firmly integrated into the kernel's power management subsystem. This code base, Pat hopes, will prove a good starting place for further work toward respectable software suspend support. 

There is one other player in this game, however: the swsusp2 work done by Nigel Cunningham and others. This code, which forked from swsusp some time ago, exists as a out-of-tree patch. It is, however, by many accounts, the most featureful and reliable software suspend implementation available for Linux. Swsusp2 offers a more polished display, the ability to abort the suspend operation, and more. Nigel has recently been making noises about trying to merge swsusp2 into the 2.6 mainline. 

The last time this topic came up, there was a significant amount of resistance. All versions of swsusp feature a "refrigerator," which is a mechanism for cooling off all processes in the system before suspending the system itself. The swsusp2 refrigerator has seen significant amounts of work intended to keep the system from refrigerating processes which might still be needed by other parts of the system before it is suspended. The result is a large number of macro calls interspersed through the rest of the kernel marking places where a process should not be refrigerated. These changes make the swsusp2 patch relatively intrusive; they also create a new kind of critical section within the kernel which looks hard to maintain over the long run. 

The current feeling, as reflected at the kernel summit, is that much of Nigel's work cannot be merged in its current form. It also needs to be split into a set of small, incremental patches before it can be considered. Hopefully this work will happen, however; swsusp2 has things to offer. If its best features can be merged in with swsusp, perhaps the kernel may yet move from three unreliable software suspend implementations to a single version which actually works.
