---
title: "Reverse mapping anonymous pages - again"
url: https://lwn.net/Articles/77106/
date: "March 24, 2004"
category: "Memory management-Object-based reverse mapping; Object-based reverse mapping; anonmm"
author: 
---

Two weeks ago, this page [described Andrea Arcangeli's "anon_vma" work](<https://lwn.net/Articles/74295/>) in some detail. This work, remember, is an attempt to improve memory scalability in the kernel by eliminating the reverse mapping ("rmap") chains used to find page table entries which reference a given page. The rmap chains can use significant amounts of low memory and can slow down `fork()` calls, so this work is of interest. 

Andrea has continued pushing the anon_vma effort through a series of kernel tree releases. The latest, [2.6.5-rc2-aa2](<https://lwn.net/Articles/77050/>), solves some of the remaining problems and comes with this statement: 

The next target is the merging of the prio_tree, but that will be a separated patch. After that this whole thing should be mergeable into mainline. 

(The prio_tree reference is about Rajesh Venkatasubramanian's [priority tree patch](<https://lwn.net/Articles/76621/>) which speeds the search for interesting virtual memory areas when a page is mapped a large number of times). 

Andrea's work is proceeding nicely, but it's worth noting that anon_vma is not the only approach to the implementation of an object-based reverse mapping scheme for anonymous memory. There is competition in the form of "anonmm" by Hugh Dickins. Hugh has recently reworked the patch and posted it for comments; interested parties can find this (multi-part) posting in the "patches" section below. 

The anon_vma patch works by creating a linked list of virtual memory areas (VMAs) which reference a given page. The anonmm patch, instead, creates a connection between an anonymous page and the `mm_struct` structures which reference it. The `mm_struct` is the top-level structure used to manage a process's virtual address space; it contains pointers to all of the process's VMAs and page tables, along with various bits of locking and housekeeping information. If you have a pointer to a process's `mm_struct` and a virtual address, you can quickly walk the page tables and determine whether the given address is a reference to a specific page. 

Most of the object-based reverse mapping has worked with the VMA structure. When performing reverse mapping of file-backed pages, use of the VMA structure is unavoidable; if multiple processes have mapped the file into their address spaces, each process likely has a different virtual address for the same page. The VMA structure contains the necessary information to determine which virtual address each process will have for a specific offset within a file. Once that address is found, the page of interest can be unmapped from that process's address space. 

Anonymous pages are different from file-backed pages, however; they are only shared between processes when a process forks (and, even then, it's a copy-on-write sharing). That means that, with one exception that we'll get to, shared anonymous pages have the same virtual address in every process. Thus, if you can track an anonymous page's virtual address and the processes which share that page, you can quickly find all of the page table entries referencing the page. 

The anonmm patch takes advantage of this fact. An anonymous page's virtual address is stored in the `index` field of the `page` structure. This field is normally used to give a page's offset within the file that backs it, but, since anonymous pages have no backing file, the field is available for this use. Hugh's patch then creates a new `anonmm` structure which is used to create a linked list of `mm_struct` structures; a pointer to this list is also stored in the `page` structure. The resulting data structure looks roughly like this: 

![\[cheesy anonmm diagram\]](https://static.lwn.net/images/ns/anonmm.png)

With this structure in place, the kernel can follow the pointers to quickly find the page tables referencing a given anonymous page. This approach, in theory, should be a little simpler and faster than the anon_vma technique; a process may have several VMAs for anonymous memory areas, but it will never have more than one `mm_struct`. 

There is one little problem with this whole scheme. It depends on the fact that every process has the same virtual address for a given, shared anonymous page. What happens when some wiseass process comes along and moves a chunk of anonymous memory with `mremap()`? At that point, the memory has a new address, and the anonmm algorithm will be unable to find it. Hugh's solution for this problem is to simply copy the pages being remapped. They are copy-on-write pages, so making copies will not create any correctness issues. The copying could be expensive - it may involve swapping in a number of pages so that they can be copied - but remapping of anonymous memory should be a sufficiently rare operation that a performance hit should not be a problem. 

Which scheme is truly faster? Martin Bligh has posted [a set of benchmarks](<https://lwn.net/Articles/77140/>) showing that, while both reverse mapping approaches are significantly faster than the mainline kernel, neither is obviously faster than the other. Andrea's work is marginally ahead in more tests than Hugh's, but, overall, the two produce roughly equivalent results. So, if one of these implementations does find its way into the 2.6 kernel, it will have to be chosen for reasons other than performance. Either that, or it will be some combination of the two; Andrea and Hugh are actively discussing ideas, so that sort of combination could happen.
