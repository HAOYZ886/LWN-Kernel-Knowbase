---
title: "VM followup: page migration and fragmentation avoidance"
url: https://lwn.net/Articles/160201/
date: "November 16, 2005"
category: "Hotplug-Memory; Memory management-Large allocations; NUMA"
author: 
---

Page migration is the act of moving a process's pages from one part of the system to another. Often, the motivation is moving pages between NUMA nodes in the hope of improving performance. When this page last [looked at the page migration patch set](<http://lwn.net/Articles/157066/>), it worked by forcing target pages out to the swap device. When the owning process later faults them in, these pages will end up on the desired node. This technique works, but it is not optimal: it would be nicer to avoid having to write the pages to disk and read them back in. 

Christoph Lameter has now followed up with the [direct migration patch set](<http://lwn.net/Articles/160202/>), which does away with the side-trip to the swap device. A look at the patch shows why things were not done this way in the first place; direct page migration involves rather more than simply copying the data over. The first step, after choosing a target page, is to lock that page so that nobody else will mess with it. There might currently be I/O active which involves that page, so the kernel must wait for any such I/O to complete. Only then can the real migration work begin. 

The kernel must establish a swap cache entry for the page, even though it intends to avoid writing the page to swap. This entry will cause the right thing to happen if a process faults on the page while it is being moved. Then all references to the page (page table entries) are unmapped. With luck, all references will go away; if references remain for any reason, the page cannot be moved. 

Actually moving the page involves copying a subset of the page status bits over, copying the page data itself, then copying the rest of the status bits. The old page is cleared out and freed. If any writeback has been queued up for the new page, it is set in motion. Then it's just a matter of cleaning up, and the page has been successfully moved. 

If the kernel runs out of free pages on the target node, it will fall back to the swap-based mechanism. So that stage of this patch's evolution remains useful. 

With this code in place, the kernel has the support it needs to try to keep a process's pages in local memory. The migration code might also prove useful for hotplug memory uses, where all pages must be vacated from a given region. Indeed, some of this code was originally written for hotplug applications. But, at this point, the migration is done on a best-effort basis. For NUMA systems, failure to move a page results in worse performance, but nothing particularly severe. For hotplug memory, instead, this sort of failure will block a memory remove operation altogether. Moving all pages in a region with 100% certainty remains a difficult problem without a complete solution at this time. 

One of the pieces of such a solution might be active memory defragmentation which, among other things, works to keep non-movable memory allocations out of memory regions which might be removed. When we looked at active defragmentation [last week](<http://lwn.net/Articles/159110/>), that patch set looked like it was in trouble. The overhead of the defragmentation code seemed to be too high, and a number of developers (Linus included) felt that this sort of functionality should be implemented using the kernel's zone system, rather then with a new layer in the memory allocator. 

Defragmentation hacker Mel Gorman doesn't give up that easily, however. He has posted [a new, "light" version of the defragmentation patch](<http://lwn.net/Articles/160125/>) which, he hopes, will be better received. As he describes it: 

This is a much simplified anti-defragmentation approach that simply tries to keep kernel allocations in groups of 2^(MAX_ORDER-1) and easily reclaimed allocations in groups of 2^(MAX_ORDER-1). It uses no balancing, tunables special reserves and it introduces no new branches in the main path. For small memory systems, it can be disabled via a config option. In total, it adds 275 new lines of code with minimum changes made to the main path. 

In this version of the patch, a new GFP flag (`__GFP_EASYRCLM`) is added; its presence indicates an allocation which the kernel can easily get back should the need arise. It is used for user-space pages (which can usually be forced out to backing store) and in a few other situations, such as for some kernel buffers. The buddy allocator already keeps track of memory in large chunks; the new code simply steers reclaimable allocations toward some chunks, while keeping the non-reclaimable allocations in others. In this way, it is hoped, there will be no situations where one non-movable page blocks the freeing of the large, contiguous region in which it is located. 

The patch works by creating a "usemap" array tracking which kind of allocation is being done from each large chunk of memory. Mel also had to split the per-CPU free lists which are used to perform fast single-page allocations; now there are two such lists, one for each allocation type. From there, it is just a matter of taking allocations from the proper pile, depending on the `__GFP_EASYRCLM` flag. 

This version certainly reduces the footprint and overhead of the defragmentation patches. It is still not the zone-based approach that others were pushing for, however. So it remains to be seen whether "active defragmentation lite" is, in the end, better received than its predecessors.
