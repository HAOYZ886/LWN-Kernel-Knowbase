---
title: Taking ZUFS upstream
url: https://lwn.net/Articles/787630/
date: "May 8, 2019"
category: "Filesystems-Nonvolatile memory"
author: "By Jake Edge May 8, 2019 LSFMM"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jake Edge**  
May 8, 2019

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2019/>)

At the 2018 Linux Storage, Filesystem, and Memory-Management Summit (LSFMM), Boaz Harrosh [introduced the ZUFS filesystem](<https://lwn.net/Articles/756625/>). At this year's event, he was back to talk about what it would take to merge ZUFS into the mainline. ZUFS, which Harrosh pronounced as both "zoo-eff-ess" and "zoofs", has been running in production for his employer's (NetApp's) customers for some time now, so he wondered if it was something that could go upstream. 

ZUFS is the "zero-copy user-mode filesystem". When developing it, NetApp set out to do the impossible: create a high-performance filesystem that runs in user space. It needs to run in user space because there are components, libraries, filesystems, and so on, that may be used but are licensed in ways that are not compatible with the GPLv2 used by the kernel. 

[ ![\[Boaz Harrosh\]](https://static.lwn.net/images/2019/lsf-harrosh-sm.jpg) ](<https://lwn.net/Articles/787803/>)

NetApp has shipped ZUFS as a product and customers are happy with it, Harrosh said. It uses persistent memory for performance on the front end; data from there is moved to slower storage over time. That way, customers get the speed of persistent memory, but can store as much data as they want. For optimal performance, the company recommends having 8% of the total storage as persistent memory. A lot of QA testing has been done on the filesystem and customers trust it with their data. 

The kernel part of ZUFS is released under the GPL, but the user-space side is broken into two parts. One is a "systemd server" that is open source, released under the [FreeBSD](<https://en.wikipedia.org/wiki/BSD_licenses#2-clause_license_\(%22Simplified_BSD_License%22_or_%22FreeBSD_License%22\)>) (or 2-clause BSD) license, so that it can be used on operating systems such as Windows or FreeBSD. The other piece is a plugin mechanism that allows vendors to register their code with the user-space server. These plugins will implement the filesystems; the plugins can be released under any license, including a proprietary license. 

So, he asked, do we want the kernel open-source project part of ZUFS in the mainline kernel? He and his colleagues think it is "very very stable". Harrosh has been developing filesystems for many years, he said; in the past, whenever the filesystem crashed, you would have to reboot the virtual machine it was running in. But for ZUFS, that is all different; if the user-space server crashes, you can just restart it and remount the filesystem. 

In ZUFS, the kernel piece is just a broker that provides a fast communication path between the application and the server. A round trip on that path takes 4µs for a simple read or write. For a filesystem made with the [Filesystem in Userspace](<https://en.wikipedia.org/wiki/Filesystem_in_Userspace>) (FUSE) interface, that same round trip takes ten times as long, Harrosh said. 

If the project is simply going to be a NetApp pet project, the company will continue to maintain it, he said. But if it is interesting to others, it could go upstream. Jan Kara suggested applying the communication techniques used by ZUFS to FUSE as a way for others to get access, but Harrosh does not think that is possible. ZUFS is a completely different idea that is unlike FUSE or anything else. 

Ted Ts'o said that most FUSE filesystems he knows about would not benefit from the ZUFS communication scheme because they are not performant enough. He thought that if ZUFS lived in its own directory, and did not make big changes outside of it, that it could perhaps be merged. That would allow others to experiment with it and for FUSE to perhaps incorporate parts of it. Ric Wheeler pointed out that there are some FUSE filesystems, Gluster and CephFS, for example, that do care about performance. 

Harrosh said that the main novelty of ZUFS is that all of the communication is synchronous and is all done on a single CPU. Everything in ZUFS is done on a single CPU; the application grabs a CPU channel and the server registers threads on that CPU, so the server runs on it. It is all completely lockless and there are no copies made of the data, which is directly read from or written to the application buffers—or to/from DMA and RDMA devices. 

Trond Myklebust wondered what made it impossible to incorporate the ZUFS ideas in FUSE and Amir Goldstein asked how a filesystem using libfuse could use libzufs (or its equivalent). Harrosh said that ZUFS is "very incompatible" with libfuse, but is actually compatible with the filesystem code. Harrosh said that making a ZUFS filesystem was much easier than making a FUSE filesystem. He looked into FUSE and hit the performance wall right away. So he did this work and would like everyone to be able to use it. 

ZUFS is, of course, much newer than FUSE; he wonders, for example, if there are additional steps he needs to take to ensure that ZUFS is not leaving behind security holes. There are uses for ZUFS beyond simply filesystems, he said, any kind of server can use it; it could be used for SQL server communication over the lockless CPU channels, for example. It is "so fast and so different" that Harrosh thinks getting it in the kernel will cause an appetite to develop. 

Steven Whitehouse thought that separating the fast communication mechanism from the filesystem pieces might make for an easier path, at least for the first part, into the kernel. But Harrosh said that he is not sure how to separate the VFS plugin aspect from that of the communication channel. ZUFS "intimately sits" under the VFS and acts as a POSIX filesystem, he doesn't know how to split those things apart. 

The general consensus was that use cases will be needed before the communication channel stuff can go in the mainline; Linus Torvalds and others will ask about them. If the code can be used by FUSE that will only help smooth the path, so working with the FUSE developers to see how the two can cooperate would make sense, Harrosh said in summary. In addition, more users beyond just the NetApp filesystem is probably needed. 

He asked for examples of the bigger FUSE users; attendees responded that Gluster and Oracle database filesystem were two. Harrosh said that NetApp has customers using Oracle; using ZUFS allows them to get a 10x performance increase without seeing any difference. He would like to have a workshop or similar at an upcoming conference to show how to write ZUFS servers to encourage others to learn and experiment with the technology. At this point, it may require some marketing to get it upstream, he thought.
