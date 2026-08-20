---
title: Adding a huge zero page
url: https://lwn.net/Articles/517465/
date: "September 26, 2012"
category: "Huge pages; Memory management-Huge pages; Zero page"
author: "By Jonathan Corbet September 26, 2012"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
September 26, 2012

The [transparent huge pages](<https://lwn.net/Articles/423584/>) feature allows applications to take advantage of the larger page sizes supported by most contemporary processors without the need for explicit configuration by administrators, developers, or users. It is mostly a performance-enhancing feature: huge pages reduce the pressure on the system's translation lookaside buffer (TLB), making memory accesses faster. It can also save a bit of memory, though, as the result of the elimination of a layer of page tables. But, as it turns out, transparent huge pages can actually increase the memory usage of an application significantly under certain conditions. The good news is that a solution is at hand; it is as easy as a page full of zeroes. 

Transparent huge pages are mainly used for anonymous pages — pages that are not backed by a specific file on disk. These are the pages forming the data areas of processes. When an anonymous memory area is created or extended, no actual pages of memory are allocated (whether transparent huge pages are enabled or not). That is because a typical program will never touch many of the pages that are part of its address space; allocating pages before there is a demonstrated need would waste a considerable amount of time and memory. So the kernel will wait until the process tries to access a specific page, generating a page fault, before allocating memory for that page. 

But, even then, there is an optimization that can be made. New anonymous pages must be filled with zeroes; to do anything else would be to risk exposing whatever data was left in the page by its previous user. Programs often depend on the initialization of their memory; since they know that memory starts zero-filled, there is no need to initialize that memory themselves. As it turns out, a lot of those pages may never be written to; they stay zero-filled for the life of the process that owns them. Once that is understood, it does not take long to see that there is an opportunity to save a lot of memory by sharing those zero-filled pages. One zero-filled page looks a lot like another, so there is little value in making too many of them. 

So, if a process instantiates a new (non-huge) page by trying to read from it, the kernel still will not allocate a new memory page. Instead, it maps a special page, called simply the "zero page," into the process's address space instead. Thus, all unwritten anonymous pages, across all processes in the system, are, in fact, sharing one special page. Needless to say, the zero page is always mapped read-only; it would not do to have some process changing the value of zero for everybody else. Whenever a process attempts to write to the zero page, it will generate a write-protection fault; the kernel will then (finally) get around to allocating a real page of memory and substitute it into the process's address space at the right spot. 

This behavior is easy to observe. As Kirill Shutemov [described](<https://lwn.net/Articles/515526/>), a process executing a bit of code like this: 

```
posix_memalign((void **)&p, 2 * MB, 200 * MB);
        for (i = 0; i < 200 * MB; i+= 4096)
            assert(p[i] == 0);
        pause();
```

will have a surprisingly small resident set at the time of the `pause()` call. It has just worked through 200MB of memory, but that memory is all represented by a single zero page. The system works as intended. 

Or, it does until the transparent huge pages feature is enabled; then that process will show the full 200MB of allocated memory. A growth of memory usage by two orders of magnitude is not the sort of result users are typically looking for when they enable a performance-enhancing feature. So, Kirill says, some sites are finding themselves forced to disable transparent huge pages in self defense. 

The problem is simple enough: there is no huge zero page. The transparent huge pages feature tries to use huge pages whenever possible; when a process faults in a new page, the kernel will try to put a huge page there. Since there is no huge zero page, the kernel will simply allocate a real zero page instead. This behavior leads to correct execution, but it also causes the allocation of a lot of memory that would otherwise not have been needed. Transparent huge page support, in other words, has turned off another important optimization that has been part of the kernel's memory management subsystem for many years. 

Once the problem is understood, the solution isn't that hard. Kirill's patch adds a special, zero-filled huge page to function as the huge zero page. Only one such page is needed, since the transparent huge pages feature only uses one size of huge page. With this page in place and used for read faults, the expansion of memory use simply goes away. 

As always, there are complications: the page is large enough that it would be nice to avoid allocating it if transparent huge pages are not in use. So there's a lazy allocation scheme; Kirill also added a reference count so that the huge zero page can be returned if there is no longer a need for it. That reference counting slows a read-faulting benchmark by 1%, so it's not clear that it is worthwhile; in the end, the developers might conclude that it's better to just keep the zero huge page around once it has been allocated and not pay the reference counting cost. This is, after all, a situation that [has come about before](<https://lwn.net/Articles/340370/>) with the (small) zero page. 

There have not been a lot of comments on this patch; the implementation is relatively straightforward and, presumably, does not need a lot in the way of changes. Given the obvious and measurable benefits from the addition of a huge zero page, it should be added to the kernel sometime in the fairly near future; the 3.8 development cycle seems like a reasonable target.
