---
title: "Memory power management, take 2"
url: https://lwn.net/Articles/479886/
date: "February 8, 2012"
category: "Memory management-Power management; Partial array self refresh PASR; Power management"
author: "By Jonathan Corbet February 8, 2012"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
February 8, 2012

Last June, LWN [looked at a set of memory power management patches](<https://lwn.net/Articles/446493/>) intended to allow the system to force unused banks of memory into partial-array self-refresh (PASR) mode. PASR makes the memory unavailable to the CPU, but it does ~~preserve those contents and~~ reduce power consumption. Last year's patch added a new layer of zones to the page allocator with the idea that specific zones \- which corresponded to the regions of memory with independent PASR control \- could be vacated and powered down when memory use was light. That patch set has not since returned, possibly because developers were worried about the (significant) overhead of adding another layer of zones to the system. 

For a little while since, things have been quiet on the memory power management front. Recently, though, a new and seemingly unrelated [PASR patch set](<https://lwn.net/Articles/478049/>) was posted to linux-kernel by Maxime Coquelin. This version adds no new zones; instead, it works at a lower level beneath the buddy allocator. 

The first step is to boot the kernel with the new `ddr_die=` parameter, describing the physical layout of the system's memory. Another parameter (`interleaved`) must be used if physically-interleaved memory is present on the system. It would, of course, be nice to obtain this information directly from the hardware, but, in the embedded world where Maxime works, such mechanisms, if they are present at all, must be implemented on a per-subarchitecture or per-board basis. The final patch in the series does add built-in support for the Ux500 system in a "board support" file, but that is the only system supported without boot-time parameters at this early stage. 

For each region defined at boot time, the PASR code sets up a `pasr_section` structure: 

```
struct pasr_section {
    	phys_addr_t start;
    	struct pasr_section *pair;
    	unsigned long free_size;
    	spinlock_t *lock;
    	struct pasr_die *die;
        };
```

The key value here is `free_size`, which tracks how many free pages exist within this section. When the kernel allocates a page for use, it must tell the PASR code about it with a call to: 

```
void pasr_kget(struct page *page, int order);
```

Pages that are freed should be marked with: 

```
void pasr_kput(struct page *page, int order);
```

To a first approximation, these functions just increment and decrement `free_size`. If `free_size` reaches the size of the segment, there are no used pages within that segment and it can be powered down. As soon as the first page is allocated from such a segment, it must be powered back up. 

Adding this accounting to the memory management code is just a matter of adding a few `pasr_kget()` and `pasr_kput()` calls to the buddy allocator. Most other allocations in the kernel have their ultimate source in the buddy allocator, so this approach will catch most of the memory allocation traffic in the system - though it could be somewhat fooled by unused pages held by the slab allocator. There is no integration with "carveout-style" allocators like ION or CMA, but that could certainly be added at some point. 

One piece that is missing, though, is the mechanism by which a memory section becomes entirely free and eligible for PASR. The kernel tends to spread pages of data throughout memory, and it does not drop them without a specific reason to do so; a typical system shows almost no "free" pages at all even if it is not currently doing anything. The [intent](<https://lwn.net/Articles/479889/>) is to use the feature in conjunction with a "page cache flush governor," but that code does not exist at this time. There was also talk of setting up a large "movable" zone and using the compaction code to create large, free chunks within that zone. 

The other thing that is missing at this point is any kind of measurement of how much power is actually saved using PASR. That will certainly need to be provided before this code can be considered for inclusion. Meanwhile, it has the appearance of a less-intrusive PASR capability that might just get past the roadblocks that stopped its predecessor.
