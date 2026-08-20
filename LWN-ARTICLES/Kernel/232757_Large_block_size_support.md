---
title: Large block size support
url: https://lwn.net/Articles/232757/
date: "May 2, 2007"
category: "Block layer; Memory management-Page cache"
author: 
---

On its face, it doesn't seem like Christoph Lameter's [large block size support patch](<http://lwn.net/Articles/231793/>) would be that controversial. This patch set equips the page cache to hold blocks which are larger than the system's page size by storing them in higher-order, compound pages. That, in turn, enables filesystems to work with larger blocks. The patch should make operations on large files more efficient and improve the kernel's support for some types of hardware. The patch might eventually get merged, but not before more discussion has happened. 

The problem is that this patch is not without its difficulties. It adds a certain amount of complexity to the core virtual memory subsystem to implement what is, in all reality, a feature which has been rejected before: larger pages. The patch currently ducks the most difficult part of the problem - handling faults on larger pages, needed to make `mmap()` work - meaning that more complexity can be expected in the future. Larger blocks in the page cache means more demand for higher-order pages, which are already in short supply on many systems; that, in turn, means that the anti-fragmentation patches would almost certainly be needed as well. Use of larger pages in the page cache can also lead to more internal fragmentation and less efficient memory use. 

For all these reasons, Andrew Morton has been [expressing some reservations](<https://lwn.net/Articles/232763/>): 

And make no mistake: the latter disadvantage is huge. Because if we do the PAGE_CACHE_SIZE hack (sorry, but it _is_), we have to do it *for ever*. Maintaining and enhancing core MM and VFS becomes harder and more costly and slower and more buggy *for ever*. The ramp for people to become competent on core MM becomes longer. Our developer pool becomes smaller, and proportionally less skilled. 

Andrew is not necessarily opposed to the patch; he is more concerned that it not be merged until it has been carefully compared with the alternatives. He suggests keeping the page cache entry size unchanged, but trying to allocate entries in higher-order groups. That would result in larger blocks being stored contiguously in memory without the memory subsystem changes. Filesystems could use those larger blocks, and hardware could treat them as single units in scatter/gather lists for DMA, leading to more efficient operations. 

Another possibility which has been raised is raising the maximum size of hardware scatter/gather lists or allowing them to be chained. Drivers could then set up larger I/O operations, improving efficiency without requiring the other changes. 

Still, there is support for Christoph's patch. It would make support of larger blocks relatively straightforward for the lower layers, perhaps enabling the removal of some real hacks found in some drivers and filesystems now. The patch would also allow ext3 filesystems with larger block sizes - sometimes created on ia64 systems, which use larger pages - to be mounted on other architectures. Christoph Hellwig [likes the idea](<https://lwn.net/Articles/232766/>) that a higher-order page cache could force a solution to the longstanding problem of physical memory fragmentation. To many, it seems like a straightforward and necessary solution to a longstanding problem. 

So the large block size idea is unlikely to just go away. It may be a while, though, before its proponents can do enough homework and benchmarking to fully address the worries which have been expressed. Fundamental changes are often the ones which take the longest to get into the kernel, so there is little that is surprising here. Just don't ask for a prediction of the final outcome.
