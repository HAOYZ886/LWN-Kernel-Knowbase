---
title: The trouble with DMA masks
url: https://lwn.net/Articles/543408/
date: "March 20, 2013"
category: "Device drivers-Support APIs; Direct memory access"
author: "By Jonathan Corbet March 20, 2013"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
March 20, 2013

Almost any I/O device worth its electrons will support direct memory access (DMA) transactions; to do otherwise is to be relegated to the world of low-bandwidth, high-overhead I/O. But "DMA-capable" devices are not all equally so; many of them have limitations restricting the range of memory that can be directly accessed. The 24-bit limitation that afflicted ISA devices in the early days of the personal computer is a classic example, but contemporary hardware also has its limits. The kernel has long had a mechanism for working around these limitations, but it turns out that this subsystem has some interesting problems of its own. 

DMA limitations are usually a result of a device having fewer address lines than would be truly useful. The 24 lines described by the ISA specification are an obvious example; there is simply no way for an ISA-compliant device to address more than 16MB of physical memory. PCI devices are normally limited to a 32-bit address space, but a number of devices are limited to a smaller space as a result of dubious hardware design; as is so often the case, hardware designers have shown a great deal of creativity in this area. But users are not concerned with these issues; they just want their peripherals to work. So the kernel has to find a way to respect any given device's special limits while still using DMA to the greatest extent possible. 

The kernel's DMA API (described in [Documentation/DMA-API.txt](<https://lwn.net/Articles/543419/>)) abstracts and hides most of the details of making DMA actually work with any specific device. This API will, for example, endeavor to allocate memory that falls within the physical range supported by the target device. It will also transparently implement "bounce buffering" — copying data between a device-inaccessible buffer and an accessible buffer — if necessary. To do so, however, the DMA API must be informed of a device's addressing limits. That is done through the provision of a "DMA mask," a bitmask describing the memory range reachable by the device. The documentation describes the mask this way: 

The dma_mask represents a bit mask of the addressable region for the device. I.e., if the physical address of the memory anded with the dma_mask is still equal to the physical address, then the device can perform DMA to the memory. 

The problem, as recently [pointed out](<https://lwn.net/Articles/543424/>) by Russell King, is that the DMA mask is not always interpreted that way. He points to code like the following, found in `block/blk-settings.c`: 

```
void blk_queue_bounce_limit(struct request_queue *q, u64 dma_mask)
        {
    	unsigned long b_pfn = dma_mask >> PAGE_SHIFT;
```

What is happening here is that the code is right-shifting the DMA mask to turn it into a "page frame number" (PFN). If one envisions a system's memory as a linear array of pages, the PFN of a given page is simply its index into that array (though memory is not always organized so simply). By treating a DMA mask as, for all practical purposes, another way of expressing the PFN of the highest addressable page, the block code is changing the semantics of how the mask is interpreted. 

Russell explained how that can be problematic. On some ARM systems, memory does not start at a physical address of zero; the physical address of the first byte can be as high as 3GB (`0xc0000000`). If a system configured in this way has a device with a 26-bit address limitation (with the upper bits being filled in by the bus hardware), then its DMA mask should be set to `0xc3ffffff`. Any physical address within the device's range will be unchanged by a logical AND operation with this mask, while any address outside of that range will not. 

But what then happens when the block code right-shifts that mask to get a PFN from the mask? The result (assuming 4096-byte pages) is `0xc3fff`, which is a perfectly valid PFN on a system where the PFN of the first page will be `0xc0000`. And that is fine until one looks at the interactions with a global memory management variable called `max_low_pfn`. Given that name, one might imagine that it is the maximum PFN contained within low memory — the PFN of the highest page that is directly addressable by the kernel without special mappings. Instead, `max_low_pfn` is a _count_ of page frames in low memory. But not all code appears to treat it that way. 

On an x86 system, where memory starts at a physical address of zero (and, thus, a PFN of zero), that difference does not matter; the count and the maximum are the same. But on more complicated systems the results can be interesting. Returning to the same function in `blk-settings.c`: 

```
blk_max_low_pfn = max_low_pfn - 1;  /* Done elsewhere at init time */
    
        if (b_pfn < blk_max_low_pfn)
    	dma = 1;
        q->limits.bounce_pfn = b_pfn;
```

Here we have a real page frame number (calculated from the DMA mask) compared to a count of page frames, with decisions on how DMA must be done depending on the result. It would not be surprising to see erroneous results from such an operation; with regard to the discussion in question, it seems to have caused bounce buffering to be done when there was no need. One can easily see other kinds of trouble that could result from this type of confusion; inconsistent views of what a variable means will rarely lead to good things. 

Fixing this situation is not going to be straightforward; Russell had "no idea" of how to do it. Renaming `max_low_pfn` to something like `low_pfn_count` might be a first step as a way to avoid further confusion. Better defining the meaning of a DMA mask (or, at least, ensuring that the kernel's interpretation of a mask adheres to the existing definition) sounds like a good idea, but it could be hard to implement in a way that does not break obscure hardware — some of that code can be fragile indeed. One way or another, it seems that the DMA interface, which was designed by developers working with relatively straightforward hardware, is going to need some attention from the ARM community if it's going to meet that community's needs.
