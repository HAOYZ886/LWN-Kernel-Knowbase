---
title: Active memory defragmentation
url: https://lwn.net/Articles/105021/
date: "October 5, 2004"
category: "Memory management-Large allocations"
author: 
---

"High order" allocations, in the kernel, are attempts to obtain multiple, contiguous pages for an application which needs more than one page in a single, physically-contiguous block. These allocations have always been a problem for the kernel to satisfy; once the system has been running for a while, physical memory is usually fragmented to the point that very few groups of adjacent, free pages exist. Last month, this page looked at [Nick Piggin's kswapd changes](<https://lwn.net/Articles/101230/>) which attempt to mitigate this problem somewhat. There are other people working in this area, however. 

One of those is Marcelo Tosatti, who posted [a patch](<https://lwn.net/Articles/104843/>) which adds active memory defragmentation to the kernel. At a high level, the algorithm used is relatively simple: to obtain free blocks of order _N_ , start with the largest, smaller blocks you can find, and try to relocate the contents of the pages immediately before and after the block. If enough pages can be moved, a larger block of free pages will have been created. 

Naturally, this process seems rather more complicated when looked at closely. Not all pages can be relocated; those which are locked or reserved, for example, are not touchable. The patch also declines to work with pages which are currently under writeback; until the writeback I/O completes, those pages must not move. A number of more complicated cases, such as moving pages which are part of a nonlinear mapping, are not handled with the current patch. 

If a page does appear to be relocatable, it must first be locked and have its contents copied to the new page. Then all page tables which reference the old page must be re-pointed to the new page. Reverse mapping information, if any, must be set correctly. If there is a copy of the page in swap, that copy must be connected with the new page. And so on. Marcelo's patch responds to many of the more complicated cases by simply refusing to move the page. Even so, Marcelo reports good results in creating large, contiguous blocks of free memory. 

Of course, there are a few glitches, including problems on SMP systems. But, says Marcelo, never fear: 

But it works fine on UP (for a few minutes :)), and easily creates large physically contiguous areas of memory. 

It was pointed out that this patch has some common features with a different effort: the drive to support hotpluggable memory. When memory is to be removed from the system, all pages currently stored in that memory must be relocated. In essence, the hotplug memory patches seek to create a large block of free memory which happens to cover a specific set of physical addresses. 

Dave Hansen [described](<https://lwn.net/Articles/105023/>) two patches adding hotplug memory support - one done at IBM, and one from Fujitsu. Each apparently has its strong and weak points. 

Between Marcelo's work and the hotplug patches, there is a significant amount of experience in moving pages aside to free blocks of memory. An effort to bring together those patches into a single one containing the best of each will probably be necessary before any can be merged. But the end result of that work could be an end to problems with high-order allocations.
