---
title: "Buffered I/O without page-cache thrashing"
url: https://lwn.net/Articles/806980/
date: "December 12, 2019"
category: "Memory management-Page cache"
author: "By Jonathan Corbet December 12, 2019"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
December 12, 2019

Linux offers two modes for file I/O: buffered and direct. Buffered I/O passes through the kernel's page cache; it is relatively easy to use and can yield significant performance benefits for data that is accessed multiple times. Direct I/O, instead, goes straight between a user-space buffer and the storage device. It can be much faster for situations where caching by the operating system isn't necessary, but it is complex to use and contains traps for the unwary. Now, it seems, Jens Axboe has come up with [a way to get many of the benefits of direct I/O](<https://lwn.net/ml/linux-fsdevel/20191210162454.8608-1-axboe@kernel.dk/>) with a lot less bother. 

Direct I/O can give better performance than buffered I/O in a couple of ways. One of those is simply avoiding the cost of copying the data between user space and the page cache; that cost can be significant, but in many cases it is not the biggest problem. The real issue may be the effect of buffered I/O on the page cache. 

A process that performs large amounts of buffered I/O spread out over one or more large (relative to available memory) files will quickly fill the page cache (and thus memory) with cached file data. If the process in question does not access those pages after performing I/O, there is no benefit to keeping the data in memory, but it's there anyway. To be able to allocate memory for other uses, the kernel will have to reclaim some pages from somewhere. That can be expensive for the system as a whole, even if "somewhere" is the data associated with this I/O activity. 

The memory-management subsystem tries to do the right thing in this situation. Pages added to the cache via buffered I/O go onto the inactive list; unless they are accessed a second time in the near future, they will be the first pages to be kicked back out. But there is still a fair amount of overhead associated with implementing this behavior; Axboe ran a simple test and described the results this way: 

The test case is pretty basic, random reads over a dataset that's 10x the size of RAM. Performance starts out fine, and then the page cache fills up and we hit a throughput cliff. CPU usage of the IO threads go up, and we have kswapd spending 100% of a core trying to keep up. 

This kind of problem can be avoided by switching to direct I/O, but that brings challenges and problems of its own. Axboe has concluded that there may be a third way that can provide the best of both worlds. 

That third way is a new flag, `RWF_UNCACHED`, which is provided to the [`preadv2()` and `pwritev2()`](<http://man7.org/linux/man-pages/man2/readv.2.html>) system calls. If present, this flag changes the requested I/O operation in two ways, depending on whether the affected file pages are currently in the page cache or not. When the data is present in the page cache, the operation proceeds as if the `RWF_UNCACHED` flag were not present; data is copied to or from the pages in the cache. If the pages are absent, instead, they will be added to the page cache, but only for the duration of the operation; those pages will be removed from the page cache once the operation completes. 

The result, in other words, is buffered I/O that does not change the state of the page cache; whatever was present there before will still be there afterward, but nothing new will be added. I/O performed in this way will gain most of the benefits of buffered I/O, including ease of use and access to any data that is already cached, but without filling memory with unneeded cached data. The result, Axboe says, is readily observable: 

With this, I can do 100% smooth buffered reads or writes without pushing the kernel to the state where kswapd is sweating bullets. In fact it doesn't even register. 

This new flag thus seems like a significant improvement for a variety of workloads. In particular, workloads where it is known that the data will only be used once, or where the application performs its own caching in user space, may well benefit from running with the `RWF_UNCACHED` flag. 

The implementation of this new behavior is not complicated; the entire patch set (which also adds support to [io_uring](<https://lwn.net/Articles/776703/>)) involves just over 200 lines of code. Of course, as Dave Chinner [pointed out](<https://lwn.net/ml/linux-fsdevel/20191211002349.GC19213@dread.disaster.area/>), there is something missing: all of the testing infrastructure needed to ensure that `RWF_UNCACHED` behaves as expected and does not corrupt data. Chinner also [noted](<https://lwn.net/ml/linux-fsdevel/20191211011415.GE19213@dread.disaster.area/>) some performance issues in the write implementation, suggesting that an entire I/O operation should be flushed out at a time rather than the page-at-a-time approach taken in the original patch set. Axboe has already [reworked the code](<https://lwn.net/ml/linux-fsdevel/20191211152943.2933-1-axboe@kernel.dk/>) to address that problem; the boring work of writing tests and precisely documenting semantics will follow at some future point. 

If `RWF_UNCACHED` proves to work as well in real-world workloads, it may eventually be seen as one of those things that somebody should have thought of many years ago. Things often turn out this way. Solving the problem isn't hard; the hard part is figuring out which problem needs to be solved in the first place. That, and writing tests and documentation, of course.
