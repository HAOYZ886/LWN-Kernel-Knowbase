---
title: In brief
url: https://lwn.net/Articles/348357/
date: "August 26, 2009"
category: "Workqueues-Lazy"
author: "By Jonathan Corbet August 26, 2009"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
August 26, 2009

**What is direct I/O, really?** Linux, like many operating systems, supports direct I/O operations to block devices. But how, exactly, should programmers expect direct I/O to work? As [a recent document](<https://lwn.net/Articles/348739/>) posted by Ted Ts'o notes, there is no real specification for what direct I/O means: 

It is not a part of POSIX, or SUS, or any other formal standards specification. The exact meaning of O_DIRECT has historically been negotiated in non-public discussions between powerful enterprise database companies and proprietary Unix systems, and its behaviour has generally been passed down as oral lore rather than as a formal set of requirements and specifications. 

Ted's document is an attempt to better specify what is really going on when a process requests a direct I/O operation. It is currently focused on the ext4 filesystem, but the hope is to forge a consensus among Linux filesystem developers so that consistent semantics can be obtained on all filesystems. 

**Can you thaw out TuxOnIce?** TuxOnIce is the perennially out-of-tree hibernation implementation. It has a number of nice features which are not available with the mainstream version; these features have never managed to get into a form where they could be merged. TuxOnIce developer Nigel Cunningham has recently [concluded](<https://lwn.net/Articles/348741/>) that it looks like this merger is not going to happen because the relevant people are simply too busy. He says: 

Given that this has been the outcome so far, I see no reason to imagine that we're going to make any serious progress any time soon. 

In response, he is now actively looking for developers who would like to take on the task of getting TuxOnIce (or, at least, parts of it) into the mainline. He has put together [a "todo" list](<https://lwn.net/Articles/348743/>) for potentially interested parties. 

**Lazy workqueues**. Kernel developers have been concerned for years that the number of kernel threads was growing beyond reason; see, for example, [this article](<http://lwn.net/Articles/229873/>) from 2007\. Jens Axboe recently became concerned himself when he noticed that his system (a modest 64-processor box) had 531 kernel threads running on it. Enough, he decided, was enough. 

His response was the [lazy workqueue](<http://lwn.net/Articles/347822/>) concept. As might be expected, this patch is an extension of the workqueue mechanism. A "lazy" workqueue can be created with `create_lazy_workqueue()`; it will be established with a single worker thread. Unlike single-threaded workqueues, though, lazy workqueues still try to preserve the concept of dedicated, per-CPU worker threads. Whenever a task is submitted to a lazy workqueue, the kernel will direct it toward the thread running on the submitting CPU; if no such thread exists, the kernel will create it. These threads will exit if they are idle for a sufficient period. 

The end result was a halving of the number of kernel threads on Jens's system. That still seems like too many threads, but it's a good step in the right direction. 

**Embedded x86**. Thomas Gleixner started [his patch series](<http://lwn.net/Articles/348244/>) with a note that the "embedded nightmare" has finally come to the x86 architecture. The key development here is a new set of patches intended to support Intel's new "Moorestown" processor series; these patches added a bunch of code to deal with the new quirks in this processor. Rather than further clutter the x86 architecture code, Thomas decided that it was time for a major cleanup. 

The result is a new, global `platform_setup` structure designed to tell the architecture code how to set up the current processor. It includes a set of function pointers which handle platform-specific tasks like locating BIOS ROMs, setting up interrupt handling, initializing clocks, and much more; it is a 32-part patch in all. This new structure is able to encapsulate many of the initialization-time differences between the 32-bit and 64-bit x86 architectures, the new "Moorestown" architecture, and various virtualized variants as well. It is also runtime-configurable, so a single kernel should be able to run efficiently on any of the supported systems. 

**O_NOSTD**. Longstanding Unix practice dictates that applications are started with the standard input, output, and error I/O streams on file descriptors 0, 1, and 2, respectively. The assumption that these file descriptors will be properly set up is so strong that most developers never think to check them. So interesting things can happen if an application is run with one or more of the standard file descriptors closed. 

Consider, for example, running a program with file descriptor 2 closed. The next file the program opens will be assigned that descriptor. If something then causes the program to write to (what it thinks is) the standard error stream, that output will, instead, go to the other file which had been opened, probably corrupting that file. A malicious user can easily make messes this way; when setuid programs are involved, the potential consequences are worse. 

There are a number of ways to avoid falling into this trap. An application can, on startup, ensure that the first three file descriptors are open. Or it can check the returned file descriptor from `open()` calls and use `dup()` to change the descriptor if need be. But these options are expensive, especially considering that, almost all of the time, the standard file descriptors are set up just as they should be. 

Eric Blake has proposed a new alternative in the form of the [`O_NOSTD`](<http://lwn.net/Articles/348331/>) flag. The semantics are simple: if this flag is provided to an `open()` call, the kernel will not return one of the "standard" file descriptors. If this patch goes in (and there does not seem to be any opposition to that), application developers will be able to use it to ensure that they are not getting any file descriptor surprises without additional runtime cost. 

There is a cost, of course, in the form of a non-standard flag that will not be supported on all platforms. One could almost argue that it would be better to add a specific flag for cases where a file descriptor in the [0..2] range is desired. But that would be a major ABI change to say the least; it's not an idea that would be well received. 

**Linux-ARM mailing lists**. Russell King has [announced](<https://lwn.net/Articles/348043/>) that the ARM-related mailing lists on arm.linux.kernel.org will be shut down immediately. He is, it seems, not happy about some of the criticism he has received about the operation of those lists. So the lists will be moving, though exactly where is not entirely clear. David Woodhouse has [created a new set of lists](<https://lwn.net/Articles/348044/>) on infradead; he appears to have moved the subscriber lists over as well. There is also a push to [move the list traffic to vger](<https://lwn.net/Articles/348046/>), but the preservation of the full set of lists and their subscribers suggests that the infradead lists are the ones which will actually get used.
