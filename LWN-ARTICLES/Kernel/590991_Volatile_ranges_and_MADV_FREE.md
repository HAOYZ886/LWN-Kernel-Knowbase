---
title: Volatile ranges and MADV_FREE
url: https://lwn.net/Articles/590991/
date: "March 19, 2014"
category: Volatile ranges
author: "By Jonathan Corbet March 19, 2014"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
March 19, 2014

Within the kernel, the "shrinker" interface allows the memory-management subsystem to inform other subsystems that memory is tight and that some space should be freed if possible. Various attempts have been made to add a similar mechanism that would allow the kernel to ask user-space processes to do some tidying up, but all have run up against the familiar problems of complexity and the general difficulty of getting memory-management changes merged. That doesn't stop developers from trying, though; recently two new patches of this type have been posted. 

Both of these patch sets implement variations on a feature that has often gone by the name [volatile ranges](<https://lwn.net/Kernel/Index/#Volatile_ranges>). A volatile range is a region of memory in a process's address space that is used to store data that can be regenerated if need be. If the kernel finds itself short of memory, it can take pages from a volatile range, secure in the knowledge that the process using that range of memory can recover from the loss, albeit with a possible performance hit. But, as long as memory remains plentiful, volatile ranges will not be reclaimed by the kernel and the data cached there can be freely used by applications. 

Much of the volatile range work is motivated by the desire to create a replacement for Android's [ashmem](<http://elinux.org/Android_Kernel_Features#ashmem>) mechanism that is better integrated with the core memory-management subsystem. But there are other potential users of this functionality as well. 

#### Volatile ranges

There have been many versions of the volatile ranges patch set over the last few years. At times, volatile ranges were implemented with the `posix_fadvise()` system call; at others, it was added to `fallocate()` instead. Other versions have made it a feature of `madvise()`. But [version 11 of the volatile ranges patch set](<https://lwn.net/Articles/590771/>) from John Stultz takes none of those approaches. Instead, it adds a new system call: 

```
int vrange(void *start, size_t length, int mode, int *purged);
```

In this incarnation, a `vrange()` call operates on the `length` bytes of memory beginning at `start`. If `mode` is `VRANGE_VOLATILE`, that range of memory will be marked as volatile. If, instead, `mode` is `VRANGE_NONVOLATILE`, the volatile marking will be removed. In this case, though, some or all of the pages previously marked as being volatile might have been reclaimed; in that case, `*purged` will be set to a non-zero value to indicate that the previous contents of that memory range are no longer available. If `*purged` is set to zero, the application knows that the memory contents have not been lost. 

A process may continue to access memory contained within a volatile range. Should it attempt to access a page that has been reclaimed, though, it will get a `SIGBUS` signal to indicate that the page is no longer there. Thus, programs that are prepared to handle that signal can use volatile ranges without the need for a second `vrange()` call before actually accessing the memory. 

This version of the patch differs from its predecessors in another significant way: it only works with anonymous pages while the previous versions worked only with the tmpfs filesystem. Working with anonymous pages satisfies the need to simplify the patch set as much as possible in the hope of getting it reviewed and eventually merged, but it has a significant cost: the inability to work with tmpfs means that volatile ranges are not a viable replacement for ashmem. The intent is to support the file-backed case (which adds more complexity) after there is consensus on the basic patch. 

Internally, `vrange()` works at the virtual memory area (VMA) level. All pages within a VMA are either volatile or not; if need be, VMAs will be split or coalesced in response to `vrange()` calls. This should make a `vrange()` call reasonably fast since there is no need to iterate over every page in the range. 

#### MADV_FREE

A different approach to a similar problem can be seen in Minchan Kim's [`MADV_FREE`](<https://lwn.net/Articles/590693/>) patch set. This patch adds a new command to the existing `madvise()` system call: 

```
int madvise(void *addr, size_t length, int advice);
```

Like `vrange()`, `madvise()` operates on a range of memory specified by the caller; what it does is determined by the `advice` argument. Callers can specify `MADV_SEQUENTIAL` to tell the kernel that the pages in that range will be accessed sequentially, or `MADV_RANDOM` to indicate the opposite. The `MADV_DONTNEED` call causes the kernel to reclaim the indicated pages immediately and drop their contents. 

The new `MADV_FREE` operation is similar to `MADV_DONTNEED`, but there is an important difference. Rather than reclaiming the pages immediately, this operation marks them for "lazy freeing" at some future point. Should the kernel run low on memory, these pages will be among the first reclaimed for other uses; should the application try to use such a page after it has been reclaimed, the kernel will give it a new, zero-filled page. But if memory is not tight, pages marked with `MADV_FREE` will remain in place; a future access to those pages will clear the "lazy free" bit and use the memory that was there before the `MADV_FREE` call. 

There is no way for the calling application to know if the contents of those pages have been discarded or not without examining the data contained therein. So a program could conceivably implement something similar to volatile ranges by putting a recognizable structure into each page before the `MADV_FREE` operation, then testing for that structure's presence before accessing any other data in the pages. But that does not seem to be the intended use case for this feature. 

Instead, `MADV_FREE` appears to be aimed at user-space memory allocator implementations. When an application frees a set of pages, the allocator will use an `MADV_FREE` call to tell the kernel that the contents of those pages no longer matter. Should the application quickly allocate more memory in the same address range, it will use the same pages, thus avoiding much of the overhead of freeing the old pages and allocating and zeroing the new ones. In short, `MADV_FREE` is meant as a way to say "I don't care about the data in this address range, but I may reuse the address range itself in the near future." 

It's worth noting that `MADV_FREE` is already supported by BSD kernels, so, unlike `vrange()`, it would not be a Linux-only feature. Indeed, it would likely improve the portability of programs that use this feature on BSD systems now. 

Neither patch has received much in the way of reviews as of this writing. The real review, in any case, is likely to happen at this year's [Linux Storage, Filesystem, and Memory Management Summit](<http://events.linuxfoundation.org/events/linux-storage-filesystem-and-mm-summit>), which begins on March 24\. LWN will be there, and we promise to make at least a token effort to not be too distracted by the charms of California wine country; stay tuned for reports from that discussion.
