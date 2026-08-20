---
title: Facing down mapcount madness
url: https://lwn.net/Articles/974223/
date: "May 21, 2024"
category: "Memory management-struct page"
author: "By Jonathan Corbet May 21, 2024 LSFMM+BPF"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
May 21, 2024

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2024/>)

The [`page` structure](<https://elixir.bootlin.com/linux/v6.9.1/source/include/linux/mm_types.h#L35>) is a complicated beast, but some parts of it are more intimidating than others. The `mapcount` field is one of the scarier parts. It allegedly records the number of references to the page in page tables, but, as David Hildenbrand described during the memory-management track at the [2024 Linux Storage, Filesystem, Memory Management, and BPF Summit](<https://events.linuxfoundation.org/lsfmmbpf/>), things are more complicated than that. Few people truly understand the semantics of this field, but the situation will hopefully get better over time. 

There are a number of problems surrounding the page mapping count, starting with the fact that a page-table mapping is only one way to create a reference to a page. Reference-tracking confusion has [led to severe bugs](<https://lwn.net/Articles/895439/>) in the past. The adoption of folios has, in the short term at least, made things worse in some ways (while improving it in others), since mappings can happen at both the folio and page levels. Determining if a folio is mapped can require iterating over the mapping counts of all the pages it contains, which gets slower as folios get larger. All of this leads to a desire to clarify the use of mapping counts, and to eliminate the use of page-level mapping counts whenever possible. 

Hildenbrand started by referring back to "simpler times" when the kernel maintained a simple, 31-bit map count for each page. If that count was zero, then the page was not mapped into user space; a count of one indicated that there was a single user, while anything larger meant that the page was shared. But then the kernel added transparent huge pages, and life got more complicated. It was a natural evolution that led to flags like [`PG_double_map`](<https://elixir.bootlin.com/linux/v6.0.19/source/include/linux/page-flags.h#L878>), which indicated that the page was mapped at both the page-table (PTE) and page-middle-directory (PMD) levels — that it was mapped as both a base page and a huge page, in other words. There followed a whole series of functions for handling the mapping count with names like [`page_trans_huge_map_swapcount()`](<https://elixir.bootlin.com/linux/v5.0.21/source/mm/swapfile.c#L1484>). Increasingly, nobody really understood what `mapcount` really meant. 

[![\[David
Hildenbrand\]](https://static.lwn.net/images/conf/2024/lsfmm/DavidHildenbrand-sm.png)](<https://lwn.net/Articles/974228/>) That said, things have improved; the folio work has helped to straighten a lot of things out. The semantics of `mapcount` are "almost clear" now, he said. A count of zero means that a folio is not mapped; if it is greater than zero, then mappings exist. A count of one indicates an exclusive mapping; a count greater than one says that the folio _might_ be mapped shared. There is a function, `folio_likely_mapped_shared()`, in linux-next that makes an "educated guess" as to whether a given folio is shared. 

Part of the objective here is to stop keeping track of mappings at the page level; doing so requires fixing code that is using that information. The [`page_mapped()`](<https://elixir.bootlin.com/linux/v6.9.1/source/include/linux/mm.h#L1285>) function is easy to remove, and [`total_mapcount()`](<https://elixir.bootlin.com/linux/v6.8.10/source/include/linux/mm.h#L1243>) went away in 6.9. [`page_mapcount()`](<https://elixir.bootlin.com/linux/v6.9.1/source/include/linux/mm.h#L1218>) is harder, since there is no direct translation to a folio function. Instead, most of the users of `page_mapcount()` have been removed; the last few call sites (including those in [KSM](<https://lwn.net/ml/linux-mm/20240416172533.663418-1-david@redhat.com/>) and [khugepaged](<https://lwn.net/ml/linux-mm/20240424122630.495788-1-david@redhat.com/>)) are going to be challenging to fix, though. 

There are some other problems yet to be solved as well, he said. Large folios that are smaller than the PMD size cannot have PMD mappings, so they only have PTE-level mappings. That means maintaining the per-page map counts in each page, which is expensive; the atomic operations on each page add up. Some other planned optimizations may make maintaining those per-page counts impossible. As a result, the kernel may not be able to tell if a folio is mapped shared; it is also not possible to handle folios that are larger than the PMD size. That latter problem could perhaps be addressed by adding a map count for each PMD entry covered by a folio, and perhaps extending that solution to higher levels of the page-table hierarchy as well. That is not a pleasing solution and should be avoided if possible, but it can be a backup if nobody comes up with anything better. 

[![\[Mapcount table\]](https://static.lwn.net/images/conf/2024/lsfmm/mapcount-table-sm.jpg)](<https://lwn.net/Articles/974230/>) Hildenbrand put up a slide showing the various use cases for the mapping count, both in the present and the future. All is good for small folios, he said, but it is hard to keep track of whether large folios are shared in current kernels. That situation is somewhat improved in the mm-stable tree (which may have moved into the mainline for 6.10 by the time you read this), but there is still work to be done. 

One place where the shared status of a folio is important is in memory-use accounting. There are three different sizes used to describe a process's memory use. The resident-set size (RSS) is the number of pages that a process has resident in memory at any given time. The unique set size (USS) only counts pages that are unique to the process, not counting the shared pages. The proportional set size (PSS) is calculated by dividing the number of shared pages by the number of processes sharing them. If a process maps 100 pages shared with three others, its PSS will increase by 25\. 

If a process maps a single page from a 16-page folio, all three set sizes will grow by one page — 4KB. That is wrong, Hildenbrand said, since the full 16 pages are all in memory; the increase should be 64KB. But there is no way to get that result in the kernel currently. On the other hand, the current model works correctly if a folio is split. 

Calculating these set sizes requires `page_mapcount()` to determine if a page is shared and, if so, how widely it is shared. In the absence of a per-page map count, some other solution will have to be found. One possibility is to just use the folio mapping count, and to keep a count of mapped pages at the PMD level. For most other uses, including the USS calculation, all that is really needed is to know whether a folio is mapped exclusively or not. 

Upcoming changes will cause the kernel to lose its ability to track the number of pages mapped within a folio; that will result in charging a user for an entire folio if any page is mapped. It might also cause USS to be too small if a folio is mapped with a combination of exclusive and mapped pages, and PSS may lose precision. It is not clear that this will be a big problem; there will be a debugging option to allow developers to get a better handle on the situation. 

One potential problem for the future is an overflow of the page reference count, which includes the map count but also any other types of reference that a page might have. Overflow is not seen as a problem for small folios now; Matthew Wilcox pointed out that it would require a system with terabytes of installed memory to even get close. But large folios, with more pages (and thus more reference counts to add up) are a different story, especially on 32-bit systems. Michal Hocko suggested just making the reference count a 64-bit quantity and seeing if anybody complains. Hildenbrand said that the kernel could also simply avoid incrementing the reference count if the mapping count is greater than zero; that would save some atomic operations as well. 

By this point, time had run out. As the session closed, it was pointed out that some drivers use the `mapcount` field for their own purposes on pages that are not otherwise mapped. Wilcox suggested that such uses need to be "excised from the kernel".
