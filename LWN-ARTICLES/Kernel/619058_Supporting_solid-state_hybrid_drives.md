---
title: "Supporting solid-state hybrid drives"
url: https://lwn.net/Articles/619058/
date: "November 5, 2014"
category: "Block layer-Solid-state storage devices; Solid-state storage devices"
author: "By Jonathan Corbet November 5, 2014"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
November 5, 2014

In recent years we have seen the addition of a number of subsystems to the kernel that provide high-speed caching for data on (relatively) slow drives; examples include [bcache](<https://lwn.net/Articles/497024/>) and [dm-cache](<https://lwn.net/Articles/540996/>). But there is nothing preventing drive manufacturers from building this kind of caching into their products directly. The result of such bundling is "solid-state hybrid drives" — rotating drives that have some flash storage built in as well. Properly used, that flash storage can speed accesses to frequently used data. But it turns out that getting to "properly used" is not quite as straightforward as one might think. 

Of course, one can simply leave everything up to the drive itself. Left to its own devices (so to speak), the drive will observe which blocks are frequently accessed and work to keep those blocks in fast storage. But the operating system — or the programs running on that system — will often have a better idea of which data will be most useful in the future. If that information is communicated to the drives, the result should be better use of fast storage, and, thus, better performance. 

Enabling that communication is the goal of [this patch set](<https://lwn.net/Articles/618489/>) posted by Jason Akers. The response to that patch set from the kernel community makes it clear, though, that there is still some work to be done to figure out the best way to get the best possible performance from such drives. 

This patch set uses the per-process I/O priority value as a way of signaling information about cache usage. That priority can be set by way of the [`ionice` command](<http://linux.die.net/man/1/ionice>). Using a few bits of the priority field, the user can specify one of a number of policies (listed here in symbolic form): 

  * `IOPRIO_ADV_EVICT` says that the data involved in I/O operations should be actively removed from the cache, should it be found there. It's a way of saying that the data will, with certainty, not be used again in the near future. 

  * `IOPRIO_ADV_DONTNEED` says that the data should not be cached, but that there is no need to actively evict it from the cache if it's already there. 

  * `IOPRIO_ADV_NORMAL` leaves caching policy up to the drive, as if no advice had been provided at all. 

  * `IOPRIO_ADV_WILLNEED` indicates that the data will be needed again in the near future and, thus, should be stored in the cache. 

This patch set is unlikely to be merged in anything close to its current form for a few reasons. One of those is that, as a few developers pointed out, associating I/O caching policy with a process is a bit strange. Any given process may want different caching policies for different files it works with; indeed, it may want different policies for different parts of the same file. Creating a single, per-process policy makes this kind of use nearly impossible. 

Beyond that, as Dave Chinner [pointed out](<https://lwn.net/Articles/619060/>), the process that generates an I/O operation in user space may not be the process that submits the I/O to the block subsystem. Many filesystems use worker threads to perform actual submission; that breaks the link with the process that originally created the I/O operation. Filesystems, too, may wish to adjust caching policy; giving metadata a higher priority for the cache than data is one obvious possibility. As it happens, there is a way for filesystems to adjust the I/O priority value on individual requests, but it is not the most elegant of APIs. 

For these reasons, some developers have suggested that the caching policy should be set on a per-file basis with a system call like `fadvise()` rather than on a per-process basis. Even better, as Jens Axboe [noted](<https://lwn.net/Articles/619061/>), would be to add a mechanism by which processes could provide hints on a per-operation basis. The approach used in the [non-blocking buffered read proposal](<https://lwn.net/Articles/612483/>) might be applicable for that type of use. 

There is another problem with this patch set, though: the types of "advice" that can be provided is tied tightly to the specifics of how the current generation of hybrid drives operates. It offers low-level control over a single level of cache and not much else. Future drives may operate in different ways that do not correspond well to the above-described operations. Beyond that, hybrid drives are not the only place where this kind of advice can be provided; it can also be useful over NFS 4.2, with persistent memory devices, and with the upcoming T10/T13 "logical block markup descriptors." There is a strong desire to avoid merging a solution that works with one type of current technology, but that will lack relevance with other technologies. 

Martin Petersen has put some time into trying to find an optimal way to provide advice to storage devices in general. His approach is to avoid specific instructions ("evict this data from the cache") in favor of a description why the I/O is being performed. He [described](<https://lwn.net/Articles/619062/>) his results as ""a huge twisted mess of a table with ponies of various sizes"", but it's not all that complicated in the end. 

That table consists of a set of I/O classes, along with the performance implications of each class. There is a "transaction" class with stringent completion-time and latency requirements and a high likelihood that the data will be accessed again in the near future. The "streaming" class also wants fast command completion, but the chances of needing the data again soon are quite low. Other classes include "metadata" (which is like transactions but with a lower likelihood of needing the data again), "paging," "data," and "background" (which has low urgency and no need for caching). 

Given an (unspecified) API that uses these I/O classes, the low-level driver code can map the class of any specific I/O operation onto the proper advice for the hardware. That mapping might be a bit trickier than one might imagine, though, as the hardware gets more complex. There is also the problem of consistency across devices; if drivers interpret the classes differently, the result could be visible performance differences that create unhappy users. 

These issues will need to be worked out, though, if Linux systems are to drive hybrid devices in anything other than the default, device-managed mode. Given a suitable kernel and user-space API, the class-based approach looks like it should be flexible enough to get the most out of near-future hardware. Getting there, though, means a trip back to the drawing board for the authors of the current hybrid-drive patches.
