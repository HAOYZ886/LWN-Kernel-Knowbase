---
title: The big chunk memory allocator
url: https://lwn.net/Articles/417037/
date: "November 24, 2010"
category: "Memory management-Large allocations"
author: "By Jonathan Corbet November 24, 2010"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
November 24, 2010

Device drivers - especially those dealing with low-end hardware - sometimes need to allocate large, physically-contiguous memory buffers. As the system runs and memory fragments, those allocations are increasingly likely to fail. That had led to a lot of schemes based around techniques like setting aside memory at boot time; the [contiguous memory allocator (CMA)](<https://lwn.net/Articles/396702/>) patch set covered here in July is one example. There is an alternative approach out there, though, in the form of Hiroyuki Kamezawa's [big chunk memory allocator](<https://lwn.net/Articles/416284/>). 

The big chunk allocator provides a new allocation function for large contiguous chunks: 

```
struct page *alloc_contig_pages(unsigned long base, unsigned long end,
    				    unsigned long nr_pages, int align_order);
```

Unlike CMA, the big chunk allocator does not rely on setting aside memory at boot time. Instead, it will attempt to organize a suitable chunk of memory at allocation time by moving other pages around. Over time, the memory compaction and page migration mechanisms in the kernel have gotten better and memory sizes have grown. So it is more feasible to think that this kind of large allocation might be more possible than it once was. 

There are some advantages to the big chunk approach. Since it does not require that memory be set aside, there is no impact on the system when there is no need for large buffers. There is also more runtime flexibility and no need for the system administrator to figure out how much memory to reserve at boot time. The down sides will be that memory allocation becomes more expensive and the chances of failure will be higher. Which system will work better in practice is entirely unknown; answering that question will require some significant testing by the people who need the large allocation capability.
