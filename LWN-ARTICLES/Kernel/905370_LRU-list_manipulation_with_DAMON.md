---
title: "LRU-list manipulation with DAMON"
url: https://lwn.net/Articles/905370/
date: "August 22, 2022"
category: "Memory management-DAMON; Releases-6.0"
author: "By Jonathan Corbet August 22, 2022"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
August 22, 2022

The [DAMON](<https://www.kernel.org/doc/html/latest/admin-guide/mm/damon/>) subsystem, which entered the kernel during the 5.15 release cycle, uses various heuristics to determine which pages of memory are in active use. Since the beginning, the intent has been to use this information to influence memory management. The 6.0 kernel contains another step in this direction, giving DAMON the ability to actively reorder pages on the kernel's least-recently-used (LRU) lists. 

The kernel's memory-management developers would like nothing better than the ability to know which pages of memory will be needed in the near future; the kernel could then make sure that those pages were resident in RAM. Unfortunately, current hardware is unable to provide that information, so the memory-management code must make guesses instead. Usually, the best guess is that pages that have been used in the recent past are likely to be used again soon, while those that have gone untouched for some time are probably not needed. 

#### LRU lists

This approach works well, but there is still a problem: there are limits to how closely the kernel can track the usage of each page. Making a note of every access would slow the system to a crawl, so something else must be done. The LRU lists are one way in which the memory-management subsystem tries to answer this question in an efficient way. 

The less-simplified view is that there is a pair of LRU lists for each memory zone, and for each memory control group as well. The core idea remains the same, though. 

In a simplified view, there are two LRU lists in the system, called the "active" and "inactive" lists. Pages in normal use are put onto the head of the active list. Over time, they will move down the list as other pages are added, though some events can promote a given page back to the head. Pages that reach the tail of the active list are relatively likely to have gone unused in the recent past, but the association is approximate. 

Occasionally, the kernel will pull a set of pages off the tail of the active list and place them, instead, at the head of the inactive list. When this happens, the pages are "deactivated", meaning that they are marked in the page tables as "not present". Should some process try to access such a page, a soft page fault will result; the kernel will then observe that the page is still in use and move it back to the active list. Pages that remain on the inactive list, instead, will find their way to the tail, where they will be reclaimed when the kernel needs memory for other uses. 

The LRU lists are, thus, a key part of the mechanism that decides which pages stay in RAM and which are reclaimed. Despite their name, though, these lists are at best a rough approximation of which pages have been least (or most) recently used. The description might be better given as "least recently noticed to be used" instead. If there were a better mechanism for understanding which pages are truly in heavy use, it should be possible to use that information to improve on the current LRU lists. 

#### Reordering the lists

DAMON ("Data Access MONitor") is meant to be that mechanism. Through some clever algorithms (described in [this article](<https://lwn.net/Articles/812707/>)), DAMON tries to create a clearer picture of actual memory usage while, at the same time, limiting its own CPU usage. DAMON is designed to be efficient enough to use on production systems while being accurate enough to improve memory-management decisions. 

The 5.16 kernel saw the addition of DAMOS ("DAMON operation schemes"), which adds a rule-based mechanism that can cause actions to be taken whenever specific criteria are met. For example, DAMOS could be configured to pass a region that has not been accessed in the last _N_ seconds to the equivalent of an `[madvise(MADV_COLD)](<https://man7.org/linux/man-pages/man2/madvise.2.html>)` call. Various other options are available; they are all described in detail in [`Documentation/admin-guide/mm/damon/usage.rst`](<https://www.kernel.org/doc/html/latest/admin-guide/mm/damon/usage.html>). 

The work merged for 6.0 adds two new operations to DAMOS: `lru_prio` and `lru_deprio`. The first will cause the indicated pages to be moved to the head of the active list, making them the last pages that the kernel will try to deactivate or reclaim; the second, instead, will deactivate the given pages, causing them to be moved to the inactive lists. With this change, in other words, DAMOS is reaching deep into the memory-management subsystem, using its (hopefully) superior information to make the ordering of the LRU lists closer to actual usage. This sorting could be especially useful if the system comes under sudden memory pressure and has to start reclaiming memory quickly. 

Author SeongJae Park calls this mechanism "proactive LRU-list sorting" or PLRUS. When properly tuned, he claimed in [the patch series cover letter](<https://lwn.net/ml/linux-kernel/20220613192301.8817-1-sj@kernel.org/>), this mechanism can yield some nice results: ""In short, PLRUS achieves 10% memory PSI (some) reduction, 14% major page faults reduction, and 3.74% speedup under memory pressure"". The term "PSI (some)" here refers to the [pressure-stall information](<https://lwn.net/Articles/759781/>) produced by the kernel, which is a measure of how much processes are being delayed waiting for memory. 

The "when properly tuned" caveat is important, though; DAMOS has a complex set of parameters to describe action thresholds and to limit how much CPU time is used by DAMOS itself. Adjusting those parameters can result in significant changes to how the core memory-management subsystem goes about its work. DAMOS offers a lot of flexibility to a full-time administrator who understands how memory management works and who is able to accurately measure the effects of changes. It also makes it easy to completely wreck a system's performance. 

To aid administrators who do not have the time or skills to come up with an optimal DAMOS tuning for their workload, Park also added a new kernel module called `damon_lru_sort`. It uses DAMOS to perform proactive LRU-list sorting under a set of ""conservative"" parameters that are meant to safely improve performance while minimizing overhead. This module will make using the LRU-list sorting feature easier, but it still has a significant set of tuning knobs; [the documentation](<https://www.kernel.org/doc/html/latest/admin-guide/mm/damon/lru_sort.html>) describes them all. 

This mechanism is aimed at a similar problem to that addressed by the [multi-generational LRU](<https://lwn.net/Articles/894859/>) work, which currently seems on track to be merged in 6.1. The multi-generational LRU, too, tries to create a more accurate picture of which pages are in active use so that better page-replacement decisions can be made. There are a number of open questions about how the movement of pages between the generations should be handled; there is talk of allowing the loading of BPF programs to control those decisions, but DAMOS might be able to help as well. The integration between the two mechanisms does not currently exist, but could be a good thing to add. 

The advent of this type of ability to tweak memory management is, obviously, a sign that better performance is always desirable. It is also, perhaps, an indication that creating a memory-management subsystem that performs optimally for all workloads is beyond our current capabilities. Kernel developers tend to prefer not to add new configuration knobs on the theory that the kernel should be able to configure itself. Here, though, new knobs are being added in large numbers. Some problems are, it seems, still too hard for the kernel to solve without help.
