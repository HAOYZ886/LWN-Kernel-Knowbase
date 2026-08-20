---
title: Adaptive file readahead
url: https://lwn.net/Articles/155510/
date: "October 12, 2005"
category: "Memory management-Readahead; Readahead"
author: 
---

Readahead is a technique employed by the kernel in an attempt to improve file reading performance. If the kernel has reason to believe that a particular file is being read sequentially, it will attempt to read blocks from the file into memory before the application requests them. When readahead works, it speeds up the system's throughput, since the reading application does not have to wait for its requests. When readahead fails, instead, it generates useless I/O and occupies memory pages which are needed for some other purpose. 

The current kernel readahead implementation uses a window 128KB in length. When readahead seems appropriate, the kernel will speculatively bring in the next 128KB of file data. If the application continues to read sequentially through that data, the next 128KB chunk will be brought in when the application is part-way through the first one. This implementation works, but Wu Fengguang thinks that it can be made better. 

In particular, Wu thinks that the fixed readahead window size should, instead, adapt to both the application's behavior and the global state of the system. His [adaptive readahead patch](<https://lwn.net/Articles/155097/>) is an implementation of this thought. It is a work of daunting complexity, but the core ideas are reasonably straightforward. 

The adaptive readahead patch tries to balance two constraints: readahead should be performed aggressively, but not to the point that the system starts thrashing or readahead pages get recycled before the application uses them. Every time a readahead decision is to be made for a specific file, the adaptive code looks at how much memory is available for readahead and how quickly the application has been working through the file. If memory is tight, or if the disk holding the file is congested, readahead will not be performed at all. 

The code also looks at the pressure on the inactive page lists and tries to figure out whether any readahead pages are in danger of falling off that list and being reclaimed. In that situation, the readahead pages will be moved back up the list, keeping them in memory for a bit longer. This "rescue" operation helps to keep previous readahead work from being wasted; since it is only performed when the application consumes data from the file, it will not happen if the reading process has stalled entirely. But, when the application _is_ working through the data, it will get another chance to benefit from readahead which has already been performed. No more readahead will be started in that situation, however. 

If, instead, the application is making use of its readahead pages and the memory is available, the readahead window can grow up to 1MB. For streaming media or data processing applications which work their way sequentially through large files, this enlarged window can lead to significant performance gains. 

In fact, Wu claims results which are "pretty optimistic." They include a 20-100% improvement for applications doing parallel reads, and the ability to run 800 1KB/sec simultaneous streams on a 64MB system without thrashing. The page cache hit rate is claimed to be 91%, which is quite good. 

The adaptive readahead patch might, thus, be a worthwhile addition to the Linux memory management subsystem. There has been little discussion (none, actually) of the patch on the list, however. Complicated patches working in an obscure corner of memory management do not receive the same level of review as, say, new filesystems, it would seem. In any case, a patch of this nature will require a good deal of testing before it can be considered for any sort of merge. So, while adaptive readahead may indeed make its way into the mainline, it's not something to expect to see in the very near future.
