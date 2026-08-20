---
title: Contiguous memory allocation for drivers
url: https://lwn.net/Articles/396702/
date: "July 21, 2010"
category: "Contiguous memory allocator; Device drivers-Support APIs; Memory management-Large allocations"
author: "By Jonathan Corbet July 21, 2010"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
July 21, 2010

Allocation of physically-contiguous memory buffers is required by many device drivers, but it cannot always be reliably done on long-running Linux systems. That leads to all kinds of unsatisfying workarounds in driver code. The [contiguous memory allocator patches](<http://lwn.net/Articles/396657/>) recently posted by Michal Nazarewicz are an attempt to solve this problem in a consistent way for all drivers. 

A few years ago, when your editor was writing the camera driver for the original OLPC XO system, a problem turned up. The video acquisition hardware on the system was capable of copying video frames into memory via DMA operations, but only to physically contiguous buffers. There was, in other words, no scatter/gather DMA capability built into this (cheap) DMA engine. A choice was thus forced: either allocate memory for video acquisition at boot time, or attempt to allocate it on the fly when the camera is actually used. The former choice is reliable, but it has the disadvantage of leaving a significant chunk of memory idle (on a memory-constrained system) whenever the camera is not in use - most of the time on most systems. The latter choice does not waste memory, but is unreliable - large, contiguous allocations are increasingly hard to do as memory gets fragmented. In the OLPC case, the decision was to sacrifice the memory to ensure that the camera would always work. 

This particular problem has been faced many times by many developers over the years; each driver author has tended to go with whatever _ad hoc_ solution seems to make sense at the time. For some years, the ["bigphysarea" patch](<http://lwn.net/Articles/111132/>) was available to help, but that patch was never put into the mainline and has not seen any maintenance for some time. So the problem remains unsolved in any sort of general sense. 

The contiguous memory allocation (CMA) patches are an attempt to put together a flexible solution which can be used in all drivers. The basic technique will be familiar: CMA grabs a chunk of contiguous physical memory at boot time (when it's plentiful), then doles it out to drivers in response to allocation requests. Where it differs is mainly in an elaborate mechanism for defining the memory region(s) to reserve and the policies for handing them out. 

A system using CMA will always need to have at least one boot-time parameter describing the memory region(s) to use and the policy for allocating from those regions. The syntax used is rather complex, to the point that a large portion of the patch is made up of parsing code; see the included [Documentation/cma.txt file](<https://lwn.net/Articles/396707/>) for the full details. A simple example of a CMA command-line option would be something like: 

```
cma=c=10M cma_map=camera=c
```

This defines a 10MB region (called "c") and states that allocation requests from the `camera` device should be satisfied from this region. Multiple regions can be defined, each with its own size, alignment constraints, and allocation algorithm, and memory regions can be split into different "kinds" as well. The "kinds" feature might be used to separate large and small allocations, or to put different buffers into different DMA zones or NUMA nodes. The more complex command lines are reminiscent of regular expressions, but with less readability. The purpose behind this complexity is to enable a great deal of flexibility in how memory is handled without the need to change the drivers which are working with that memory. Whether that flexibility is worth the cost is not (to your editor, at least) entirely clear. 

A driver can actually allocate a memory chunk with: 

```
#include <linux/cma.h>
    
        unsigned long cma_alloc(const struct device *dev, const char *kind,
    	     		    unsigned long size, unsigned long alignment);
```

If all goes well, the return value will be the physical address of the allocated memory region. 

For reasons which are not entirely clear, buffers allocated with CMA have a reference count associated with them. So two functions are provided to manipulate that count: 

```
int cma_get(unsigned long addr);
        int cma_put(unsigned long addr);
```

Since reference counting is used, there is no `cma_free()` function; instead, the memory chunk is passed to `cma_put()` and freed internally when the reference count goes to zero. 

CMA comes with a best-fit allocator, but it is designed to work with multiple internal allocators. So, should there be a need to use a different allocation algorithm, it's a really straightforward matter to add it to the system. Naturally enough, the command-line syntax offers a way to specify which allocator should be used for each region. 

In summary: CMA offers a solution to a problem which driver authors have been dealing with for some years. Your editor suspects, though, that it will require some changes before a mainline merge can be contemplated. The complexity of the solution is probably more than is really called for in this situation, and the whole thing might benefit from some integration with the DMA mapping infrastructure. But, someday, it would be nice to incorporate a solution to the large-buffer problem that all drivers can use.
