---
title: "The ZUFS zero-copy filesystem"
url: https://lwn.net/Articles/756625/
date: "June 6, 2018"
category: "Filesystems-Nonvolatile memory"
author: "By Jake Edge June 6, 2018 LSFMM"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jake Edge**  
June 6, 2018

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2018/>)

At the 2018 Linux Storage, Filesystem, and Memory-Management Summit (LSFMM), Boaz Harrosh presented his zero-copy user-mode filesystem (ZUFS). It is both a filesystem in its own right and a framework similar to FUSE for implementing filesystems in user space. It is geared toward extremely low latency and high performance, particularly for systems using persistent memory. 

Harrosh began by saying that the idea behind his talk is to hopefully entice others into helping out with ZUFS. There are lots of "big iron machines" these days, some with extremely fast I/O paths (e.g. NVMe over fabrics with throughput higher than memory). "For some reason" there may be a need to run a filesystem in user space but the current interface is slow because "everyone is copy happy", he said. 

[ ![\[Boaz Harrosh\]](https://static.lwn.net/images/2018/lsf-harrosh-sm.jpg) ](<https://lwn.net/Articles/756686/>)

Al Viro asked if Harrosh had looked at [OrangeFS](<https://lwn.net/Articles/643165/>), which can share its pages with a user-space component. Harrosh said that he had worked with OrangeFS in the past, but that it has "nowhere near the performance" he is seeking. 

He is focused on copies. If a system mainly uses NVDIMMs (i.e. persistent memory), the memory bandwidth should be used for storage. So ZUFS is "very strict that nothing is copied anywhere". Anything that can be accessed via a pointer to persistent memory will be; "even metadata is zero copy". 

He showed some system diagrams of ZUFS that are similar to those in his 2017 Linux Plumbers Conference [slides [`.pptx`]](<http://linuxplumbersconf.org/2017/ocw//system/presentations/4703/original/ZUFS_for_LinuxPlumbers_LPC.pptx>). There is one kernel component, the ZU Feeder (ZUF), that feeds remote procedure calls (RPCs) from applications to the ZU Server (ZUS), which lives in user space. ZUS can have various `.so` files linked to it that implement different filesystems using the framework; some of those might be proprietary. ZUF is released under the GPL, while ZUS is BSD licensed. 

There are multiple ZUFS threads (ZTs), each with an affinity to a single core. Each ZT is dedicated to a particular application; there is no shared information between the ZTs and ZUS, so no locks are needed. A 4MB per-CPU zero-copy region (ZT-vma) is shared between the application and the ZT, so each CPU has its own area that can be used to communicate between the server and the application. 

For a write operation, the application maps its buffers into its per-CPU ZT-vma and initiates the operation. ZT gets the pointer and length and does a `memcpy()` from the ZT-vma data to persistent memory. For a read, the application maps buffers to hold the data and a ZT fills them from the persistent memory. It supports multiple applications, with "not a lock in sight". 

The in-kernel portion of ZUFS includes a ZUF-root, which is a mini-filesystem that allows the normal `mount` command to be used. The kernel will have knowledge of the filesystem types and mounts, but the filesystems are really mounted in user space. ZUS is a thin layer that implements VFS operations. It uses direct I/O by default, but can optionally use the page cache. 

ZUFS is a zero-copy replacement for FUSE. It sacrifices some of the security of FUSE because it does not have a server per filesystem, but the API for ZUFS is simpler than FUSE. It also does not rely on copying, as FUSE does, of course. 

He was looking for feedback, but a whirlwind tour of a new filesystem with a lot of differences from the usual fare may have been a bit overwhelming; there were not too many comments on any "big holes" that attendees saw. He said that there is a complete filesystem implementation at this point, only missing extended attributes (xattrs) and access-control lists (ACLs); it can run xfstests and is "pretty stable". It does, however, take some shortcuts; that means the server has a lot of ways to crash the kernel, which Viro called a "non-starter" in terms of getting it merged.
