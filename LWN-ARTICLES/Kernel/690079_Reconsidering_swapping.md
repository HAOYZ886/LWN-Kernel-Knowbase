---
title: Reconsidering swapping
url: https://lwn.net/Articles/690079/
date: "June 7, 2016"
category: "Memory management-Swapping"
author: "By Jonathan Corbet June 7, 2016"
---

By **Jonathan Corbet**  
June 7, 2016

"Swapping" is generally considered to be a bit of a dirty word among long-time Linux users, who will often go to considerable lengths to avoid it. The memory-management (MM) subsystem has been designed to facilitate that avoidance whenever possible. Now, though, MM developer Johannes Weiner is suggesting that, in light of recent developments in hardware, swapping deserves another look. His [associated patch set](<https://lwn.net/Articles/690069/>) includes benchmark results indicating that he may be on to something. 

#### Is swapping still bad?

User-accessible memory on a Linux system is divided into two broad classes: file-backed and anonymous. File-backed pages (or page-cache pages) correspond to a segment of a file on disk; if they do not contain newly written data that has not yet made it back to persistent storage, these pages can be easily reclaimed for other uses. Anonymous pages do not correspond to a file on disk; they hold the run-time data generated and used by a process. Reclaiming an anonymous page requires writing its contents to the swap device. 

As a general rule, reclaiming anonymous pages (swapping) is seen as being considerably more expensive than reclaiming file-backed pages. One of the key reasons for this difference is that file-backed pages can be read from (and written to) persistent storage in large, contiguous chunks, while anonymous pages tend to be scattered randomly on the swap device. On a rotating storage device, scattered I/O operations are expensive, so a system that is doing a lot of swapping will slow down considerably. It is far faster to read a bunch of sequentially stored file-backed pages — and, since the file is usually current on disk, those pages may not need to be written at reclaim time at all. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

Swapping is so much slower that many administrators try to configure their systems to do as little swapping as possible. At its most extreme, this can involve not setting up a swap device at all; this common practice deprives the kernel of any way to reclaim anonymous pages, regardless of whether that memory could be put to better use elsewhere. An intermediate step is to use the `swappiness` tuning knob ([described here](<https://lwn.net/Articles/83588/>) in 2004) to bias the system strongly toward reclaiming file-backed pages. Setting `swappiness` to zero will cause the kernel to swap only when memory pressure reaches dire levels. 

Johannes starts off his patch set by noting that this mechanism was designed around the characteristics of rotating storage. Anytime the drive used for swapping needed to perform a seek — which would happen often with randomly placed I/O — throughput would drop dramatically. Hence the strong aversion to swapping if it could possibly be avoided. But, Johannes notes, technology has moved on and some of these decisions should be reconsidered: 

With the proliferation of fast random IO devices such as SSDs and persistent memory, though, swap becomes interesting again, not just as a last-resort overflow, but as an extension of memory that can be used to optimize the in-memory balance between the page cache and the anonymous workingset even during moderate load. Our current reclaim choices don't exploit the potential of this hardware. 

Not only should the system be more willing to swap out anonymous memory, Johannes claims, but, at times, swapping may well be a better option than reclaiming page-cache pages. That could be true if the swap device is faster than the drives used to hold files; it is also true if the system is reclaiming needed file-backed pages while memory is clogged with unused anonymous pages. 

#### Deciding when to swap

The first step in the patch set is to widen the range of possible settings for the `swappiness` knob. In current kernels, it can go from zero (no swapping at all if possible) to 100 (reclaim anonymous and file-backed pages equally). Johannes raises the maximum to 200; at that value, the system will strongly favor swapping. That is a possibility nobody has ever wanted before, but fast drives have now made it useful. 

While there may always be a use for knobs like `swappiness`, the best kind of system is one that tunes itself without the need for administrator intervention. So Johannes goes on to change the mechanism that decides whether to reclaim pages from the anonymous least-recently-used (LRU) list or the file-backed LRU. For each list, he introduces the concept of the "cost" of reclaiming a page from that list; the reclaim code then directs its efforts toward the list that costs the least to reclaim pages from. 

The first step is to track the cost of "rotations" on each LRU. The MM code does its best to reclaim pages that are not in active use. This is done by occasionally passing through the list and clearing the "referenced" bit on each page. The pages that are used thereafter will have that bit set again; those that still have the referenced bit cleared on a subsequent scan have not been touched in the meantime. Those pages are the least likely to be missed and are, thus, the first to be reclaimed. Pages which _have_ been referenced, instead, are "rotated" to the head of the list, giving them a period of time before they are again considered for reclaim. 

That rotation costs a bit of CPU time. If a particular LRU list has a lot of referenced pages in it, scanning that list will use a relatively large amount of time for a relatively small payoff in reclaimable pages; in this case, the kernel may well be better off scanning the other list, which may have more unused pages. To that end, Johannes's patch set tracks the number of rotated pages and uses it to establish the cost of reclaiming from each list. 

While rotation has a cost, that cost pales relative to that of reclaiming a page that will be quickly faulted back into memory — even if it is written to a fast device in the meantime. As it happens, Johannes [added a mechanism](<https://lwn.net/Articles/495543/>) to track "refaulted" pages back in 2012; it is used in current kernels to determine how large the active working set is at any given time. This mechanism can also tell the kernel whether it is reclaiming too many anonymous or file-backed pages. The final patch in the set uses refault information to adjust the cost of reclaiming from each LRU; if pages taken from one LRU are quickly faulted back in, the kernel will turn its attention to the other LRU instead. 

In the current patch set, the cost of a refaulted page is set to be 32 times the cost of a rotated page. Johannes suggests in the comments that this value is arbitrary and may change in the future. For now, the intent is to cause refaults to dominate in the cost calculation, but, he says, there may well be settings where refaults cost less than rotations. 

The patch set comes with a number of benchmarks to show its impact on performance. A PostgreSQL benchmark goes from 81 to 105 transactions per second with the patches applied; the refault rate is halved, and kernel CPU time is reduced. A streaming I/O benchmark, which shouldn't create serious memory pressure, runs essentially unchanged. So, as far as Johannes's testing goes, the numbers look good. 

Memory-management changes are fraught with potential hazards, though, and it is entirely possible that other workloads will be hurt by these changes. The only way to gain confidence that this won't happen is wider testing and review. This patch set is quite young; there have been some favorable reviews, but that testing has not yet happened. Thus, it may be a while before this code goes anywhere near a mainline kernel. But it has been clear for a while that the MM subsystem is going to need a number of changes to bring its design in line with current hardware; this work may be a promising step in that direction.
