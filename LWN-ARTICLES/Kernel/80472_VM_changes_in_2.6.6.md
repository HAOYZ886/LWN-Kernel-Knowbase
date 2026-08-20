---
title: VM changes in 2.6.6
url: https://lwn.net/Articles/80472/
date: "April 14, 2004"
category: "Memory management; Object-based reverse mapping; anon vma"
author: 
---

Among the patches merged into the upcoming 2.6.6 release is a set of virtual memory changes. Changes to such a fundamental subsystem are always of interest, especially in the middle of a "stable" kernel series. Here, then, is a quick discussion of what has transpired. 

In response to the reverse mapping VM discussions over the last month or so, Hugh Dickins has posted a series of patches which prepare the kernel for a full object-based reverse-mapping scheme and the removal of the per-page PTE chains. Hugh's patches carefully leave room for the inclusion of either his [anonmm](<https://lwn.net/Articles/77106/>) patches or Andrea Arcangeli's [anon_vma](<https://lwn.net/Articles/75198/>) work, though he seems to expect that anon_vma will win out. The full set of patches posted so far can be found in the "memory management" part of the "patches and updates" section, below. 

Of those patches, the first three have been merged as of this writing. [rmap 1](<http://lwn.net/Articles/79913/>) simply creates a new include file (`linux/rmap.h`) and moves much of the reverse-mapping declarations there. The second patch ([rmap 2](<http://lwn.net/Articles/79914/>)) changes the way the swap subsystem keeps track of swap cache pages; this change is needed to free up a couple of `struct page` fields for reverse mapping tasks. Finally, [rmap 3](<http://lwn.net/Articles/79915/>) finishes out the `struct page` work for various architectures. 

Later patches in Hugh's series get more ambitious; [rmap 7](<https://lwn.net/Articles/80242/>) adds object-based reverse mapping for file-backed memory. Those patches have not been merged as of this writing, however. 

A completely different set of patches which changes how the page cache works has been merged. The [description](<https://lwn.net/Articles/80474/>) of this work, as written by Andrew Morton, reads: 

The basic problem which we (mainly Daniel McNeil) have been struggling with is in getting a really reliable fsync() across the page lists while other processes are performing writeback against the same file. It's like juggling four bars of wet soap with your eyes shut while someone is whacking you with a baseball bat. 

This work made some fundamental changes in how page cache pages are tracked. The `struct page` structure has long included a field called "`list`", being a `list_head` structure used to track the state of the page. When the page is marked dirty, or placed under I/O, it is put on a list with other such pages. Unfortunately, managing those lists as the state of the page changes proves to be difficult; hence the juggling analogy. 

In response, the page lists have been removed altogether; as a side-benefit, this change shrinks `struct page` by eight bytes - a significant savings, considering that there is one such structure for every physical page in the system. The lists have been replaced with an enhanced radix tree which supports "tagging" of pages. When a page is dirtied, it is simply marked dirty in the radix tree, rather than being added to a list. Similarly, pages which are currently being written back to disk are marked. A new set of radix tree operations allows the kernel to find these pages when the need arises. Searching the tree is not as fast as following a dedicated list, but the radix tree implementation appears to be fast enough that few people will notice the difference. 

These changes required touching a lot of VM and page cache code; every user of the `page->list` field had to be fixed. As a result of the changes, the order in which dirty pages are written to disk has changed; writing always happens in file-offset order now. This change appears to be an improvement for many applications; Andrew reports as much as 30% faster benchmark results. I/O can slow down for some situations involving parallel writes on SMP systems, however.
