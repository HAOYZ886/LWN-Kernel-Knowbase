---
title: "DazukoFS: a stackable filesystem for virus scanning"
url: https://lwn.net/Articles/318705/
date: "February 11, 2009"
category: "Security-Security technologies"
author: "By Jake Edge February 11, 2009"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jake Edge**  
February 11, 2009

A longstanding out-of-tree kernel feature--used by half-a-dozen or more virus scanners--[Dazuko](<http://dazuko.dnsalias.org/wiki/index.php/Main_Page>) has recently changed its _modus operandi_ in an effort to be included into the mainline. Dazuko, and now [DazukoFS](<https://lwn.net/Articles/317785/>), are mechanisms to control access to files, which are generally used to stop Windows viruses from propagating on Linux servers. The goal is similar in many ways to that of [fsnotify](<https://lwn.net/Articles/311850/#fsnotify>)/[fanotify](<https://lwn.net/Articles/306804/>)/[TALPA](<https://lwn.net/Articles/295568/>), but the DazukoFS implementation as a stackable filesystem is a completely different approach. 

The Dazuko project started almost exactly seven years ago as an effort to allow user-space programs--Windows-style anti-virus scanners mostly--to make file access decisions. One of the reasons to have the scanning in user space--aside from the zero probability of getting one added to the kernel--is to keep it vendor-neutral by not favoring any particular anti-virus engine. But the means to that end was system call hooking, which is a technique that is seriously frowned upon by kernel hackers. Dazuko made an [abortive move to the LSM API](<http://lwn.net/Articles/206075/>), but ran into various problems, including the inability to stack multiple security modules. Eventually, the project started looking at a stackable filesystem as a solution that would be palatable for moving into the mainline. 

Originally suggested for Dazuko by Christoph Hellwig in 2004, a stackable filesystem has a number of advantages over the other solutions. It is a self-contained solution that won't require core kernel code changes if anti-virus developers wish to add new features. It also would add another stackable filesystem to the kernel, which may help foster a more general stackable filesystem framework. But the main reason is that the project sees it as the most likely path into the mainline. Main developer John Ogness [explains](<https://lwn.net/Articles/317785/>): 

Nearly seven years of out-of-tree development were more than enough to prove that out-of-tree kernel drivers have an unnecessarily large maintenance cost (which increases with each new kernel release). With DazukoFS mainline, anti-virus vendors would finally have an official interface and implementation on which to base their online scanning applications. 

DazukoFS is mounted atop an already-mounted filesystem in order to handle file access decisions for files in the underlying filesystem. For example: 

```
mount -t dazukofs /opt /opt
```

sets up the `/opt` filesystem for checking by user-space processes that open a special `/dev` file. All of the scanning application interaction with DazukoFS is done through `/dev` files, all of which is documented in `[Documentation/filesystems/dazukofs.txt](<https://lwn.net/Articles/318741/>)`

File access decisions are made by processes or threads which make up a "group". Groups act as a pool of available scanners to allow multiple outstanding file access decisions. Once the pool is fully occupied, file accesses will block until one becomes available. Groups are registered by writing "`add=MyGroupName`" to `/dev/dazukofs.ctrl`. A group id will then be assigned, which can be parsed from the output of reading the `dazukofs.ctrl` file. Group ids are then used to access the proper device for providing access decisions. 

Based on the group id (N), a `/dev/dazukofs.N` file is created. Each process in the group registers itself by opening that device. It should then block in a read of the device waiting for a file access event. Each event has three pieces of information that are read from the device file: an event id, the process id of the accessing program, and the number of an open file descriptor that can be used to read the contents of the file. The scanning process should then perform whatever actions it requires to make the decision whether to allow or deny the access. 

Because it gets passed an open file descriptor, the scanning process does not need any special privileges beyond those required to access the `/dev/dazukofs*` files. Once it has made the decision, the scanning process writes a string indicating the result to the device. It is then responsible for closing the file descriptor for the accessed file. 

There are a few additional things that can be done via the user-space API: deleting groups, providing for some crash protection within groups, and handling accesses to protected files from within DazukoFS, all of which are described in the `Documentation` file. 

There is also a major caveat that goes with this release of DazukoFS: 

DazukoFS does not support writing to memory mapped files. This should not cause the kernel to crash, but will instead result in the application failing to perform the writes (although mmap() will appear to be successful from the application's viewpoint!). 

That is done, at least partially, to avoid race conditions where a malicious program overwrites the file contents between the scanning and the actual access. This is a general achilles' heel for virus scanning mechanisms, but silently ignoring writes to mapped files is a rather extreme reaction to that problem. TALPA, which has subsequently become fanotify, defines this problem away as not being a part of the threat model it is handling. Perhaps DazukoFS should do something similar. 

It would seem likely that only one of the two proposed solutions for user-space file scanning will end up in the mainline. Ogness mentions fanotify in his patch submission: 

I am aware of the current work of Eric Paris to implement a file access control mechanism within a unified inotify/dnotify framework. Although I welcome any official interface to provide a file access control mechanism for userspace applications in Linux, I feel that DazukoFS provides a more elegant solution. (Note that the two projects do not conflict with each other.) 

So far, there has been no comment on the v2 patch submission, but there were some suggestions to the [first submission](<https://lwn.net/Articles/312716/>) back in December. The kernel filesystem hackers are pretty busy folks in general, but right now there are numerous filesystems in various states of review: btrfs, POHMELFS, DST, FS-Cache, and others. Those may be using up all of the available review bandwidth. Ogness recently [announced](<http://lists.gnu.org/archive/html/dazuko-devel/2009-02/msg00001.html>) that he will be dropping support for the 2.x version of Dazuko--based on system call hooks--to focus on DazukoFS. In it he notes the lack of review: 

As you probably know, DazukoFS has been submitted for inclusion in the mainline Linux kernel. Unfortunately it is getting practically no attention. I do not know if the silence is because I am not CC'ing the correct people, because those people refuse to look at it, or because no one has any time for it. 

From the announcement, it seems clear that Ogness has the patience necessary to shepherd DazukoFS through the kernel inclusion process. It would seem that spending some time working with Eric Paris to try to find some common ground between their two solutions might be time well spent as well.
