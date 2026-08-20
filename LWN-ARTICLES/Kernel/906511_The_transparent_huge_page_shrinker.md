---
title: The transparent huge page shrinker
url: https://lwn.net/Articles/906511/
date: "September 8, 2022"
category: "Huge pages; Memory management-Huge pages"
author: "By Jonathan Corbet September 8, 2022"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
September 8, 2022

Huge pages are a mechanism implemented by the CPU that allows the management of memory in larger chunks. Use of huge pages can increase performance significantly, which is why the kernel has a "transparent huge page" mechanism to try to create them when possible. But a huge page will only be helpful if most of the memory contained within it is actually in use; otherwise it is just an expensive waste of memory. [This patch set](<https://lwn.net/ml/linux-kernel/cover.1661461643.git.alexlzhu@fb.com/>) from Alexander Zhu implements a mechanism to detect underutilized huge pages and recover that wasted memory for other uses. 

The base page size on most systems running Linux is 4,096 bytes, a number which has remained unchanged for many years even as the amount of memory installed in those systems has grown. By grouping (typically) 512 physically contiguous base pages into a huge page, it is possible to reduce the overhead of managing those pages. More importantly, though, huge pages take far fewer of the processor's scarce translation lookaside buffer (TLB) slots, which cache the results of virtual-to-physical address translations. TLB misses can be quite expensive, so expanding the amount of memory that can be covered by the TLB (as huge pages do) can improve performance significantly. 

The downside of huge pages (as with larger page sizes in general) is internal fragmentation. If only part of a huge page is actually being used, the rest is wasted memory that cannot be used for any other purpose. Since such a page contains little useful memory, the hoped-for TLB-related performance improvements will not be realized. In the worst cases, it would clearly make sense to break a poorly utilized huge page back into base pages and only keep those that are clearly in use. The kernel's memory-management subsystem can break up huge pages to, among other things, facilitate reclaim, but it is not equipped to focus its attention specifically on underutilized huge pages. 

Zhu's patch set aims to fill that gap in a few steps, the first being figuring out which of the huge pages in the system are being fully utilized — and which are not. To that end, a scanning function is run every second from a kernel workqueue; each run will look at up to 256 huge pages to determine how fully each is utilized. Only anonymous huge pages are scanned; this work doesn't address file-backed huge pages. The results can be read out of `/sys/kernel/debug/thp_utilization` in the form of a table like this: 

```
Utilized[0-50]: 1331 680884
        Utilized[51-101]: 9 3983
        Utilized[102-152]: 3 1187
        Utilized[153-203]: 0 0
        Utilized[204-255]: 2 539
        Utilized[256-306]: 5 1135
        Utilized[307-357]: 1 192
        Utilized[358-408]: 0 0
        Utilized[409-459]: 1 57
        Utilized[460-512]: 400 13
        Last Scan Time: 223.98
        Last Scan Duration: 70.65
```

This output (taken from the cover letter) is a histogram showing the number of huge pages containing a given number of utilized base pages. The first line, for example, shows the number of huge pages for which no more than 50 base pages are in active use. There are 1,331 of those pages, containing 680,884 unused base pages. There is a clear shape to the results: nearly all pages fall into one of the two extremes. As a general rule, a huge page is either fully utilized or almost entirely unused. 

An important question to answer when interpreting this data is: how does the code determine which base pages within a huge page are actually used? The CPU and memory-management unit do not provide much help in this task; if the memory is mapped as a huge page, there is no per-base-page "accessed" bit to look at. Instead, Zhu's patch scans through the memory itself to see what is stored there. Any base pages that contain only zeroes are deemed to be unused, while those containing non-zero data are counted as being used. It is clearly not a perfect heuristic; a program could initialize pages with non-zero data then never touch them again. But it may be difficult to design a better one that doesn't involve actively breaking apart huge pages into base pages. 

The results of this scanning identify a clear subset of the huge pages in a system that should perhaps be broken apart. In current kernels, though, splitting a zero-filled huge page will result in the creation of a lot of zero-filled base pages — and no immediate recovery of the unused memory. Zhu's patch set changes the splitting of huge pages so that it simply drops zero-filled base pages rather than remapping them into the process's address space. Since these are anonymous pages, the kernel will quickly substitute a new, zero-filled page should the process eventually reference one of the dropped pages. 

The final step is to actively break up underutilized huge pages when the kernel is looking for memory to reclaim. To that end, the scanner will add the least-utilized pages (those in the `0-50` bucket shown above) to a new linked list so that they can be found quickly. A new shrinker is registered with the memory-management subsystem that can be called when memory is tight. When invoked, that shrinker will pull some entries from the list of underutilized huge pages and split them, resulting in the return of all zero-filled base pages found there to the system. 

Most of the comments on this patch set have been focused on details rather than the overall premise. David Hildenbrand [expressed a concern](<https://lwn.net/ml/linux-kernel/00f2dee2-ebc1-e732-f230-bc5b17da9f80@redhat.com/>) that unmapping zero-filled pages in an area managed by a [`userfaultfd()`](<https://man7.org/linux/man-pages/man2/userfaultfd.2.html>) handler could create confusion if that handler subsequently receives page faults it was not expecting. Zhu [answered](<https://lwn.net/ml/linux-kernel/A92C4953-F9BC-4687-BB03-2202D94D6F5D@fb.com/>) that, if this is a concern, zero-filled base pages in `userfaultfd()`-managed areas could be mapped to the kernel's shared zero page instead. 

The kernel has supported transparent huge pages since the feature was [merged](<https://lwn.net/Articles/423584/>) into the 2.6.38 kernel in 2011, but it is still not enabled for all processes. One of the reasons for holding back is the internal-fragmentation problem, which can outweigh the benefits that transparent huge pages provide. Zhu's explicit goal is to make that problem go away, allowing the enabling of transparent huge pages by default. If this work is successful, it could represent an important step for a longstanding kernel feature that, arguably, has never quite lived up to its potential.
