---
title: "Working-set protection for anonymous pages"
url: https://lwn.net/Articles/815342/
date: "March 19, 2020"
category: "Memory management-Page replacement algorithms"
author: "By Jonathan Corbet March 19, 2020"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
March 19, 2020

The kernel's memory-management subsystem goes to great lengths to keep the pages that are actually in use in memory. But sometimes it gets things wrong, leading to reduced performance or, in the worst cases, flat-out thrashing. We may be about to see a significant improvement, though, thanks to [a patch set](<https://lwn.net/ml/linux-kernel/1584423717-3440-1-git-send-email-iamjoonsoo.kim@lge.com/>) from Joonsoo Kim changing how anonymous pages (those containing data not backed by files on disk) are managed. As it turns out, all that had to be done was to make use of some work that already exists in related parts of the memory-management code. 

#### LRU lists

A bit of background may be helpful for understanding how this patch set works; we'll start with a highly oversimplified picture, then add some details as we go. 

Virtual-memory systems allow applications to address far more memory than can actually fit into the physical memory installed in the system, so a significant part of any given process's address space is likely to exist only on secondary storage at any given time. Obviously, the pages that are in physical memory should be the ones that the process is going to use in the near future. The kernel cannot know for sure which pages will be useful, though, so it must fall back onto a set of heuristics that allow it to guess as well as it can. 

Some of those heuristics are reasonably straightforward. For example, if a process is observed to be working through a file sequentially, chances are pretty good that it will soon be wanting the pages of the file immediately after those it is accessing now. Another heuristic, found at the core of almost any virtual-memory implementation, is that pages that have been used recently are likely to be used in the future, while those that have languished unused for a while may not be worth keeping around. 

To implement that last approach, the kernel maintains a "least-recently used" (LRU) list; all user-space pages in physical memory are kept on that list. The kernel occasionally checks the pages on the LRU list and moves those that have been accessed recently to the head of the list. When more memory is needed, to bring in pages from secondary storage, for example, pages at the tail end of the list are reclaimed. 

In truth, the kernel maintains more than one LRU list. To begin with, the "LRU list" is actually two lists: the "active" and "inactive" lists. The active list functions mostly as described in the previous paragraph, except that, when pages fall off the tail of the list, they are put onto the inactive list instead. At that point, the protections on those pages are set to disallow all user-space access. Should some process access one of those pages, a "soft" page fault will result; the page will be made accessible again and returned to the active list. When memory is needed, pages will be reclaimed from the inactive list. 

The inactive list thus serves as a sort of second chance for pages that are on their way out. But it also plays another role: managing pages that are highly likely to only be used once. A classic example is a process reading through a file; the pages read will probably be processed and not be needed again. These pages probably should not push out memory that _is_ likely to be useful in the future. The kernel handles this case by putting newly faulted, file-backed pages directly onto the inactive list; they will only move to the active list if they are accessed again before being reclaimed. 

In truth, things are more complicated that that; among other things, there are more than two LRU lists. Relevant here is the fact that there are separate active and inactive lists for file-backed and anonymous pages. It is common to reclaim file-backed pages before anonymous pages, since the former often need not be written back (while anonymous pages must always be written to swap) and can be easier to get back if need be. It is also worth noting that, in the case of file-backed pages, the kernel maintains "shadow entries" to remember (for a while) the existence of pages that have been reclaimed off the inactive list. If those pages are "refaulted" back in, the kernel knows that it's pushing out useful pages and can make adjustments to try to avoid doing that. 

#### Improving anonymous LRU-list behavior

Kim's patch set addresses two significant differences between how anonymous and file-backed pages are handled. One of those is that, while file-backed pages are faulted into the inactive list as described above, anonymous pages go directly to the active list. If an application faults in a lot of anonymous pages, it will likely push other useful pages off the active list onto the inactive list. If the newly faulted pages are only used once, though, they will push aside other, more useful pages needlessly. To address this, Kim's patch set causes anonymous pages to be faulted into the inactive list, just like file-backed pages are. If those pages are truly useful, they will be promoted to the active list when the soft fault happens; otherwise they can be reclaimed relatively quickly. 

The other change addresses the fact that refault tracking, in current kernels, is only done for the file-backed LRU list. Once an anonymous page is reclaimed, the kernel forgets about its history. As it turns out, the previous change (faulting pages into the inactive list) can exacerbate some bad behavior: adding new pages to the inactive list can quickly push out other pages that were just faulted in before they can be accessed a second time and promoted to the active list. If refault tracking were done for the anonymous LRU list, this situation could be detected and dealt with. 

So the patch set adds this tracking for anonymous pages. In a sense the work was straightforward, since the infrastructure for refault tracking already exists and can be reused; it simply needs to be extended to track more than one LRU list. There are some added details, though. Since anonymous pages are written to swap when they are reclaimed, the shadow LRU entry used to track refaults can be written there as well rather than being kept in RAM, for example. 

Kim included a number of benchmarks showing how these patches improve memory-management behavior in various situations. What really [got the attention](<https://lwn.net/ml/linux-kernel/20200226193942.30049da9c090b466bdc5ec23@linux-foundation.org/>) of memory-management maintainer Andrew Morton, though, was [this automated test result](<https://lore.kernel.org/linux-mm/20200227022905.GH6548@shao2-debian/>) showing a 400% improvement in a virtual-memory scalability test. He asked: ""One wonders why on earth we weren't doing these things in the first place?"" Kim [replied](<https://lwn.net/ml/linux-kernel/20200227074748.GA18113@js1304-desktop/>) with copies of the patches adding the current behavior in 2002 — written by a certain Andrew Morton, who [acknowledged](<https://lwn.net/ml/linux-kernel/20200229204000.298de32521885b8af858a50d@linux-foundation.org/>) that it may well be time to revisit some of that work. 

There does not appear to be any opposition to this work in the memory-management community. That does not necessarily mean that it will be merged soon; memory-management patches often require a lot of testing and review before developers become confident enough to apply them. That is doubly true of patches affecting heuristics, since they can often cause unexpected problems in surprising places. These patches have not even made it into the -mm tree yet, a step that would increase both testing and review. So, even though Morton has [said](<https://lwn.net/ml/linux-kernel/20200227153639.951d6a42080e8d4227872e64@linux-foundation.org/>) that ""given all the potential benefits, perhaps I should be more aggressive here"", this work doesn't look like 5.7 material. It may well find its way upstream shortly thereafter, though.
