---
title: Unmapped page cache control
url: https://lwn.net/Articles/419713/
date: "December 13, 2010"
category: "Memory management-Virtualization; Virtualization"
author: "By Jonathan Corbet December 13, 2010"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
December 13, 2010

Virtualization places some interesting demands on the host system, many of which are related to memory management. When two agents within the system both believe that they are in charge of memory, interesting conflicts are bound to arise. A recent patch from Balbir Singh shows the kind of effort which is being made to address these conflicts, but it also gives a hint at how a more ambitious effort might really solve the problem. 

The Linux page cache keeps copies of pages from on-disk files in main memory in the hopes of avoiding I/O operations when those pages are accessed. At any given time, the page cache can easily account for more than half of the system's total memory usage. The actual size of the page cache varies over time; as other types of memory use (kernel memory and anonymous pages) grow, the page cache shrinks to make room. Balancing the demands of the page cache with other memory users can be a challenge, but Linux seems to get it close to right most of the time. 

Balbir's [patch](<https://lwn.net/Articles/419564/>) is intended to give the system administrator a bit more control over page cache usage; to that end, it provides a new boot-time parameter (`unmapped_page_control`) which sets an upper bound on the number of unmapped pages in the cache. "Unmapped" pages are those which are not mapped into any process's address space - they do not appear in any page tables. Unmapped pages arguably have a lower chance of being needed in the near future; they are also a bit easier for the system to get rid of. This patch thus gives the system administrator a relatively easy way to minimize page cache memory usage. 

The obvious question is: why? Page cache pages will be reclaimed anyway if the system has other needs for the memory, so there would seem to be little point in shrinking it prematurely. The problem, it seems, is virtualization. When a process on a virtualized system reads a page from a file, the guest operating system will dutifully store a copy of that page in its page cache. The actual read operation, though, will be passed through to (and executed by) the host, which will also store a copy in its page cache. So the page gets cached twice - perhaps even more times if it is used by multiple virtual machines. Caching a page can be a good thing, but caching multiple copies is likely to be too much of a good thing. 

So what Balbir's patch is doing can be explained this way: it is forcibly cleaning copies of pages out of guest page caches to minimize duplicate copies. The memory freed in this way can be captured by a balloon driver and returned to the host, making it available for more productive use elsewhere in the system. 

This technique should clearly improve the situation. Less duplication is good, and, if the guest ends up needing some of the freed pages, those pages stand a good chance of being found in the host's page cache. But one can't help but wonder if it might not be an overly indirect approach. Rather than forcibly reclaim pages from the guests' caches, might it be better to have all of the systems share the same page cache? A single, unified page cache could be managed to maximize the performance of the system as a whole; that should yield better results than managing a number of seemingly independent page caches. 

Virtualization based on containers has exactly this type of unified page cache since all of the containers are running on the same kernel. That may be one of the reasons why containers are seen to perform better than fully virtualized systems. Bringing a shared page cache to the virtualized world would be a bit of a challenge, though, which probably has a lot to do with why it has not already been done. 

To begin with, there would be some clear security issues. A virtualized system should be unable to access any resources which have not been explicitly provided to it. Any sort of shared page cache would have to be designed in a way which would leave the host in control of which pages are visible to each guest. In practice, that would probably mean using the virtualized block drivers which make filesystems available to virtualized guests now. Rather than "read" a page into a page controlled by the guest, the driver could somehow just map the host's copy of the page into the guest's address space. 

Making that work correctly would require the addition of a significant new, Linux-only API between the host and the guest. It would be hard to do it in a way which maintained any sort of illusion that the guest is running on hardware of its own. Such a scheme would complicate memory management in the guest - hardware is increasingly dynamic, but individual pages of memory still do not come and go spontaneously. A shared page cache would also frustrate attempts to use huge pages for guest memory. 

In other words, the difficulties of sharing the page cache between hosts and guests look to be decidedly nontrivial. It is not surprising that we are still living in a world where scarce memory pages can be soaked up with duplicate copies of data. As long as that situation holds, there will be a place for patches which cause guests to behave in ways which are more friendly to the system as a whole.
