---
title: POSIX_FADV_VOLATILE
url: https://lwn.net/Articles/468896/
date: "November 22, 2011"
category: Memory management; Volatile ranges; posix fadvise
author: "By Jonathan Corbet November 22, 2011"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
November 22, 2011

Caching plays an important role at almost all levels of a contemporary operating system. Without the ability to cache frequently-used objects in faster memory, performance suffers; the same idea holds whether one is talking about individual cache lines in the processor's memory cache or image data cached by a web browser. But caching requires resources; those needs must be balanced with other demands on the same resources. In other words, sometimes cached data must be dropped; often, overall performance can be improved if the program doing the caching has a say in what gets removed from the cache. A recent patch from John Stultz attempts to make it easier for applications to offer up caches for reclamation when memory gets tight. 

John's [patch](<https://lwn.net/Articles/468837/>) takes a lot of inspiration from the [ashmem](<http://elinux.org/Android_Kernel_Features#ashmem>) device implemented for Android by Robert Love. But ashmem functions like a device and performs its own memory management, which makes it hard to merge upstream. John's patch, instead, tries to integrate things more deeply into the kernel's own memory management subsystem. So it takes the form of a new set of options to the `posix_fadvise()` system call. In particular, an application can mark a range of pages in an open file as "volatile" with the `POSIX_FADV_VOLATILE` operation. Pages that are so marked can be discarded by the kernel if memory gets tight. Crucially, even dirty pages can be discarded - without writeback - if they have been marked volatile. This operation differs from `POSIX_FADV_DONTNEED` in that the given pages will not (normally) be discarded right away - the application _might_ want the contents of volatile pages in the future, but it will be able to recover if they disappear. 

If a particular range of pages becomes useful later on, the application should use the `POSIX_FADV_NONVOLATILE` operation to remove the "volatile" marking. The return value from this operation is important: a non-zero return from `posix_fadvise()` indicates that the kernel has removed one or more pages from the indicated range while it was marked volatile. That is the only indication the application will get that the kernel has accepted its offer and cleaned out some volatile pages. If those pages have not been removed, `posix_fadvise()` will return zero and the cached data will be available to the application. 

There is also a `POSIX_FADV_ISVOLATILE` operation to query whether a given range has been marked volatile or not. 

Rik van Riel [raised](<https://lwn.net/Articles/468902/>) a couple of questions about this functionality. He expressed concern that the kernel might remove a single page of a multi-page cached object, thus wrecking the caching while failing to reclaim all of the memory used to cache that object. Ashmem apparently does its own memory management partially to avoid this very situation; when an object's memory is reclaimed, all of it will be taken back. John would apparently rather avoid adding another least-recently-used list to the kernel, but he did respond that it might be possible to add logic to reclaim an entire volatile range once a single page is taken from that range. 

Rik also worried about the overhead of this mechanism and proposed an alternative that he has apparently been thinking about for a while. In this scheme, applications would be able to open (and pass to `poll()`) a special file descriptor that would receive a message whenever the kernel finds itself short of memory. Applications would be expected to respond by freeing whatever memory they can do without. The mechanism has a certain kind of simplicity, but could also prove difficult in real-world use. When an application gets a "free up some memory" message, the first thing it will probably need to do is to fault in its code for handling that message - an action which will require the allocation of more memory. Marking the memory ahead of time and freeing it directly from the kernel may turn out to be a more reliable approach. 

After the recent [frontswap discussions](<https://lwn.net/Articles/465317/>), it is perhaps unsurprising that nobody has dared to observe that volatile memory ranges bear a more than passing resemblance to transcendent memory. In particular, it looks a lot like "cleancache," which was merged in the 3.0 development cycle. There are differences: putting a page into cleancache removes it from normal memory while volatile memory can remain in place, and cleancache lacks a user-space interface. But the core idea is the same: asking the system to hold some memory, but allowing that memory to be dropped if the need arises. It could be that the two mechanisms could be made to work together. 

But, as noted above, nobody has mentioned this idea, and your editor would certainly not be so daring. 

One other question that has not been discussed is whether this code could eventually replace ashmem, reducing the differences between the mainline and the Android kernel. Any such replacement would not happen anytime soon; ashmem has its own ABI that will need to be supported by Android kernels for a long time. Over years, a transition to `posix_fadvise()` could possibly be made if the Android developers were willing to do so. But first the `posix_fadvise()` patch will need to get into the mainline. It is a very new patch, so it is hard to say if or when that might happen. Its relatively non-intrusive nature and the clear need for this capability would tend to argue in its favor, though.
