---
title: Further progress toward removing the page map count
url: https://lwn.net/Articles/1073418/
date: "May 27, 2026"
category: "Memory management-Folios"
author: "By Jonathan Corbet May 27, 2026 LSFMM+BPF"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
May 27, 2026

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2026/>)

David Hildenbrand has been [working for some time](<https://lwn.net/Articles/1013649/>) to get rid of the `mapcount` field of `struct page`. At the 2026 [Linux Storage, Filesystem, Memory Management, and BPF Summit](<https://events.linuxfoundation.org/lsfmmbpf/>), he was clearly feeling like he was getting close to that goal; he described some plans and future challenges in a memory-management-track session. 

The `mapcount` field was created to track the number of mappings (page-table entries) that refer to the given page. Among other things, a `mapcount` of zero means that the page has no references and can be reclaimed. Maintaining `mapcount` has become increasingly challenging and expensive as the memory-management system has grown in complexity, so Hildenbrand has been looking for ways to get rid of it. This session was, he said, maybe one of the last times he will have to bring up this topic. 

Since the 6.15 release, the kernel has had the `NO_PAGE_MAPCOUNT` configuration option, which enables the code that is being developed to eliminate the use of the `mapcount` field. It is marked experimental because, along with the fact that it is indeed experimental code, it makes some of the accounting data less precise; whether that will create problems for user space is not yet clear. Since that option was added, progress has been made on a number of fronts, and no problems have been reported. But, since the option is marked experimental, he said, nobody is testing it, so the lack of problem reports is only so comforting. 

Next, he plans to properly support [folios](<https://lwn.net/Articles/1064861/>) larger than the PMD size (2MB on x86). The code actually handles PUD-size folios now as a sort of special case, but no sizes between the two are supported. He also wants to support mapping a large folio as an arbitrary collection of page sizes; a 1GB PUD-size huge page might be mapped as a combination of PMD-size and 4KB base pages, for example. Once that is working properly, it will be time to remove the experimental marker from the configuration option. 

He [sent a patch set](<https://lwn.net/ml/all/20260412-mapcount-v1-0-05e8dfab52e0@kernel.org/>) in April with the arbitrary-size support and removal of `mapcount`. That work adds a new field, `_total_pages_mapped`, to the `folio` structure that counts all base pages once for each time they are mapped; a PMD-level mapping would increase this count by 512, for example. Accounting for mappings in this way makes some statistics imprecise, he said, but he doesn't know if anybody cares about it. 

If a folio has even one base page mapped, that folio is counted as fully mapped in this new field; mapping a single base page out of a PMD-size folio will, once again, increase `_total_pages_mapped` by 512\. This accounting does not change how the resident-set size is calculated, though. How the new code is able to answer questions about folios does change a bit. Some questions, such as whether a folio has any pages mapped, the total number of mappings it has, whether it has unexpected references, and whether it is mapped shared or exclusive are easily answered with the new count. On the other hand, the kernel cannot give a definitive answer to whether an anonymous folio is partially mapped. 

One place where this could be a problem, he said, is in the [`/proc/_PID_ /pagemap` file](<https://docs.kernel.org/admin-guide/mm/pagemap.html>), which has a field indicating how many processes a given page is mapped into. With Hildenbrand's changes, this field might mark an exclusively mapped page as being shared in some situations. It was, he said, a mistake to have ever exported that field. The proportional-share fields (`Pss` and `Pss_Dirty`) in [`/proc/_PID_ /smaps`](<https://docs.kernel.org/filesystems/proc.html#:~:text=%2Fproc%2FPID%2Fsmaps>) become less precise, as do [`/proc/_PID_ /numa_maps`](<https://docs.kernel.org/filesystems/proc.html#:~:text=The%20%2Fproc%2Fpid%2Fnuma%5Fmaps>) and [`/proc/kpagecount`](<https://docs.kernel.org/admin-guide/mm/pagemap.html#:~:text=%2Fproc%2Fkpagecount>). He does not think that anybody will care about these cases, which only come about for partially mapped folios. An audience member asserted that the `Pss` value was broken in any case, since processes are able to influence it. 

Hildenbrand concluded by saying that he would like to make `NO_PAGE_MAPCOUNT` option the default in the near future. Kiryl Shutsemau suggested to just try it, perhaps with a longer-than-usual trial period in linux-next, to see what breaks.
