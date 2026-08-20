---
title: "Merging the multi-generational LRU"
url: https://lwn.net/Articles/894859/
date: "May 12, 2022"
category: "Memory management-Page replacement algorithms"
author: "By Jonathan Corbet May 12, 2022 LSFMM"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
May 12, 2022

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2022/>)

Many types of kernel changes can be hammered into shape on the mailing lists. There are certain types of patches, however, that have a hard time getting to the finish line that way; they are sufficiently large and invasive that they need an actual gathering of the developers involved. The [multi-generational LRU work](<https://lwn.net/Articles/856931/>) (MGLRU) falls into this category, which is why it was the subject of a full-hour session at the [2022 Linux Storage, Filesystem, Memory-management and BPF Summit](<https://events.linuxfoundation.org/lsfmm/>) (LSFMM). The discussion held there may well have opened the doors for this code to be merged in the near future. 

#### MGLRU introduction

The session was led by Yu Zhao, the developer of the MGLRU work. He started by saying that RAM performance will continue to play a major role in the performance of our systems as a whole. Getting the best performance, he said, requires overcommitting memory, leading to a couple of problems. One of those is deciding which objects should be present in memory at any given time; that is what MGLRU is for. Increasing the number of pages that can be kept in memory is also an active area of work — defragmentation and zram, for example — but he wasn't there to talk about those. [![\[Yu Zhao\]](https://static.lwn.net/images/conf/2022/lsfmm/YuZhao-sm.png)](<https://lwn.net/Articles/894872/>)

He provided an introduction to MGLRU, which is also described in the above-linked article. At its core, it divides memory into a number of buckets called "generations". A page's generation reflects its "age" — how long it has been since the page was last accessed. The management of these pages is done by a mechanism Zhao described as a "clock with two hands". The aging hand scans the accessed bit of pages to see if they have been used since the last scan; pages that have been used are marked to be moved to the youngest generation. The eviction hand will actually move pages to the correct generation; those that end up in the oldest generation are the coldest and can be considered for reclaim. 

One of the interesting design decisions behind MGLRU is that its scanning walks through process page tables rather than scanning physical memory. This is partly for efficiency, Zhao said; the LRU walk in current kernels is constantly having to switch between different process's page tables, which creates cache misses and slows things down. The problem with walking page tables, though, is that they can be sparse, with a lot of empty entries; scanning those brings no benefit. So the MGLRU code includes a [bloom filter](<https://en.wikipedia.org/wiki/Bloom_filter>) that helps it to avoid walking page-table pages that contain few active entries. The MGLRU code also tries to learn from its mistakes by noticing when pages it reclaims are quickly brought back into memory. To that end, it incorporates a [proportional-integral-derivative (PID) controller](<https://en.wikipedia.org/wiki/PID_controller>) to redirect its attention when it seems to be making the wrong decisions. 

Johannes Weiner started the discussion by asking about how the aging works. If a page in the oldest generation is seen to be accessed, is it moved to the youngest generation, or just to the next-younger generation? Zhao answered that it actually depends on the type of access. If the page was accessed via a page table, then it goes to the youngest generation; if, instead, it was accessed via a file descriptor, it only goes up one generation. There are two reasons for that: the cost of evicting file-backed pages is lower, and the system can see every access (since they are done through the kernel), while accesses via page tables can only be observed once on every scan. Andrew Morton asked whether the dirtyness of a page is factored in; the answer was that dirty pages are moved up one generation. 

#### Extensions

Weiner continued by noting that, in general, generational garbage-collection algorithms try to look at how long objects have been in use. Everything starts in the oldest generation, and becomes less likely to be reclaimed over time if it is used. The MGLRU, though, starts everything in the youngest generation, and will also promote pages directly there. What, he asked, do generations buy when they are managed this way? 

The answer was a bit surprising: it seems that the full mechanism for moving pages between generations is not yet in place. When MGLRU was first posted, Zhao said, it was called a "framework". There are a lot of different use cases out there, from servers to phones and more, and there is a lot of variety even within a single category like phones. Coming up with a generation-assignment algorithm that works everywhere would be a challenge, so MGLRU will allow it to be customized. There will BPF hooks that will be called on each page needing generation assignment; they will be provided with the associated process ID, the page address, its type (anonymous or file-backed), and, for page faults, the type of the fault. The called program can then tell the memory-management subsystem which generation the page should be placed in. The networking subsystem, Zhao continued, started with a single congestion-control algorithm. The number of those algorithms has grown over time, and now their implementation is moving to BPF. The MGLRU, he said, is heading down the same path. 

Weiner admitted that he hadn't known about this aspect of the MGLRU. Zhao said that this machinery is not in the current patch posting, but should probably be added. 

A future MGLRU feature, Zhao continued, could be detection of internal fragmentation with transparent huge pages. There are a lot of applications that suggest turning off this feature now; if their memory is sparsely accessed, using huge pages can end up wasting a lot of memory. He said that Redis and memcached are among the applications that are affected by this. The problem is that access to a single base page can make an entire huge page appear to be hot, potentially wasting up to 511 base pages in each huge page. 

Internal fragmentation of huge pages can be detected by initially mapping them using base-page entries, then watching the access pattern with MGLRU. If most of the base pages end up in the younger generations (and are thus being used), the mapping can be turned into a huge-page mapping; otherwise, the unused pages can simply be reclaimed. Michal Hocko asked whether this code exists now; the answer was "no, but it is likely to happen within the next four years". Hocko then suggested focusing on the code that is being considered now. Before allowing that to happen, Zhao suggested that ballooning for virtual machines is another potential extension, allowing unused pages to be taken away. 

#### Enabled by default?

Hocko said that there have been concerns about MGLRU expressed on the mailing lists. He asked where things should go from here. MGLRU has some nice potential for extension, he said, but the current LRU implementation has been improved over many years and benefits from a lot of accumulated experience. He suggested that MGLRU could be merged alongside the existing LRU with an opt-in approach. Merging is the only way to find out how well MGLRU really works across workloads, he said, but he was nervous about switching over to it by default. 

That said, he continued, perhaps enabling by default could be considered; that would be a "trial by fire" for both the code and its developer. It would obviously be necessary to tell users clearly how to turn it off. There are advantages to both approaches, he said, and maintaining two LRUs will have a huge cost for as long as it lasts. What, he asked, can the group agree on? 

Mel Gorman said that, if this code is merged, it should be enabled by default. That said, he worried that most distributions would _not_ be able to enable it because the MGLRU currently places a relatively low limit on the maximum number of CPUs that the kernel can support. Zhao said that this limitation would be removed in the next version of the patch set. Part of the problem with CPU counts is evidently the number of page flags that MGLRU needs; Zhao suggested that he had a way to free up some page flags, provoking curiosity and raised eyebrows in the group. There followed a digression on how this might be done that didn't reach any firm conclusions. 

Bringing the discussion back into focus, Gorman said that his testing shows that MGLRU performs reasonably well; it is better on single-node machines than on NUMA systems, though. 

Morton said that, if this code is going to succeed, it will start with a relatively small number of users. It will get better over time as the problems are addressed, and people will start switching over to it. He worried, though, that the development history of the MGLRU code is hidden, and that the code itself is "inscrutable"; he suggested putting a lot of time into internal documentation. Zhao, Morton said, needs to "tell a story" to bring developers up to speed. 

Continuing, Morton said that the current LRU still appears to perform better for some workloads. The kernel still has multiple slab allocators, but he would rather not do that again; MGLRU should be made better for all workloads. On the other hand, we don't have that concern for filesystems; we encourage users to choose between them. Perhaps the same could be done for the LRU. 

He concluded that he could envision merging this code "in the next cycle", but it is going to be a challenge. Adding MGLRU takes developers who have worked on memory management for decades and "turns them into new hires" who will have to face a complex code base with no comments, and with nobody in the next cubicle to ask about something they don't understand. 

Zhao said that he has been having a hard time getting users to try MGLRU without it being upstream. This has evidently been a problem even within Google. His group ended up hiring an outside firm to do the benchmarking on this code. 

#### "Expect a few bug reports"

Hocko said that, if this code is merged, nobody seems to object to enabling it by default. He told Zhao to "expect a few bug reports" when that happens, and asked whether Zhao was prepared for a massive amount of work. Is he prepared to see the whole thing reverted if he can't keep up with that work? Morton suggested enabling MGLRU in linux-next for a single day to see what happens, but Gorman said that it wouldn't be possible to get useful information in that little time "unless it's an outright failure". Real memory-management issues tend to be more subtle and take more time to come out, he said; the only way to find them will be for major distributors to enable MGLRU by default. 

Zhao sought to reassure the group that Google would continue to support this work; he said that both Android and the data center would start to use it once it's merged. The budget for this work has been planned for years ahead, he said. He has also been working with other companies, in private, to get them to use it, and will continue to do so. Hopefully he will eventually be able to take a break, someday, but not soon. 

Morton said that merging it disabled by default can still have a lot of value, especially if Google can put resources behind testing and improving it. Gorman said that there are no good choices; if it is not enabled by default, most people won't use it. He is not opposed to merging this code, but would like to see it enabled. When transparent huge pages were merged, he said, the feature was enabled by default and "set everything on fire". It took three years to sort it all out, but without having been enabled for all users, it would never have been fixed. 

Zhao said he would prefer to follow the process used at Google, which involves starting with a small group of users and slowly ramping up. Gorman answered that an approach like that is good for a fleet, but it doesn't help major distributors decide when to make a switch. Even when something works in the whole fleet, it doesn't mean that it will work in the general case. Weiner said that some distributors would turn MGLRU on even if it were disabled by default; he mentioned Arch in particular. That might be a good way to avoid a "total flag day". Matthew Wilcox agreed, saying that enabling transparent huge pages by default was actually the wrong thing to do. Documentation lives forever, he said, and vendors are still telling users to disable transparent huge pages even though the problems have long since been fixed. 

Zhao said that he could live with MGLRU by default, but then it could become a problem for others; if he breaks things, users will suffer. So he thinks that is a risky approach; switching to MGLRU by default after a year might be better. Weiner said that, if MGLRU is off by default, there should be a set time frame to enable it — a maximum of a couple of development cycles. 

At that point, a rather tired set of memory-management developers called an end to the session, and to the day. It seems highly likely that this work will be merged in the near future, though Morton's suggestion of doing it for 5.19 might strike others as a bit hasty. Whether it will be enabled for all users, though, is far from clear.
