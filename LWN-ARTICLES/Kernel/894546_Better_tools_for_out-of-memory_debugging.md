---
title: "Better tools for out-of-memory debugging"
url: https://lwn.net/Articles/894546/
date: "May 11, 2022"
category: "Memory management-Out-of-memory handling"
author: "By Jonathan Corbet May 11, 2022 LSFMM"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
May 11, 2022

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2022/>)

Out-of-memory (OOM) situations are dreaded by users, system administrators, and kernel developers alike. Usually, all that is known is that a lot of memory is being used somewhere and the system has run out, but the kernel provides little help to anybody trying to figure out where the memory has gone. In a memory-management session at the [2022 Linux Storage, Filesystem, Memory-management and BPF Summit](<https://events.linuxfoundation.org/lsfmm/>) (LSFMM), Kent Overstreet asked what could be done to improve OOM reports and reduce the pain for all involved. 

The kernel writes a report to the system log when an OOM problem occurs, he began, but those reports often do not include information on memory managed by the slab allocator. Other times, there can be hundreds of those reports, which is not necessarily much more helpful. With a new reporting system he has been working on (described briefly in [this article](<https://lwn.net/Articles/892611/>)), the report only includes the ten slabs with the most allocated memory, which often is what he wants to see. Even more important for debugging OOM problems is information on the kernel's shrinkers, which are responsible for reclaiming memory from caches when it is needed elsewhere. [![\[Kent Overstreet\]](https://static.lwn.net/images/conf/2022/lsfmm/KentOverstreet-sm.png)](<https://lwn.net/Articles/894551/>) There is currently no information available on what the shrinkers are doing, but Overstreet's new report can include what was requested of each shrinker and how much the shrinker was actually able to free. 

This information is useful, he said, but it's only a start. The OOM-report code hasn't changed much since 2006, and it is showing its age; there is a lot of room for improvement. Johannes Weiner, who has done much of the work on OOM reporting, added that even the 2006 work was "mostly a refactoring". It was generally agreed that developers need better tools to address these out-of-memory situations. 

A part of the problem, Overstreet said, is that outputting information from the kernel in a human-readable way is not easy. There are various "pretty printers" available, mostly in the form of special [`printk()` format specifiers](<https://docs.kernel.org/core-api/printk-formats.html>). But these are all hidden away in the `printk()` code and are hard to find; pretty printers are better placed with the code that manages the data they are printing. With the right infrastructure, he said, the kernel can have "thousands of pretty printers" and its output will get much better. 

With regard to improving the OOM reports specifically, he suggested adding rate limiting as a first step. Once, say, a dozen reports have been printed, there is not likely to be much value in creating more. Reorganizing the reports to separate information about kernel-space and user-space memory would help. Information on fragmentation in the page allocator is needed and, as mentioned above, more information about shrinkers. 

Weiner said that the report used to just print the top memory-consuming tasks rather than all of them, but that got removed for some reason. Michal Hocko responded that dumping out the top tasks is not easy; it requires locking the task list, which is expensive. Beyond that, partial information on the state of the system can be misleading and make it hard to get a complete picture; the top consumers may not be the problem if the sheer number of tasks overwhelms the system. What should be in the report depends on the situation. If, for example, a `GFP_NOWAIT` allocation request is failing, then the shrinkers (which will not be invoked in that situation) are probably not relevant. That is also true for high-order allocations; in that case, compaction is failing and developers need to know why. What's in the report now, he said, is a compromise — the information that is usually useful. 

Overstreet said that the contents of the report will always be a compromise, but it is possible to create better reports than what the kernel has now. Hocko said that he has to process a lot of OOM reports, and his feeling is that there are simply too many numbers in them. What he often wants to do is to check the proportion of memory used by the slab allocator relative to that on the least-recently-used (LRU) lists. If the LRU memory dominates, then the problem is almost certainly in user space; calling out that situation explicitly would help. 

Weiner suggested starting with the most useful summary information; the rest can come afterward. Verbosity in these reports is not necessarily a problem, especially if they are rate-limited. Hocko countered that, while rate limiting is nice in theory, it doesn't actually work. The problem is that `printk()` can be slow, especially when serial consoles are being used; just dumping all of the information in a report can bog down the machine, and it takes too long to trigger the limit. 

Ira Weiny asked how it is that memory can be allocated without the kernel knowing where it went. The problem is that the tracking infrastructure just isn't there. Overstreet said that he has a mechanism to track memory usage by call site that is efficient enough to use on production systems; it employs the [dynamic debugging](<https://lwn.net/Articles/434833/>) mechanism. The `pr_debug()` macro is changed to create a static structure at the call site that is placed in its own ELF section. This mechanism can then be used to wrap `kmalloc()` calls and remember where they came from. 

Hocko asked why the existing tracing mechanism couldn't be used for this purpose; Overstreet answered that he wants something that is always on, so he can look at the allocation numbers at any time. Paul McKenney suggested using a BPF program to store call-site information in a map. Weiner answered, though, that he had tried that once and it was trickier than it seems. There are cases, such as freeing memory in an interrupt handler, that are hard to handle. 

The session concluded without much in the way of firm conclusions. Overstreet closed by saying that he keeps "finding stupid stuff" in the kernel, and that developers are not looking at memory allocations the way they should be. With luck, better tools will improve that situation in the future.
