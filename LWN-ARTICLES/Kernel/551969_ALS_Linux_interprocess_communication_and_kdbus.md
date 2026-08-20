---
title: "ALS: Linux interprocess communication and kdbus"
url: https://lwn.net/Articles/551969/
date: "May 30, 2013"
category: Message passing; kdbus
author: "By Jake Edge May 30, 2013"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jake Edge**  
May 30, 2013

As part of the developer track at this year's [Automotive Linux Summit Spring](<http://events.linuxfoundation.org/events/automotive-linux-summit-spring>), Greg Kroah-Hartman talked about interprocess communication (IPC) in the kernel with an eye toward the motivations behind [kdbus](<https://github.com/gregkh/kdbus>). The work on kdbus is progressing well and Kroah-Hartman expressed optimism that it would be merged before the end of the year. Beyond just providing a faster D-Bus (which could be accomplished without moving it into the kernel, he said), it is his hope that kdbus can eventually replace Android's binder IPC mechanism. 

#### Survey of IPC

[ ![\[Greg Kroah-Hartman\]](https://static.lwn.net/images/2013/als-gregkh-sm.jpg) ](<https://lwn.net/Articles/552241/>)

There are a lot of different ways to communicate between processes available in Linux (and, for many of the mechanisms, more widely in Unix). Kroah-Hartman strongly recommended Michael Kerrisk's book, [_The Linux Programming Interface_](<http://man7.org/tlpi/>), as a reference to these IPC mechanisms (and most other things in the Linux API). Several of his [slides [PDF]](<https://github.com/gregkh/presentation-kdbus/blob/master/kdbus.pdf?raw=true>) were taken directly from the book. All of the different IPC mechanisms fall into one of three categories, he said: signals, synchronization, or communication. He used diagrams from Kerrisk's book (page 878) to show the categories and their members. 

There are two types of signals in the kernel, standard and realtime, though the latter doesn't see much use, he said. 

Synchronization methods are numerous, including futexes and `eventfd()`, which are both relatively new. Semaphores are also available, both as the "old style" System V semaphores and as "fixed up" by POSIX. The latter come in both named and unnamed varieties. There is also file locking, which has two flavors: record locks to lock a portion of a file and file locks to prevent access to the whole file. However, the code that implements file locking is "scary", he said. Threads have four separate types of synchronization methods (mutex, condition variables, barriers, and read/write locks) available as well. 

For communication, there are many different kernel services available too. For data transfer, one can use pseudo-terminals. For byte-stream-oriented data, there are pipes, FIFOs, and stream sockets. For communicating via messages, there are both POSIX and System V flavored message queues. Lastly, there is shared memory which also comes in POSIX and System V varieties along with `mmap()` for anonymous and file mappings. Anonymous mappings with `mmap()` were not something Kroah-Hartman knew about until recently; they ended up using them in kdbus. 

#### Android IPC

"That is everything we have today, except for Android", Kroah-Hartman said. All of the existing IPC mechanisms were "not enough for Android", so that project added ashmem, pmem, and binder. Ashmem is "POSIX shared memory for the lazy" in his estimation. The Android developers decided to write kernel code rather than user-space code, he said. Ashmem uses virtual memory and can discard memory segments when the system is under memory pressure. Currently, ashmem lives in the staging tree, but he thinks that Google is moving to other methods, so it may get deleted from the tree soon. 

Pmem is a mechanism to share physical memory. It was used to talk to GPUs. Newer versions of Android don't use pmem, so it may also go away. Instead, Android is using the [ION memory allocator](<https://lwn.net/Articles/480055/>) now. 

Binder is "weird", Kroah-Hartman said. It came from BeOS and its developers were from academia. It was developed and used on systems without the System V IPC APIs available and, via Palm and Danger, came to Android. It is "kind of like D-Bus", and some (including him) would argue that Android should have used D-Bus, but it didn't. It has a large user-space library that must be used to perform IPC with binder. 

Binder has a number of serious security issues when used outside of an Android environment, he said, so he stressed that it should _never_ be used by other Linux-based systems. 

In Android, binder is used for [intents](<http://developer.android.com/guide/components/intents-filters.html>) and app separation; it is good for passing around small messages, not pictures or streams of data. You can also use it to pass file descriptors to other processes. It is not particularly efficient, as sending a message makes lots of hops through the library. A [presentation [YouTube]](<http://www.youtube.com/watch?v=NWhyADzgoiI>) at this year's Android Builders Summit showed that one message required eight kernel-to-user-space transitions. 

#### More IPC

A lot of developers in the automotive world have used QNX, which has a nice message-passing model. You can send a message and pass control to another process, which is good for realtime and single processor systems, Kroah-Hartman said. Large automotive companies have built huge systems on top of QNX messages, creating large libraries used by their applications. They would like to be able to use those libraries on Linux, but often don't know that there is a way to get the QNX message API for Linux. It is called [SIMPL](<http://www.icanprogram.com/simpl/>) and it works well. 

Another solution, though it is not merged into the kernel, is [KBUS](<http://code.google.com/p/kbus/>), which was created by some students in England. It provides simple message passing through the kernel, but cannot pass file descriptors. Its implementation involves multiple data copies, but for 99% of use cases, that's just fine, he said. Multiple copies are still fast on today's fast processors. The KBUS developers never asked for it to be merged, as far as he knows, but if they did, there is "no reason not to take it". 

D-Bus is a user-space messaging solution with strong typing and process lifecycle handling. Applications subscribe to messages or message types they are interested in. They can also create an application bus to listen for messages sent to them. It is widely used on Linux desktops and servers, is well-tested, and well-documented too. It uses the operating system IPC services and can run on Unix-like systems as well as Windows. 

The D-Bus developers have always said that it is not optimized for speed. The original developer, Havoc Pennington, created a list of ideas on how to speed it up if that was of interest, but speed was not the motivation behind its development. In the automotive industry, there have been numerous efforts to speed D-Bus up. 

One of those efforts was the [`AF_BUS` address family](<https://lwn.net/Articles/504970/>), which came about because in-vehicle infotainment (IVI) systems needed better D-Bus performance. Collabora was sponsored by GENIVI to come up with a solution and `AF_BUS` was the result. Instead of the four system calls required for a D-Bus message, `AF_BUS` reduced that to two, which made it "much faster". But that solution was rejected by the kernel network maintainers. 

The systemd project rewrote `libdbus` in an effort to simplify the code, but it turned out to significantly increase the performance of D-Bus as well. In preliminary benchmarks, BMW [found [PPT]](<https://github.com/gregkh/presentation-kdbus/blob/master/stuff/EG-SI-IPC%20CommonAPI%20CPP.pptx?raw=true>) that the systemd D-Bus library increased performance by 360%. That was unexpected, but the rewrite did take some shortcuts and listened to what Pennington had said about D-Bus performance. Kroah-Hartman's conclusion is that "if you want a faster D-Bus, rewrite the daemon, don't mess with the kernel". For example, there is a Go implementation of D-Bus that is "really fast". The Linux kernel IPC mechanisms are faster than any other operating system, he said, though it may "fight" with some of the BSDs for performance supremacy on some IPC types. 

#### kdbus

In the GNOME project, there is plan for something called "[portals](<https://docs.google.com/document/d/1QTgxakyUVFMkvr-xFY2Xg9lYjcJLd6kPTl3Ij5_dL7Q/pub>)" that will containerize GNOME applications. That would allow running applications from multiple versions of GNOME at the same time while also providing application separation so that misbehaving or malicious applications could not affect others. Eventually, something like Android's intents will also be part of portals, but the feature is still a long way out, he said. Portals provides one of the main motivations behind kdbus. 

So there is a need for an enhanced D-Bus that has some additional features. At a recent GNOME hackfest, Kay Sievers, Lennart Poettering, Kroah-Hartman, and some other GNOME developers sat down to discuss a new messaging scheme, which is what kdbus is. It will support multicast and single-endpoint messages, without any extra wakeups from the kernel, he said. There will be no blocking calls to kdbus, unlike binder which can sleep, as the API for kdbus is completely asynchronous. 

Instead of doing the message filtering in user space, kdbus will do it in the kernel using [Bloom filters](<http://en.wikipedia.org/wiki/Bloom_filter>), which will allow the kernel to only wake up the destination process, unlike D-Bus. Bloom filters have been publicized by Google engineers recently, and they are an "all math" scheme that uses hashes to make searching very fast. There are hash collisions, so there is still some searching that needs to be done, but the vast majority of the non-matches are eliminated immediately. 

Kdbus ended up with a naming database in the kernel to track the message types and bus names, which "scared the heck out of me", Kroah-Hartman said. But it turned to be "tiny" and worked quite well. In some ways, it is similar to DNS, he said. 

Kdbus will provide reliable order guarantees, so that messages will be received in the order they were sent. Only the kernel can make that guarantee, he said, and the current D-Bus does a lot of extra work to try to ensure the ordering. The guarantee only applies to messages sent from a single process, the order of "simultaneous" messages from multiple processes is not guaranteed. 

Passing file descriptors over kdbus will be supported. There is also a one-copy message passing mechanism that Tejun Heo and Sievers came up with. Heo actually got zero-copy working, but it was "even scarier", so they decided against using it. Effectively, with one-copy, the kernel copies the message from user space directly into the receive buffer for the destination process. Kdbus might be fast enough to handle data streams as well as messages, but Kroah-Hartman does not know if that will be implemented. 

Because it is in the kernel, kdbus gets a number of attributes almost for free. It is namespace aware, which was easy to add because the namespace developers have made it straightforward to do so. It also integrated with the audit subystem, which is important to the enterprise distributions. For D-Bus, getting SELinux support was a lot of work, but kdbus is Linux Security Module (LSM) aware, so it got SELinux (Smack, TOMOYO, AppArmor, ...) support for free. 

#### Current kdbus status

As a way to test kdbus, the systemd team has replaced D-Bus in systemd with kdbus. The code is available in the systemd tree, but it is still a work in progress. The kdbus developers are not even looking at speed yet, but some rudimentary tests suggest that it is "very fast". Kdbus will require a recent kernel as it uses control groups (cgroups); it also requires some patches that were only merged into 3.10-rc kernels. 

The plan is to merge kdbus when it is "ready", which he hopes will be before the end of the year. His goal, though it is not a general project goal, is to replace Android's binder with kdbus. He has talked to the binder people at Google and they are amenable to that, as it would allow them to delete a bunch of code they are currently carrying in their trees. 

Kdbus will not "scale to the cloud", Kroah-Hartman said in answer to a question from the audience, because it only sends messages on a single system. There are already inter-system messaging protocols that can be used for that use case. In addition, the network maintainers placed a restriction on kdbus: don't touch the networking code. That makes sense because it is an IPC mechanism, and that is where `AF_BUS` ran aground. 

The automotive industry will be particularly interested because it is used to using the QNX message passing, which it mapped to `libdbus`. It chose D-Bus because it is well-documented, well-understood, and is as easy to use as QNX. But, it doesn't just want a faster D-Bus (which could be achieved by rewriting it), it wants more: namespace support, audit support, SELinux, application separation, and so on. 

Finally, someone asked whether Linus Torvalds was "on board" with kdbus. Kroah-Hartman said that he didn't know, but that kdbus is self-contained, so he doesn't think Torvalds will block it. Marcel Holtmann said that Torvalds was "fine with it" six years ago when another, similar idea had been proposed. Kroah-Hartman noted that getting it past Al Viro might be more difficult than getting it past Torvalds, but binder is "hairy code" and Viro is the one who found the security problems there. 

Right now, they are working on getting the system to boot with systemd using kdbus. There are some tests for kdbus, but booting with systemd will give them a lot of confidence in the feature. The kernel side of the code is done, he thinks, but they thought that earlier and then Heo came up with zero and one-copy. He would be happy if it is merged by the end of the year, but if it isn't, it shouldn't stretch much past that, and he encouraged people to start looking at kdbus for their messaging needs in the future. 

[ I would like to thank the Linux Foundation for travel assistance so that I could attend the Automotive Linux Summit Spring and LinuxCon Japan. ]
