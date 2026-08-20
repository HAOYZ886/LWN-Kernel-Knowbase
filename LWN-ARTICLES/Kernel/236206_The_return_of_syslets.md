---
title: The return of syslets
url: https://lwn.net/Articles/236206/
date: "May 30, 2007"
category: Asynchronous IO; Events reporting; Kevent; Syslets
author: 
---

Things have been quiet on the syslet/threadlet/fibril front for some time. Part of the reason for that, it would seem, is that Ingo Molnar has been busy with the completely fair scheduler work and has not been able to get back to this other little project. This work is not dead, though; instead it has been picked up by Zach Brown (who came up with the original "fibril" concept). Zach has released [an updated patch](<http://lwn.net/Articles/236151/>) bringing this work back to the foreground. He has not made a whole lot of changes to the syslet code - yet - but that does not mean that the patch is uninteresting. 

Zach's motivation for this work, remember, was to make it easier to implement and maintain proper asynchronous I/O (AIO) support in the kernel. His current work [continues](<https://lwn.net/Articles/236207/>) toward that goal: 

For the time being I'm focusing on simplifying the mechanisms that support the sys_io_*() interface so I never ever have to debug fs/aio.c (also known as chewing glass to those of us with the scars) again. 

In particular, [one part of the new syslet patch](<http://oss.oracle.com/~zab/syslets/v6/aio-issue-syslets.patch>) is a replacement for the `io_submit()` system call, which is the core of the current AIO implementation. Rather than start the I/O and return, the new `io_submit()` uses the syslet mechanism, eliminating a lot of special-purpose AIO code in the process. Zach's stated goal is to get rid of the internal `kiocb` structure altogether. The current code is more of a proof of concept, though, with a lot of details yet to fill in. Some [benchmarks](<https://lwn.net/Articles/236212/>) have been posted, though, as Zach says, ""They haven't wildly regressed, that's about as much as can be said with confidence so far."" It is worth noting that, with this patch, the kernel is able to do asynchronous _buffered_ I/O through `io_submit()`, something which the mainline has never yet supported. 

The biggest area of discussion, though, has been over Jeff Garzik's suggestion that the kevent code should be integrated with syslets. Some people like the idea, but others, [including Ingo](<https://lwn.net/Articles/236216/>), think that kevents do not provide any sort of demonstrable improvement over the current epoll interface. Ulrich Drepper, the glibc maintainer, [disagreed](<https://lwn.net/Articles/236220/>) with that assessment, saying that the kevent interface was a step in the right direction if it does not perform any better. 

The reasoning behind that point of view is worth a look. The use of the epoll interface requires the creation of a file descriptor. That is fine when applications use epoll directly, but it can be problematic if glibc is trying to poll for events (I/O completions, say) that the application does not see directly. There is a single space for file descriptors, and applications often think they know what should be done with every descriptor in that space. If glibc starts creating its own private file descriptors, it will find itself at the mercy of any application which closes random descriptors, uses `dup()` without care, etc. So there is no way for glibc to use file descriptors independently from the application. 

Possible solutions exist, such as giving glibc a set of private, hidden descriptors. But Ulrich would rather just go with a memory-based interface which avoids the problem altogether. And Linus [would rather not create any new interfaces](<https://lwn.net/Articles/236222/>) at all. All told, it has the feel of an unfinished discussion; we'll be seeing it again.
