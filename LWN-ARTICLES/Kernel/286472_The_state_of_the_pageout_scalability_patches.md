---
title: The state of the pageout scalability patches
url: https://lwn.net/Articles/286472/
date: "June 17, 2008"
category: "Memory management-Page replacement algorithms; Scalability"
author: "By Jonathan Corbet June 17, 2008"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
June 17, 2008

The virtual memory scalability improvement patch set overseen by Rik van Riel has been under construction for well over a year; LWN [last looked at it](<http://lwn.net/Articles/257541/>) in November, 2007\. Since then, a number of new features have been added and the patch set, as a whole, has gotten closer to the point where it can be considered for mainline inclusion. So another look would appear to be in order. 

One of the core changes in this patch set remains the same: it still separates the least-recently-used (LRU) lists for pages backed up by files and those backed up by swap. When memory gets tight, it is generally preferable to evict page cache pages (those backed up by files) rather than anonymous memory. File-backed pages are less likely to need to be written back to disk and they are more likely to be well laid-out on disk, making it quicker to read them back in if necessary. Current Linux kernels keep both types of pages on the same LRU list, though, forcing the pageout code to scan over (potentially large numbers of) pages which it is not interested in evicting. Rik's patch improves this situation by splitting the LRU list in two, allowing the pageout code to only look at pages which might actually be candidates for eviction. 

There comes a point, though, where anonymous pages need to be reclaimed as well. The kernel will make an effort to pick the best pages to evict by going for those which have not been recently referenced. Doing that, however, requires going through the entire list of anonymous pages, clearing the "referenced" bit on each. A large system can have many millions of anonymous pages; iterating over the entire set can take a long time. And, as it turns out, it's not really necessary. 

The VM scalability patch set now changes that behavior by simply keeping a certain percentage of the system's anonymous pages on the inactive list - the first place the system looks for pages to evict. Those pages will drift toward the front of the list over time, but will be returned to the active list if they are used. Essentially, this patch is applying a form of the "referenced" test to a portion of anonymous memory - whether or not anonymous pages are being evicted at the time - rather than trying to check the referenced state of all anonymous pages when the kernel decides it needs to reclaim some of them. 

Another set of patches addresses a different situation: pages which cannot be evicted at all. These pages might have been locked into memory with a system call like `mlock()`, be part of a locked SYSV shared memory region, or be part of a RAM disk, for example. They can be either page cache or anonymous pages. Either way, there is little point in having the reclaim code scan them, since it will not be possible to evict them. But, of course, the current reclaim code does have to scan over these pages. 

This unneeded scanning, as it turns out, can be a problem. The extensive [unevictable LRU document](<https://lwn.net/Articles/286485/>) included with the patch claims: 

For example, a non-numal x86_64 platform with 128GB of main memory will have over 32 million 4k pages in a single zone. When a large fraction of these pages are not evictable for any reason [see below], vmscan will spend a lot of time scanning the LRU lists looking for the small fraction of pages that are evictable. This can result in a situation where all cpus are spending 100% of their time in vmscan for hours or days on end, with the system completely unresponsive. 

Most of us are not currently working with systems of this size; one must spend a fair amount of money to gain the benefits of this sort of pathological behavior. Still, it seems like something which is worth fixing. 

The solution, of course, is yet another list. When a page is determined to be unevictable, that page will go onto the special, per-zone unevictable list, after which the pageout code will simply not see it anymore. As a result of the variety of ways in which a page can become unevictable, the kernel will not always know at mapping time whether a specific page can go onto the unevictable list or not. So the pageout code must keep an eye out for those pages as it scans for reclaim candidates and shunt them over to the unevictable list as they are found. In relatively short order, the locked-down pages will accumulate in this list, freeing the pageout code to concentrate on pages it can actually do something about. 

Many of the concerns which have been raised about this patch set over the last year have been addressed. A few remain, though. Some of the new features require new page flags; these flags are in extremely short supply, so there is always pressure to find ways of implementing things which do not allocate more of them. There are a few too many configuration options and associated `#ifdef` blocks. And so on. Addressing these may take a while, but convincing everybody that these (rather fundamental) memory management changes are beneficial under all circumstances may take rather longer. So, while this patch set is making progress, a 2.6.27 merge is probably not in the cards.
