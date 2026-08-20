---
title: "The multi-generational LRU"
url: https://lwn.net/Articles/851184/
date: "April 2, 2021"
category: "Memory management-Page replacement algorithms"
author: "By Jonathan Corbet April 2, 2021"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
April 2, 2021

One of the key tasks assigned to the memory-management subsystem is to optimize the system's use of the available memory; that means pushing out pages containing unused data so that they can be put to better use elsewhere. Predicting which pages will be accessed in the near future is a tricky task, and the kernel has evolved a number of mechanisms designed to improve its chances of guessing right. But the kernel not only often gets it wrong, it also can expend a lot of CPU time to make the incorrect choice. The [multi-generational LRU patch set](<https://lwn.net/ml/linux-kernel/20210313075747.3781593-1-yuzhao@google.com/>) posted by Yu Zhao is an attempt to improve that situation. 

In general, the kernel cannot know which pages will be accessed in the near future, so it must rely on the next-best indicator: the set of pages that have been used recently. Chances are that pages that have been accessed in the recent past will be useful again in the future, but there are exceptions. Consider, for example, an application that is reading sequentially through a file. Each page of the file will be put into the page cache as it is read, but the application will never need it again; in this case, recent access is not a sign that the page will be used again soon. 

The kernel tracks pages using a pair of least-recently-used (LRU) lists. Pages that have been recently accessed are kept on the "active" list, with just-accessed pages put at the head of the list. Pages are taken off the tail of the list if they have not been accessed recently and placed at the head of the "inactive" list. That list is a sort of purgatory; if some process accesses a page on the inactive list, it will be promoted back to the active list. Some pages, like those from the sequentially read file described above, start life on the inactive list, meaning they will be reclaimed relatively quickly if there is no further need for them. 

There are more details, of course. It's worth noting that there are actually two pairs of lists, one for anonymous pages and one for file-backed pages. If memory control groups are in use, there is a whole set of LRU lists for each active group. 

Zhao's patch set identifies a number of problems with the current state of affairs. The active/inactive sorting is too coarse for accurate decision making, and pages often end up on the wrong lists anyway. The use of independent lists in control groups makes it hard for the kernel to compare the relative age of pages across groups. The kernel has a longstanding bias toward evicting file-backed pages for a number of reasons, which can cause useful file-backed pages to be tossed while idle anonymous pages remain in memory. This problem has gotten worse in cloud-computing environments, where clients have relatively little local storage and, thus, relatively few file-backed pages in the first place. Meanwhile, the scanning of anonymous pages is expensive, partly because it uses a complex [reverse-mapping mechanism](<https://lwn.net/Articles/77106/>) that does not perform well when a lot of scanning must be done. 

#### Closing the generation gap

The multi-generational LRU patches try to address these problems with two fundamental changes: 

  * Add more LRU lists to cover a range of page ages between the current active and inactive lists; these lists are called "generations". 
  * Change the way page scanning is done to reduce its overhead. 

Newly activated pages are assigned to the youngest generation (though there are some exceptions described below). Over time, the memory-management subsystem will scan over a process's pages to determine whether each has been used since the last scan; any that have remained idle are moved to the next older generation. Pages of any generation that show activity are moved back to the youngest generation. 

The result of this work is a spectrum of page ages, from those quite recently accessed to those that have not been used in some time. The number of generations can be configured into the kernel; that number seems to be as small as four for phones to several times that for cloud-based servers. 

When the time comes to reclaim pages, only the oldest generation need be considered. The "oldest generation" can be different for anonymous and file-backed pages; anonymous pages can be harder to reclaim in general (they must always be written to swap) and the new code retains some of the bias toward reclaiming file-backed pages more aggressively. So file-backed pages may not escape reclaim for as many generations as anonymous pages do. The current patch only allows reclaim of file-backed pages to get one generation ahead of that for anonymous pages, though. 

The multi-generational mechanism, it is claimed, is more accurate than the current two-list approach; by the time a page makes it to the oldest generation, its chances of being unneeded are rather higher than they are for pages on the inactive list. That, in turn, means that these pages can be reclaimed more aggressively, making more memory available for tasks that will actually make use of it. This mechanism allows for ready comparison of the ages of anonymous and file-backed pages, and, by tracking the creation time of each generation, of the ages of pages in different control groups; this information is lost in current kernels. That, in turn, makes it easier to identify and reclaim idle anonymous pages. 

The other claimed advantage is in the change to how pages are scanned. Pages are accessed via the page-table entries (PTEs) in every process that has them mapped; the "recently accessed" bit lives in those page-table entries. Current kernels, though, scan through the pages themselves, and must use reverse-mapping to find and check the associated PTEs; that is expensive. The multi-generational LRU code, instead, scans over PTEs directly, an approach with better locality. A hook in the scheduler helps to track processes that have actually run since the last scan, so idle processes can be skipped. 

The multi-generational LRU also benefits from skipping many of the heuristics that are used in current kernels to decide which pages should be reclaimed. There are still a few, though. For example, when a page is first established, its generation is picked with these rules: 

  * Pages that are being faulted in are assigned to the youngest generation, as one would expect. 
  * The activation of pages that are unmapped (pages resident in memory but with no PTEs pointing to them; these can include pages chosen for reclaim but not actually reclaimed before being referenced again) are added to the second-youngest generation. This is seemingly done to avoid making the youngest generation look too big, which might delay further page scanning until the next generation can be created. 
  * Pages that are being reclaimed, but which must persist while their contents are written to backing store, are added to the second-oldest generation. That prevents another attempt to reclaim them while the writeback is underway. 
  * Pages that are being deactivated go into the oldest generation. That is also the fate of pages that were brought in by the readahead mechanism; reading those pages is a speculative act on the kernel's part in the first place, with no guarantee that they will ever be useful. 

There are a few knobs exported to user space to control this mechanism, including the ability to turn the multi-generational code off entirely; see [this documentation patch](<https://lwn.net/ml/linux-kernel/20210313075747.3781593-15-yuzhao@google.com/>) for more information. 

#### Generational change

The end result of all this work, it is claimed, is that page reclaim is much more efficient and better targeted than before. Systems like Android, when using this code, record fewer low-memory kills (when an app process is killed due to memory pressure), Chrome OS shows fewer out-of-memory kills, and server systems are better able to use available memory. It looks like an improvement all around. 

Given that, one might wonder why the multi-generational algorithm is kept separate from the rest of the memory-management code and is made optional. It is, in essence, an independent approach to page aging and reclaim that exists alongside the current LRU lists. The answer, presumably, is that there are a lot of workloads out there, and some of them may not benefit from the multi-generational approach. There will need to be a lot more testing done to learn where the multi-generational LRU falls down and what might need to be done to keep that from happening. 

The multi-generational LRU might eventually win over the memory-management developers, most of whom have not yet commented on this patch set. It does seem likely, though, that it will need to demonstrate better performance (or at least a lack of performance regressions) across the bulk of the workloads out there, to the point that it could be considered as a replacement for the current LRU rather than an addition to it. The idea of maintaining two separate LRU schemes is going to be a hard sell in the kernel community; it would be far better to just switch over completely to the multi-generational LRU if it is truly better. 

Answering that question is certain to be a long process. Even relatively small memory-management changes can take a while to merge; it is just too easy to introduce performance penalties for some users. This change is not "relatively small", so the bar for inclusion will be high. But if the multi-generational LRU lives up to its claims, it may just be able to clear that bar — eventually.
