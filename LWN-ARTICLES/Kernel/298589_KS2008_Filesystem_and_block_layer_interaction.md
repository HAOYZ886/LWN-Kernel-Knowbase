---
title: "KS2008: Filesystem and block layer interaction"
url: https://lwn.net/Articles/298589/
date: "September 16, 2008"
category: Filesystems
author: "By Jonathan Corbet September 16, 2008 LWN's 2008 Kernel Summit coverage"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
September 16, 2008

* * *

LWN's [2008 Kernel Summit coverage](<https://lwn.net/Articles/KernelSummit2008/>)

Much is happening with Linux filesystems currently; this is a situation which is likely to persist for some time. As filesystems develop, it is becoming clear that there need to be some changes in the interactions between the filesystem and block I/O layers. This kernel summit session discussed some of the places where changes are needed, but did not get much into their implementation. 

Chris Mason is the lead developer of the up-and-coming btrfs filesystem. One of the items on Chris's shopping list is a way for filesystems to obtain a better understanding of the topology and nature of the storage system underneath them. He would like, for example, to be able to determine whether a filesystem is sitting on a solid-state device or on a traditional rotating disk. Certain decisions will be made very differently depending on the nature of the underlying device; filesystems stored on solid-state drives, for example, can be laid out without being concerned about seek times. 

The topology of the device also matters. Especially when multipath storage systems are in use, the filesystem would like to be able to understand what the various paths are, and to be able to partition it into truly independent failure domains. With this information, filesystems can find the optimal ways to perform I/O to the underlying devices. 

Information needs to flow the other way as well. Upcoming filesystems will perform extensive checksumming on data, so they will be able to inform the storage layer when a block has gone bad. For mirrored devices, that will enable the storage driver to recover the block from an uncorrupted mirror - if the filesystem is able to tell it which mirror went bad. 

Chris asked for information on storage latency - how long operations can be expected to last - and the optimal I/O sizes and alignments. The motivation behind this request is to optimize I/O to solid-state devices. Here Linus jumped in and suggested that the filesystem developers should "take a deep breath and wait a year." Solid-state devices will change a lot over that time, and many of the problems which exist now will be gone by then. So filesystems designed for today's solid-state drives will contain a lot of useless code by the time those drives are truly widespread. It is better, Linus says, to just treat them as a fast, random-access disk and not worry about the details. 

Another request was for filesystems to be able to allocate their own `bio` structures, rather than using the block layer's allocation functions. That would allow the filesystems to store their own private data with the `bio` without the need to tack on a chain of separate structures via the `bi_private` pointer. There's also a general need to rework the address space operations to facilitate better layout and more rational locking. 

The kswapd process is a bit of a problem for contemporary filesystems. Kswapd is charged with freeing up pages for the memory allocator; it needs to be able to get its job done at times when system memory is very tight. Currently kswapd will attempt to write out dirty pages so that they can be freed. The problem is that this writeout can require more memory to carry out; as filesystems become more complex, the amount of extra memory needed seems to be growing. That can lead to deadlocks if that extra memory is not available. So the filesystem developers would like kswapd to concern itself exclusively with clean pages, which can be freed without performing I/O. 

One answer that came back was that the `writepage()` VFS callback can be treated as advisory. That is what btrfs does now; if a `writepage()` call comes in the context of a process with the `PF_MEMALLOC` bit set (meaning that the system is trying to free memory), the call will simply fail. That is all legal, but it can hurt performance. 

In the end, kswapd does writeout because, historically, it was possible for a Linux system to end up with all of its pages being dirty. In that kind of situation, writeout is the only way to make memory available again. But current kernels are able to keep close tabs on how much of memory is dirty at any given time, and they can avoid getting into that kind of situation. So writeout in kswapd is no longer necessary; it can, instead, be handled in contexts where memory is not in critically short supply. This change seems likely to be made in the near future. 

The final topic, discussed briefly, was I/O barriers. The filesystem developers would really like it if the more complex storage layers - such as the software RAID and device mapper code - would implement write barriers. That is a hard thing to do with the current concept of barriers, though; the performance costs will be high. James Bottomley noticed that a better job could be done with a more complex barrier API. But it is not clear whether the benefits that would come would be worth the extra cost.
