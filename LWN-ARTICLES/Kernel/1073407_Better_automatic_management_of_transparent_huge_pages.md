---
title: Better automatic management of transparent huge pages
url: https://lwn.net/Articles/1073407/
date: "May 26, 2026"
category: "Huge pages; Memory management-Huge pages"
author: "By Jonathan Corbet May 26, 2026 LSFMM+BPF"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
May 26, 2026

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2026/>)

Huge pages can improve performance by increasing translation lookaside buffer (TLB) utilization and reducing memory-management overhead. _Transparent_ huge pages (THPs) are supposed to make huge-page usage, well, transparent, Nico Pache said at the beginning of his session in the memory-management track of the 2026 [Linux Storage, Filesystem, Memory Management, and BPF Summit](<https://events.linuxfoundation.org/lsfmmbpf/>). That transparency has never worked as well as many would like; he has been working on improvements to make it easier for applications to use huge pages on Linux systems. A following session, led by David Hildenbrand, was focused on how THPs could be taken away from processes that are not using them fully. 

#### Making THPs more transparent

[![\[Nico Pache\]](https://static.lwn.net/images/conf/2026/lsfmm/NicoPache-sm.png)](<https://lwn.net/Articles/1073412/>) The only way to have the system truly allocate huge pages transparently, he said, is to set the appropriate option to `always` in sysfs (see [Documentation/admin-guide/mm/transhuge.rst](<https://docs.kernel.org/admin-guide/mm/transhuge.html>) for details on how these options work), but the implementation of that mode is not optimal. If a processes touches a single byte it will get a 2MB huge page that may never be utilized to any great extent. He has, in the past, suggested [a `defer` mode](<https://lwn.net/ml/all/20250515033857.132535-1-npache@redhat.com/>) that would leave the entire task of creating THPs to the `khugepaged` kernel thread, which assembles them after the fact when it looks like they would improve performance. The consensus seemed to be that this mode wasn't the right solution, though; the memory-management developers would rather see an `auto` mode that makes better decisions. 

Pache's goal is to define that automatic mechanism; he suggested that it should behave like `always` when memory usage is below a threshold, and like `defer` otherwise. It would combine `khugepaged` (to create huge pages after allocation) and the ["underutilized" shrinker](<https://lwn.net/Articles/906511/>) (to split them apart when they are not being fully used). When the system is in the `always` mode (below the memory-use threshold) `khugepaged` would be actively trying to find candidates for promotion to a huge page. In `defer` mode, instead, the shrinker would look for huge pages to break apart again. 

Johannes Weiner commented that, if the shrinker has to run, then user space is doing something wrong. The shrinker is not great, he said, but perhaps good enough for now, especially if it can carry the system forward until user-space allocators improve. Hildenbrand wondered if there could be better ways to tell the kernel about a process's memory-use granularity; perhaps there might be a case for some sort of BPF interface. 

Another audience member asked how the kernel might help user-space allocators make better placement decisions. Tal Zussman answered that the first step is to define with the actual goal is. At allocation time, the kernel has no information on how the memory will be used, so an interface allowing user space to provide hints might help. The more interesting problem, he said, is `khugepaged`, which would benefit from information that would allow it to optimally organize memory into multi-size THPs (mTHPs). 

Matthew Wilcox said that the underutilized shrinker works by looking for base pages filled with zeroes, which is an indication that the memory was never used. It will miss memory that was used once and never touched thereafter. He said there could be benefit to adding an [`madvise()`](<https://man7.org/linux/man-pages/man2/madvise.2.html>) operation to tell the kernel that a given range of memory is not being used. There would have to be a companion "I'm using that memory again" operation as well. 

Liam Howlett said that, while hinting interfaces are good, once memory use crosses a threshold, it would be better to just assume that THPs will be useful. Lorenzo Stoakes, though, said that the existence of various hinting interfaces shows that, at times, it is better to be a bit less transparent. Hildenbrand said that the problem with hinting interfaces is that the kernel does not always have a place to store those hints. Wilcox added that the problem with automatic thresholds is that, on Linux, the page cache is always full, so memory always seems to be in use. Hildenbrand suggested using the [pressure-stall information](<https://lwn.net/Articles/759781/>) generated by the kernel; Weiner said that the refault rate could also be used. 

Ryan Roberts suggested increasing the mTHP size for each virtual memory area (VMA) over time as usage indicates, but Weiner said that approach assumes that the process will be running for a long time. Hildenbrand again suggested some sort of BPF interface, but Pache repeated that the hope is to make THP use as transparent as possible. John Hubbard said that hinting interfaces can be useful, since user space knows more about what it is doing than the kernel does; BPF might be overkill, though. Weiner said that applications often know less about themselves than one might assume; if nothing else, they often incorporate libraries that may do surprising things. It can be good to make hinting possible, he said, but there still needs to be a solution that works out of the box. 

The conversation became increasingly unfocused as time ran down. Hildenbrand suggested that the best solution would be to fix the hardware to work better with small pages. Another audience member said that, with current memory-price trends, nobody will be able to afford a 2MB huge page next year anyway. A final suggestion was to just use the `always` mode, coupled with a smarter shrinker. 

#### Better splitting of underutilized huge pages

Later that afternoon, Hildenbrand started a separate session by thanking the organizers (with a smile) for putting this session at the very end of the schedule, when everybody was exhausted. It was, he said, a good topic to finish with, since he didn't have a lot of answers; developers could ponder on it as they headed home. Tired or not, the participants managed to hold a lively discussion on how the kernel might do a better job of splitting apart huge pages when it turns out that they are not being fully used. 

Folio splitting, he said, can be problematic. If a process unmaps some of the base pages from one of its THPs, the kernel may want to split that THP apart, but not necessarily right away. Splitting may not be possible in situations where the needed lock is not available, but splitting may also be undesirable from a performance standpoint. So the kernel defers the splitting by adding the THP to the "deferred split" list for further handling. 

Meanwhile, the kernel wants to find other huge pages that might need to be split, even if they are fully mapped. To that end, when a PMD-level (2MB on x86) THP is allocated, it is automatically added to the split list. Eventually the underutilized shrinker will come along, scan the THP for zero-filled pages, and possibly split it if the page appears to not be fully used. 

There are some problems with the current implementation, he said, some of which are more theoretical than others. One comes about when partially unmapping a THP that is split across VMAs; that will cause the THP to be added to the deferred-split list. Another problem comes about when partially mapped THPs are not detected as such. If a process unmaps an entire folio, frees part of it, then remaps the folio, the partial mapping will not be detected. 

Currently, all PMD-level THPs are added to the deferred-split list, as described above. That is not a huge problem, he said, since there aren't many of those pages in the system. In a world where mTHPs are more heavily used, though, adding them to the list could make the list far too long. The fact that there is no priority associated with placement on that list does not help; the kernel makes no distinction between THP sizes, and no distinction between underutilized and partially mapped THPs, when considering a split. There are no LRU semantics either. 

All of this, he said, drives a feeling that things should be done differently. He had an idea toward that goal, though it was ""a bad one"". All THPs would be added to a list when they are created — and kept there. The result would be a long list, but one that is not often updated. Occasionally, the kernel would scan this list. When it finds a partially mapped page, it will attempt to split it. The utilization scan (looking for zero-filled pages) would be done and, if the page looks like it is not fully used, the kernel would again try to split it. If the page _might_ be partially unmapped, a reverse-mapping scan would be done to see if, once again, it should be split. Otherwise the page would be skipped over. 

That scheme, he said, would result in the underutilized shrinker having to process a lot more huge pages. Rather than maintain a separate list, perhaps the existing LRU lists for anonymous pages could be used. Weiner said that using the anonymous LRU could end up creating a lot more I/O before finding a page that could be split; in the past, he has been unable to find a way to integrate the two lists without causing performance regressions. 

Hildenbrand answered that the shrinker would not be handling reclaim, just the huge-page maintenance. Wilcox said that integration with the LRU would cause the shrinker to scan the oldest folios in the system — the ones that are about to be swapped out anyway. Perhaps, he said, the solution is just to split THPs when they are swapped out and get rid of the shrinker entirely. Scanning for zero-filled pages at swap-out time could help to reduce I/O rates as well. Hildenbrand said that could be problematic for systems that do not have swapping enabled. 

As the session (and the conference) wound down, Hildenbrand asked if there were any better ideas to be had. The resulting silence, he said (again with a smile), should be interpreted as the group having no objections to his proposal. 

Hildenbrand has [posted the slides](<https://share.daveh.de/s/bQsDpD7xigJ6wD8/download?path=&files=LSF_MM%2726_%20Deferred_folio_splitting.pdf>) from this session.
