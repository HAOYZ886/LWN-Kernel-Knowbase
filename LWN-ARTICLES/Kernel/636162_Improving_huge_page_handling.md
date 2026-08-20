---
title: Improving huge page handling
url: https://lwn.net/Articles/636162/
date: "March 11, 2015"
category: "Huge pages; Memory management-Huge pages"
author: "By Jonathan Corbet March 11, 2015 LSFMM 2015"
---

By **Jonathan Corbet**  
March 11, 2015

* * *

[LSFMM 2015](<https://lwn.net/Articles/lsfmm2015/>)

The "huge page" feature found in most contemporary processors enables access to memory with less stress on the translation lookaside buffer (TLB) and, thus, better performance. Linux has supported the use of huge pages for some years through both the [hugetlbfs](<https://lwn.net/Articles/374424/>) and [transparent huge pages](<https://lwn.net/Articles/423584/>) features, but, as was seen in the two sessions held during the memory-management track at [LSFMM 2015](<http://events.linuxfoundation.org/events/linux-storage-filesystem-and-mm-summit>), there is still considerable room for improvement in how this support is implemented. 

[![\[Kirill Shutemov\]](https://static.lwn.net/images/conf/2015/lsfmm/KirillShutemov-sm.jpg)](<https://lwn.net/Articles/636169/>) Kirill Shutemov started off by describing his proposed changes to how reference counting for transparent huge pages is handled. This patch set was described in detail in [this article](<https://lwn.net/Articles/619738/>) last November and has not changed significantly since. The key part of the patch is that it allows a huge page to be simultaneously mapped in the PMD (huge page) and PTE (regular page) modes. It is, as he acknowledged, a large patch set, and there are still some bugs, so it is not entirely surprising that this work has not been merged yet. 

One remaining question has to do with partial unmapping of huge pages. When a process unmaps a portion of a huge page, the expected behavior is to split that page up and return the individual pages corresponding to the freed region back to the system. It is also possible, though, to split up the mapping while maintaining the underlying memory as a huge page. That keeps the huge page together and allows it to be quickly remapped if the process decides to do so. But that also means that no memory will actually be freed, so it is necessary to add the huge page to a special list where it can be truly split up should the system experience memory pressure. 

Deferred splitting also helps the system to avoid another problem: currently there is a lot of useless splitting of huge pages when a process exits. There was some talk of trying to change `munmap()` behavior at exit time, but it is not as easy as it seems, especially since the exiting process may not hold the only reference to any given huge page. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

Hugh Dickins, the co-leader of the session, pointed out that there is one complication with regard to Kirill's patch set: he is not the only one who is working with simultaneous PMD and PTE mappings of huge pages. Hugh recently posted [a patch set](<https://lwn.net/Articles/634334/>) of his own adding transparent huge page support to the tmpfs filesystem. This work contains a number of the elements needed for full support [![\[Hugh Dickins\]](https://static.lwn.net/images/conf/2015/lsfmm/HughDickins-sm.jpg)](<https://lwn.net/Articles/636170/>) for huge pages in the page cache (which is also an eventual goal of Kirill's patches). But Hugh's approach is rather different, leading to some concern in the user community; in the end, only one of these patch sets is likely to be merged. 

Hugh's first goal is to provide a more flexible alternative for users of the hugetlbfs filesystem. But his patches diverge from the current transparent huge page implementation (and Kirill's patches) in a significant way: they completely avoid the use of "compound pages," the mechanism used to bind individual pages into a huge page. Compound pages, he said, were a mistake to use with transparent huge pages; they are too inflexible for that use case. Peter Zijlstra suggested that, if this is really the case, Hugh should look at moving transparent huge pages away from compound pages; Hugh expressed interest but noted that available time was in short supply. 

Andrea Arcangeli (the original author of the transparent huge pages feature) asked Hugh to explain the problems with compound pages. Hugh responded that the management of page flags is getting increasingly complicated when huge pages are mapped in the PTE mode. So he decided to do everything in tmpfs with ordinary 4KB pages. Kirill noted that this approach makes tmpfs more complex, but Hugh thought that was an appropriate place for the complexity to be. 

When it comes to bringing huge page support to the page cache, though, it's not clear where the complexity should be. Hugh dryly noted that filesystem developers already have enough trouble with the memory-management subsystem without having to deal with more complex interfaces for huge page support. He was seemingly under the impression that there is not a lot of demand for this support from the filesystem side. Btrfs developer Chris Mason said, though, that he would love to find ways to reduce overhead on huge-memory systems, and that huge pages would help. Matthew Wilcox added that there are users even asking for filesystem support with extra-huge (1GB) pages. 

Rik van Riel jumped in to ask if there were any specific questions that needed to be answered in this session. Hugh returned to the question of whether filesystems need huge page support and, if so, what form it should take, but not much discussion of that point ensued. There was some talk of Hugh's tmpfs work; he noted that one of the hardest parts was support for the `mlock()` system call. There is a lot of tricky locking involved; he was proud to have gotten it working. 

In a brief return to huge page support in the page cache, it was noted that Kirill's reference-counting work can simplify things considerably; Andrea said it was attractive in many ways. 

There was some talk of what to do when an application calls `madvise()` on a portion of a huge page with the `MADV_DONTNEED` command. It would be nice to recover the memory, but that involves an expensive split of the page. Failure to do so can create problems; they have been noted in particular with the [jemalloc](<http://www.canonware.com/jemalloc/>) implementation of `malloc()`. See [this page](<http://dev.nuodb.com/techblog/linux-transparent-huge-pages-jemalloc-and-nuodb>) for a description of these issues. 

Even if a page is split when `madvise(MADV_DONTNEED)` is called on a portion of it, there is a concern that the kernel might come around and "collapse" it back into a huge page. But Andrea said this should not be a problem; the kernel will only collapse memory into huge pages if the memory around those pages is in use. But, in any case, he said, user space should be taught to use 2MB pages whenever possible. Trying to optimize for 4KB pages on current systems is just not worth it and can, as in the jemalloc case, create problems of its own. 

The developers closed out this session by agreeing to look more closely at both approaches. There is a lot of support for the principles behind Kirill's work. Hugh complained that he hasn't gotten any feedback on his patch set yet. While the patches are under review, Kirill will look into extending his work to the tmpfs filesystem, while Hugh will push toward support for anonymous transparent huge pages. 

#### Compaction

The topic of huge pages returned on the second day, however, when Vlastimil Babka ran a session focused primarily on the costs of compaction. The [memory compaction](<https://lwn.net/Articles/368869/>) code moves pages around to create large, physically contiguous regions of free memory. These regions can be used to support large allocations in general, but they are especially useful for the creation of huge pages. 

The problem comes in when a process incurs a page fault, and the kernel attempts to resolve it by allocating a huge page. That task can involve running compaction which, since it takes a while, can create significant latencies for the faulting process. The cost can, in fact, outweigh the performance benefits of using huge pages in the first place. There are ways of mitigating this cost, but, Vlastimil wondered, might it be better [![\[Vlastimil Babka\]](https://static.lwn.net/images/conf/2015/lsfmm/VlastimilBabka-sm.jpg)](<https://lwn.net/Articles/636172/>) to avoid allocating huge pages in response to faults in the first place? After all, it is not really known whether the process needs the entire huge page or not; it's possible that much of that memory might be wasted. It seems that this happens, once again, with the jemalloc library. 

Since it is not possible to predict the benefit of supplying huge pages at fault time, Vlastimil said, it might be better to do a lot less of that. Instead, transparent huge pages should mostly be created in the `khugepaged` daemon, which can look at memory utilization and collapse pages in the background. Doing so requires redesigning `khugepaged`, which was mainly meant to be a last resort filling in huge pages when other methods fail. It scans slowly, and can't really tell if a process will benefit from huge pages; in particular, it does not know if the process will spend a lot of time running. It could be that the process mostly lurks waiting for outside events, or it may be about to exit. 

His approach is to improve `khugepaged` by moving the scanning work that looks for huge page opportunities into process context. At certain times, such as on return from a system call, each process would scan a bit of its memory and, perhaps, collapse some pages into huge pages. It would tune itself automatically based partially on success rate, but also simply based on the fact that a process that runs more often will do more scanning. Since there is no daemon involved, there are no extra wakeups; if a system is wholly idle, there will be no page scanning done. 

Andrea protested, though, that collapsing pages in `khugepaged` is far more expensive than allocating huge pages at fault time. To collapse a page, the kernel must migrate (copy) all of the individual small pages over to the new huge page that will contain them; that takes a while. If the huge page is allocated at page fault time, this work is not needed; the entire huge page can be faulted in at once. There might be a place for process-context scanning to create huge pages before they are needed, but it would be better, he said, to avoid collapsing pages whenever possible. 

Vlastimil suggested allocating huge pages at fault time but only mapping the specific 4KB page that faulted; the kernel could then observe utilization and collapse the page in-place if warranted. But Andrea said that would needlessly deprive processes of the performance benefits that come from the use of huge pages. If we're going to support this feature in the kernel, we should use it fully. 

Andi Kleen said that running memory compaction in process context is a bad idea; it takes away opportunities for parallelism. Compaction scanning should be done in a daemon process so that it can run on a separate core; to do otherwise would be to create excessive latency for the affected processes. Andrea, too, said that serializing scanning with execution was the wrong approach; he suggested putting that work into a workqueue instead. But Mel Gorman said he would rather see the work done in process context so that it can be tied to the process's activity. 

At about this point the conversation wound down without having come to any firm conclusions. In the end, this is the sort of issue that is resolved over time with working code.
