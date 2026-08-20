---
title: Page replacement for huge memory systems
url: https://lwn.net/Articles/257541/
date: "November 7, 2007"
category: "Memory management-Page replacement algorithms"
author: "By Jake Edge November 7, 2007"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jake Edge**  
November 7, 2007

As the amount of RAM installed in systems grows, it would seem that memory pressure should reduce, but, much like salaries or hard disk space, usage grows to fill (or overflow) the available capacity. Operating systems have dealt with this problem for decades by using virtual memory and swapping, but techniques that work well with 4 gigabyte address spaces may not scale well to systems with 1 terabyte. That scalability problem is at the root of several different ideas for changing the kernel, from supporting [larger page sizes](<http://lwn.net/Articles/250335/>) to [avoiding memory fragmentation](<http://lwn.net/Articles/224829/>). 

Another approach to scaling up the memory management subsystem was recently posted to linux-kernel by Rik van Riel. His [patch](<http://lwn.net/Articles/257223/>) is meant to reduce the amount of time the kernel spends looking for a memory page to evict when it needs to load a new page. He lists two main deficiencies of the current page replacement algorithm. The first is that it sometimes evicts the wrong page; this cannot be eliminated, but its frequency might be reduced. The second is the heart of what he is trying to accomplish: 

The kernel scans over pages that should not be evicted. On systems with a few GB of RAM, this can result in the VM using an annoying amount of CPU. On systems with >128GB of RAM, this can knock the system out for hours since excess CPU use is compounded with lock contention and other issues. 

A system with 1TB of 4K pages has 256 million pages to deal with. Searching through the pages stored on lists in the kernel can take an enormous amount of time. According to van Riel, most of that time is spent searching pages that won't be evicted anyway, so in order to deal with systems of that size, the search needs to focus in on likely candidates. 

Linux tries to optimize its use of physical memory, by keeping it full, using any memory not needed by processes for caching file data in the page cache. Determining which pages are not being used by processes and striking a balance between the page cache and process memory is the job of the page replacement algorithm. It is that algorithm that van Riel would eventually like to see replaced. 

The current set of patches, though, take a smaller step. In today's kernel, there are two lists of pages, active and inactive, for each memory zone. Pages move between them based on how recently they were used. When it is time to find a page to evict, the kernel searches the inactive list for candidates. In many cases, it is looking for page cache pages, particularly those that are unmodified and can simply be dropped, but has to wade through an enormous number of process-memory pages to find them. 

The solution proposed is to break both lists apart, based on the type of page. Page cache pages (aka file pages) and process-memory pages (aka anonymous pages) will each live on their own active and inactive lists. When the kernel is looking for a specific type, it can choose the proper list to reduce the amount of time spent searching considerably. 

This patch is an update to an earlier proposal by van Riel, [covered here last March](<http://lwn.net/Articles/226756/>). The patch is now broken into ten parts, allowing for easier reviewing. It has also been updated to the latest kernel, modified to work with various features (like [lumpy reclaim](<http://lwn.net/Articles/211505/>)) that have been added in the interim. 

Additional features are planned to be added down the road, as outlined on van Riel's [page replacement design web page](<http://linux-mm.org/PageReplacementDesign>). Adding a non-reclaimable list for pages that are locked to physical memory with `mlock()`, or are part of a RAM filesystem and cannot be evicted, is one of the first changes listed. It makes little sense to scan past these pages. 

Another feature that van Riel lists is to track recently evicted pages so that, if they get loaded again, the system can reduce the likelihood of another eviction. This should help keep pages in the page cache that get accessed somewhat infrequently, but are not completely unused. There are also various ideas about limiting the sizes of the active and inactive lists to put a bound on worst-case scenarios. van Riel's plans also include making better decisions about when to run the out-of-memory (OOM) killer as well as making it faster to choose its victim. 

Overall, it is a big change to how the page replacement code works today, which is why it will be broken up into smaller chunks. By making changes that add incremental improvements, and getting them into the hands of developers and testers, the hope is that the bugs can be shaken out more easily. Before that can happen, though, this set of patches must pass muster with the kernel hackers and be merged. The external user-visible impacts of these particular patches should be small, but they are fairly intrusive, touching a fair amount of code. In addition, memory management patches tend to have a tough path into the kernel.
