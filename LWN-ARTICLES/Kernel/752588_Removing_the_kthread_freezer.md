---
title: "Removing the kthread freezer?"
url: https://lwn.net/Articles/752588/
date: "April 25, 2018"
category: Kernel threads
author: "By Jake Edge April 25, 2018 LSFMM"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jake Edge**  
April 25, 2018

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2018/>)

Using the kernel thread (kthread) freezer has been a longtime problem for a variety of reasons. It is meant as a way to suspend kthreads on the way toward system suspend, but in practice has proved problematic to the point that it came up at both the [2015](<https://lwn.net/Articles/662703/>) and [2016](<https://lwn.net/Articles/705269/>) Kernel Summits (as well as on the mailing lists over the years); the intent is to try to remove the kthread freezer entirely. To that end, Luis Rodriguez led a discussion in the filesystem track of the 2018 Linux Storage, Filesystem, and Memory-Management Summit on the problems and possible solutions. 

[ ![\[Luis Rodriguez\]](https://static.lwn.net/images/2018/lsf-rodriguez-sm.jpg) ](<https://lwn.net/Articles/752692/>)

Rodriguez has [picked up](<https://lwn.net/Articles/735382/>) the work that Jiri Kosina was doing to eliminate the kthread freezer, but is moving more cautiously than Kosina originally planned. One problem is that the kernel does not want to freeze kthreads in unexpected places, so there is a mechanism that allows the threads to block the freezing process. Part of the thinking there is that there should not be DMA in flight while the suspend is going on, Kent Overstreet said. He asked, wouldn't it be better if the drivers put themselves in a sane state for suspend? 

Dave Chinner said that even if the devices are ready to suspend, the filesystems can still be making in-memory changes. A recurring problem is that suspend would sync a filesystem to make it stable, but the filesystem would still have threads and work on workqueues that were operating on the in-memory data. That led to an inconsistent state between what was on disk and what was in the memory image used by the suspend. 

In general, Rodriguez said, the kernel should not be freezing kthreads. The threads want full control of where they can be frozen; it is hard to get it all right if it is imposed on them. But trying to address this problem in a generic form is "really hard"; phasing out the kthread freezer will be difficult, so he suggested a divide-and-conquer approach. 

For filesystems that implement the `freeze_fs()` method, it should be straightforward, but there is still a problem in getting the order right. The current mechanism freezes the most recently mounted filesystems first and thaws them in the order in which they were mounted. That is simple to do using [`iterate_supers()`](<https://elixir.bootlin.com/linux/v4.16.4/source/fs/super.c#L583>), but does it work in all cases? 

Al Viro said that it does not. There is a "nasty `ioctl()`", which he is sorry for implementing, that can break the ordering. It is quite possible that a filesystem that was mounted later shows up earlier in the list. The ordering described is also not sufficient for FUSE filesystems, Jan Kara said, though Chinner suggested those simply be skipped in the walk. 

But there are filesystems that talk to several devices, such as those hosted on a RAID device or with their journal on a separate device, Viro said. These topologies can also change at run time, so he does not recommend relying on any kind of ordering. 

In fact, a directed acyclic graph (DAG) could describe these relationships, Kara said. It would have nodes for filesystems and devices, with edges that describe the dependencies between them. It would be nice to build that DAG in the kernel, but it is not done today. Viro agreed that it is probably needed at some point. 

Rodriguez wondered whether the DAG generation was required before making any progress on eliminating the kthread freezer. As long as the existence of the problem is kept in mind, Viro said, work can proceed. If these problem configurations can be detected, suspend could be prohibited for those systems, Rodriguez said. But that will be difficult to detect without the graph, Kara said. 

There are a number of problem areas that came up in the discussion: freezing races with automounting, the control group (cgroup) freezer is "completely broken", freezing FUSE filesystems is problematic, and so on. It was noted that applications would like to know if the filesystem they are using is about to freeze so they can quiesce their own data to keep it consistent. Rodriguez was surprised to find out that there is no generic framework for the kernel to notify user space about an upcoming suspend: "That's insane!" 

No real conclusions came out of the discussion. Rodriguez plans to post his notes to the mailing list for feedback. There was also talk about discussing it more later in the summit, though that has not been scheduled as of this writing.
