---
title: Improving readahead
url: https://lwn.net/Articles/372384/
date: "February 3, 2010"
category: "Memory management-Readahead; Readahead"
author: "By Jonathan Corbet February 3, 2010"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
February 3, 2010

Readahead is the process of speculatively reading file data into the page cache in the hope that it will be useful to an application in the near future. When readahead works well, it can significantly improve the performance of I/O bound applications by avoiding the need for those applications to wait for data and by increasing I/O transfer size. On the other hand, readahead risks making performance worse as well: if it guesses wrong, scarce memory and I/O bandwidth will be wasted on data which will never be used. So, as is the case with memory management in general, readahead algorithms are both performance-critical and heavily based on heuristics. 

As is also generally the case with such code, few people dare to wander into the readahead logic; it tends to be subtle and quick to anger. One of those who dare is Wu Fengguang, who has worked on readahead a few times over the years. His latest contribution is [this set of patches](<http://lwn.net/Articles/372281/>) which tries to improve readahead performance in the general case while also making it more responsive to low-memory situations. 

The headline feature of this patch set is an increase in the maximum readahead size from 128KB to 512KB. Given the size of today's files and storage devices, 512KB may well seem a bit small. But there are costs to readahead, including the amount of memory required to store the data and the amount of I/O bandwidth required to read it. If a larger readahead buffer causes other useful data to be paged out, it could cause a net loss in system performance even if all of the readahead data proves to be useful. Larger readahead operations will occupy the storage device for longer, causing I/O latencies to increase. And one should remember that there can be a readahead buffer associated with every open file descriptor - of which there can be thousands - in the system. Even a small increase in the amount of readahead can have a large impact on the behavior of the system. 

The 512K number was reached by way of an extensive series of benchmark runs using both rotating and solid-state storage devices. With rotating disks, bumping the maximum readahead size to 512KB nearly tripled I/O throughput with a modest increase in I/O latency; any further increases, while increasing throughput again, caused latency increases that were deemed to be unacceptable. On solid-state devices the throughput increase was less (on a percentage basis) but still significant. 

These numbers hold for a device with reasonable performance, though. A typical USB thumb drive, not being a device with reasonable performance, can run into real trouble with an increased readahead size. To address this problem, the patch set puts a cap on the readahead window size for small devices. For a 2MB device (assuming such a thing can be found), readahead is limited to 4KB; for a 2GB drive, the limit is 128KB. Only at 32GB does the full 512KB readahead window take effect. 

This heuristic is not perfect. Jens Axboe [protested](<https://lwn.net/Articles/372388/>) that some solid-state devices are relatively small in capacity, but they can be quite fast. Such devices may not perform as well as they could with a larger readahead size. 

Another part of this patch set is the "context readahead" code which tries to prevent the system from performing more readahead than its memory can handle. For a typical file stream with no memory contention, the contents of the page cache can be visualized (within your editor's poor drawing skills) like this: 

> ![\[Readahead diagram\]](https://static.lwn.net/images/ns/kernel/context_readahead.png)

Here, we are looking at a representation of a stream of pages containing the file's data; the green pages are those which are in the page cache at the moment. Several recently-consumed pages behind the offset have not yet been evicted, and the full readahead window is waiting for the application to get around to consuming it. 

If memory is tight, though, we could find a situation more like this: 

> ![\[Readahead diagram\]](https://static.lwn.net/images/ns/kernel/context_readahead2.png)

Because the system is scrambling for memory, it has been much more aggressive about evicting this file's pages from the page cache. There is much less history there, but, more importantly, a number of pages which were brought in via readahead have been pushed back out before the application was able to actually make use of them. This sort of thrashing behavior is harmful to system performance; the readahead occupied memory when it was needed elsewhere, and that data will have to be read a second time in the near future. Clearly, when this sort of behavior is seen, the system should be doing less readahead. 

Thrashing behavior is easily detected; if pages which have already been read in via readahead are missing when the application tries to actually read them, things are going amiss. When that happens, the code will get an estimate of the amount of memory it can safely use by counting the number of history pages (those which have already been consumed by the application) which remain in the page cache. If some history remains, the number of history pages is taken as a guess for what the size of the readahead window should be. 

If, instead, there's no history at all, the readahead size is halved. In this case, the readahead code will also carefully shift any readahead pages which are still in memory to the head of the LRU list, making it less likely that they will be evicted immediately prior to their use. The file descriptor will be marked as "thrashed," causing the kernel to continue to use the history size as a guide for the readahead window size in the future. That, in turn, will cause the window to expand and contract as memory conditions warrant. 

Readahead changes can be hard to get into the mainline. The heuristics can be tricky, and, as Linus has [noted](<https://lwn.net/Articles/372402/>), it can be easy to optimize the system for a subset of workloads: 

The problem is, it's often easier to test/debug the "good" cases, ie the cases where we _want_ read-ahead to trigger. So that probably means that we have a tendency to read-ahead too aggressively, because those cases are the ones where people can most easily look at it and say "yeah, this improves throughput of a 'dd bs=8192'". 

The stated goal of this patch set is to make readahead more aggressive by increasing the maximum size of the readahead window. But, in truth, much of the work goes in the other direction, constraining the readahead mechanism in situations where too much readahead can do harm. Whether these new heuristics reliably improve performance will not be known until a considerable amount of benchmarking has been done.
