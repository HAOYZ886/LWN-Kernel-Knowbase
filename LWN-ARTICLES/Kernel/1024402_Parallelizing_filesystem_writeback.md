---
title: Parallelizing filesystem writeback
url: https://lwn.net/Articles/1024402/
date: "June 12, 2025"
category: Buffered IO
author: "By Jake Edge June 12, 2025 LSFMM+BPF"
---

By **Jake Edge**  
June 12, 2025

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2025/>)

Writeback for filesystems is the process of flushing the "dirty" (written) data in the page cache to storage. At the 2025 Linux Storage, Filesystem, Memory Management, and BPF Summit (LSFMM+BPF), Anuj Gupta led a combined storage and filesystem session on some work that has been done to parallelize the writeback process. Some of the performance problems that have been seen with the existing single-threaded writeback came up in a [session at last year's summit](<https://lwn.net/Articles/976856/>), where the idea of doing writeback in parallel was discussed. 

Gupta began by noting that Kundan Kumar, who posted the [topic proposal](<https://lwn.net/ml/all/20250129102627.161448-1-kundan.kumar@samsung.com/>), was supposed to be leading the session, but was unable to attend. Kumar and Gupta have both been working on a prototype for parallelizing writeback; the session was meant to gather feedback on it. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

Currently, writeback for buffered I/O is single-threaded, though applications are issuing multithreaded writes, which can lead to contention. The backing storage device is represented in the kernel by a [BDI](<https://elixir.bootlin.com/linux/v6.15.1/source/include/linux/backing-dev-defs.h#L163>) (`struct backing_dev_info`), and each BDI has a single writeback thread that processes the [`struct bdi_writeback`](<https://elixir.bootlin.com/linux/v6.15.1/source/include/linux/backing-dev-defs.h#L83>) embedded in it. Each `bdi_writeback` has a single list of inodes that need to be written back and a single [`delayed_work`](<https://elixir.bootlin.com/linux/v6.15.1/source/include/linux/workqueue.h#L113>) instance, which makes the process single-threaded. 

[ ![\[Anuj Gupta\]](https://static.lwn.net/images/2025/lsfmb-gupta-sm.png) ](<https://lwn.net/Articles/1025054/>)

Based on ideas from Dave Chinner and Jan Kara in the topic-proposal thread, he and Kumar added a `struct bdi_writeback_ctx` to contain context information for a `bdi_writeback`, Gupta said. An array of the writeback contexts was added to the BDI, which allows for multiple threads doing writeback for a BDI. That can be seen in a [patch series](<https://lwn.net/ml/all/20250529111504.89912-1-kundan.kumar@samsung.com/>) posted by Kumar at the end of May. They see two levels of parallelism that could be added, but focused on the high-level parallelism in the prototype, leaving low-level parallelism for later work. 

The high-level parallelism splits up the work based on the filesystem geometry, Gupta said. For example, XFS has [allocation groups](<https://en.wikipedia.org/wiki/XFS#Allocation_groups>) that can be used to partition the inodes needing writeback into separate contexts of the BDI. The low-level parallelism could be added by having multiple `delayed_work` entries in the `bdi_writeback`; different types of filesystem operations, such as block allocation versus pure overwrites for XFS, could be handled by separate workqueues at that level. 

Christoph Hellwig was not sure that allocation groups were the right choice for partitioning (sharding) the inodes for writeback. Gupta said that testing to see what kind of workloads benefit when using the allocation groups will be important. Parallel writeback will be an option; users can disable it if it is not beneficial for their workloads. 

Amir Goldstein asked how severe the current bottleneck with single-threaded writeback is. Gupta said that he did not have any numbers on that. Chris Mason said that in his testing it largely depends on whether large folios are available; the easiest way to see performance problems with single-threaded writeback is to turn off large folios for XFS. In some simple testing, he could get around 800MB per second on XFS with large folios disabled before the kernel writeback thread was saturated. With large folios enabled, that number goes to around 2.4GB per second. 

Hellwig was curious where the cycles were being spent in the no-large-folios case. Mason said that he did not remember the details, though pages were being moved around a lot to different lists. Matthew Wilcox suggested that the [XArray](<https://docs.kernel.org/core-api/xarray.html>) API might be part of the problem, since there is no API to clear the dirty bits on a range of folios. If the kernel is clearing those bits one-by-one, that might be a bottleneck; it could be fixed, but he has not found the time to do so. 

Jeff Layton said that the network filesystems would need some way to limit how much parallelization was happening for writeback. There may be underlying network conditions that will necessitate limiting parallel writeback. Gupta acknowledged that need and Hellwig said that local filesystems may also need a way to do that at times. One thing that Hellwig would like to see change is the Btrfs-internal mechanism for having multiple threads doing writeback with compression and checksumming; having a common solution with multiple writeback threads ""would remove a lot of crap"". He is concerned that other filesystems will pick up that code, so having a common solution would head that off; Mason said he was not opposed to that idea, though there may be large files that need their writeback handled specially. 

Over the remote link, Jan Kara said that what he was hearing was that most of the interest was in the low-level parallelism, which would parallelize the writeback of individual inodes. He cautioned that there are various problems with doing that, including potential problems with data-integrity measurements, since writes are done in a different order. 

Hellwig said that the ""global-ish linked lists"" that are used to track the inodes for writeback are a data structure that is known to cause scalability problems. Adding more threads to work on those lists will make things worse; he wondered if using an XArray instead would be better. Using allocation groups to shard the inode lists (or XArrays) makes sense for XFS—other filesystems may have their own obvious choice for sharding—but the management of the threads can be separate from the management of the lists. Other criteria could be used to determine the thread to handle a specific inode. 

Mason agreed and wanted to suggest something similar. For the first version of multithreaded writeback, he said that keeping a single inode list would make sense. Rather than work out how to shard the inodes to different lists, simply ""focus on being able to farm out multiple inodes at a time under writeback""; that will help determine where the lock contention is. Gupta was concerned about the lock contention for the inode list, but Kara did not think it would be that significant compared to all of the other work needed for writeback. 

Once the basics of the feature are working, Goldstein said, it may make sense to provide a means for filesystems to help choose inodes for the different threads. Filesystems may have information that can assist the writeback machinery to achieve better parallelism. 

Mason got in the last word of the session by relaying another bottleneck that he had found in his testing but had forgotten earlier: freeing clean pages. Once dirty pages have been written, those clean pages are freed by the kswapd kernel thread, which results in another bottleneck. He suggested that there may need to be some parallelism added there as well.
