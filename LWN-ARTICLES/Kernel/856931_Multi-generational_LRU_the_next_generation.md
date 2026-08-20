---
title: "Multi-generational LRU: the next generation"
url: https://lwn.net/Articles/856931/
date: "May 24, 2021"
category: "Memory management-Page replacement algorithms"
author: "By Jonathan Corbet May 24, 2021"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
May 24, 2021

The multi-generational LRU patch set is a significant reworking of the kernel's memory-management subsystem that promises better performance for a number of workloads; it was [covered here](<https://lwn.net/Articles/851184/>) in April. Since then, two new versions of that work have been released by developer Yu Zhao, with [version 3](<https://lwn.net/ml/linux-kernel/20210520065355.2736558-1-yuzhao@google.com/>) being posted on May 20\. Some significant changes have been made since the original post, so another look is in order. 

As a quick refresher: current kernels maintain two least-recently-used (LRU) lists to track pages of memory, called the "active" and "inactive" lists. The former contains pages thought to be in active use, while the latter holds pages that are thought to be unused and available to be reclaimed for other uses; a fair amount of effort goes into deciding when to move pages between the two lists. The multi-generational LRU generalizes that concept into multiple generations, allowing pages to be in a state between "likely to be active" and "likely to be unused". Pages move from older to newer generations when they are accessed; when memory is needed pages are reclaimed from the oldest generation. Generations age over time, with new generations being created as the oldest ones are fully reclaimed. 

That summary oversimplifies a lot of things; see the above-linked article for a more detailed description. 

#### Multi-tier, multi-generational LRU

Perhaps the largest change since the first posting of this work is the concept of "tiers", which are used to subdivide the generations of pages which, in turn, facilitates better decisions about which pages to reclaim, especially on systems where a lot of buffered I/O is taking place. Specifically, tiers are a way of sorting the pages in a generation by the frequency of accesses — but only accesses made by way of file descriptors. When a page first enters a generation, it normally goes into tier 0\. If some process accesses that page via a file descriptor, the page's usage count goes up and it will move to tier 1\. Further accesses will push the page into higher tiers; the actual tier number is the base-2 log of the usage count. 

Before looking at how these tiers are used, it is worth asking why they are managed this way — why are only file-descriptor-based accesses counted? One possible reason is never mentioned in the patch set or discussion, but seems plausible: accesses via file-descriptor will happen as the result of a system call and are relatively easy and cheap to count. Direct accesses to memory by the CPU are more costly to track and cannot reasonably be monitored with the same resolution. 

The other reason, though, is that this mechanism enables some changes to how the aging of pages brought in via I/O is done. In current kernels, a page that is brought into memory as the result of, say, a `read()` call will initially be added to the inactive list. This makes sense because that page will often never be used again. Should there be another access to the page, though, it will be made active and the kernel will try to avoid reclaiming it. This mechanism works better than its predecessors, but it is still possible for processes doing a lot of I/O to flush useful pages out of the active list, hurting the performance of the system. 

Doing better involves making use of the existing shadow-page tracking in the kernel. When pages are reclaimed for another use, the kernel remembers, for a while, what those pages contained and when the old contents were pushed out. If one of those pages is accessed again in the near future, requiring it to be brought back in from secondary storage, the kernel will notice this "refault", which is a signal that actively used pages are being reclaimed. As a general rule, refaults indicate thrashing, which is not a good thing. The kernel can respond to excessive refaulting by, for example, making the active list larger. 

The multi-generational LRU work tweaks the shadow entries to record which tier a page was in when it was reclaimed. If the page is refaulted, it can be restored to its prior tier, but the refault can also be counted in association with that tier. That allows the computation of the refault rate for each tier — what percentage of pages being reclaimed from that tier are being subsequently refaulted back into memory? It seems evident that refaults on pages in higher tiers — those which are being accessed more frequently — would be worth avoiding in general. 

This refault information is used by comparing the refault rates of the higher tiers against that of tier 0, which contains, remember, pages that are accessed directly by the CPU and pages that have not been accessed at all. If the higher tiers have a refault rate that is higher than the tier 0 rate, then pages in those tiers are moved to a younger generation and thus protected (for a while) from reclaim. That has the effect of focusing reclaim on the types of pages that are seeing fewer refaults. 

The other piece of the puzzle is that the memory-management code no longer automatically promotes pages on the second file-descriptor-based access, as is done in current kernels. Instead, pages resulting from I/O stay in the oldest generation unless they have been moved, as the result of usage, into a tier that is refaulting at a higher rate than directly accessed pages. That, as Zhao explained in [this lengthy message](<https://lwn.net/ml/linux-kernel/CAOUHufbz_f4EjtDsMkmEBbQphXj3ET+X6SM8JUPQ4b2jJmUzvA@mail.gmail.com/>), has the effect of preventing these pages from forcing out directly accessed pages that are more heavily used. That should give better performance on systems doing a lot of buffered I/O; [this remark from Jens Axboe](<https://lwn.net/ml/linux-kernel/3ddd4f8a-8e51-662b-df11-a63a0e75b2bc@kernel.dk/>) suggests that it does indeed help. 

Another change from the first version is the addition of a user-space knob to force the eviction of one or more generations. The purpose of this feature appears to be to allow job controllers to make some space available for incoming work; [this documentation patch](<https://lwn.net/ml/linux-kernel/20210520065355.2736558-15-yuzhao@google.com/>) contains a little more information. 

#### Multiple patch generations

The multi-generational LRU work remains promising, and it has garnered a fair amount of interest. Its path into the mainline kernel still looks long and difficult, though. Johannes Weiner [raised a concern](<https://lwn.net/ml/linux-kernel/YHcpzZYD2fQyWvEQ@cmpxchg.org/>) that was mentioned in the first article as well: the multi-generational LRU, as implemented now, sits alongside the existing memory-management code as a separate option, essentially giving the kernel two page-reclaim mechanisms. That will always be a hard sell for reasons described by Weiner: 

> It would be impossible to maintain both, focus development and testing resources, and provide a reasonably stable experience with both systems tugging at a complicated shared code base. 

So the new code would have to replace the existing system, which is a tall order. It would have to be shown to perform better (or, at least, not worse) for just about any workload, at a level of confidence that would motivate the replacement of code that has ""billions of hours of production testing and tuning"". The only way to do this is to merge the changes as a series of small, evolutionary steps. So the multi-generational LRU patch set would have to be broken up into a series of changes, none of which are so big that the memory-management developers don't feel that they can be safely merged. 

Over the years, the kernel has absorbed massive changes this way, but it is not a fast or easy process. Weiner suggested a couple of areas that could be focused on as a way of beginning the task of getting parts of this work upstream and making the rest easier to consider. If this advice is followed, some progress toward merging the multi-generational LRU could be made in the relatively near future. But this functionality as a whole is likely to have to go through numerous generations of revisions before it all makes it into a mainline kernel.
