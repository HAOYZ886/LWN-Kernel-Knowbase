---
title: "Virtual Memory II: the return of objrmap"
url: https://lwn.net/Articles/75198/
date: "March 10, 2004"
category: "Memory management-Object-based reverse mapping; Object-based reverse mapping; anon vma"
author: 
---

Andrea Arcangeli not only wants to make the Linux kernel scale to and beyond 32GB of memory on 32-bit processors; he seems to be in a real hurry. There are, it would seem, customers waiting for a 2.6-based distribution which can run in such environments. 

For Andrea, the real culprit in the exhaustion of low memory is clear: it's the reverse-mapping virtual memory ("rmap") code. The rmap code was first described on this page [in January, 2002](<http://lwn.net/2002/0124/kernel.php3>); its purpose is to make it easier for the kernel to free memory when swapping is required. To that end, rmap maintains, for each physical page in the system, a chain of reverse pointers; each pointer indicates a page table which has a reference for that page. By following the rmap chains, the kernel can quickly find all mappings for a given page, unmap them, and swap the page out. 

The rmap code solved some real performance problems in the kernel's virtual memory subsystem, but it, too has a cost. Every one of those reverse mapping entries consumes memory - low memory in particular. Much effort has gone into reducing the memory cost of the rmap chains, but the simple fact remains: as the amount of memory (and the number of processes using that memory) goes up, the rmap chains will consume larger amounts of low memory. Eliminating the rmap overhead would go a long way toward allowing the kernel to scale to larger systems. Of course, one wants to eliminate this overhead while not losing the benefits that rmap brings. 

Andrea's approach is to bring back and extend the object-based reverse mapping patches. The initial object-based patch was created by Dave McCracken; LWN [covered this patch](<http://lwn.net/Articles/23732/>) a year ago. Essentially, this patch eliminates the rmap chains for memory which maps a file by following pointers "the long way around" and searching candidate virtual memory areas (VMAs). Andrea has [updated this patch](<https://lwn.net/Articles/74812/>) and fixed some bugs, but the core of the patch remains the same; see last year's description for the details. 

[Last week](<https://lwn.net/Articles/73100/>), we raised the possibility that the virtual memory subsystem could see fundamental changes in the course of the 2.6 "stable" series. This week, Linus [confirmed that possibility](<https://lwn.net/Articles/75217/>) in response to Andrea's object-based reverse mapping patch: 

I certainly prefer this to the 4:4 horrors. So it sounds worth it to put it into -mm if everybody else is ok with it. 

Assuming this work goes forward, it has the usual implications for the stable kernel. Even assuming that it stays in the -mm tree for some time, its inclusion into 2.6 is likely to destabilize things for a few releases until all of the obscure bugs are shaken out. 

Dave McCracken's original patch, in any case, only solves part of the problem. It gets rid of the rmap chains for file-backed memory, but it does nothing for anonymous memory (basic process data - stacks, memory obtained with `malloc()`, etc.), which has no "object" behind it. File-backed memory is a large portion of the total, especially on systems which are running large Oracle servers and use big, shared file mappings. But anonymous memory is also a large part of the mix; it would be nice to take care of the rmap overhead for that as well. 

To that end, Andrea has posted [another patch](<https://lwn.net/Articles/75098/>) (in preliminary form) which provides object-based reverse mapping for anonymous memory as well. It works, essentially, by replacing the rmap chain with a pointer to a chain of virtual memory area (VMA) structures. 

Anonymous pages are always created in response to a request for memory from a single process; as a result, they are never shared at creation time. Given that, there is no need for a new anonymous page to have a chain of reverse mappings; we know that there can be only a single mapping. Andrea's patch adds a union to `struct page` which includes the existing `mapping` pointer (for non-anonymous memory) and adds a couple of new ones. One of those is simply called `vma`, and it points to the (single) VMA structure pointing to the page. So if a process has several non-shared, anonymous pages in the same virtual memory area, the structure looks somewhat like this: 

![\[Anonymous reverse mapping\]](https://static.lwn.net/images/ns/anonvma1.png)

With this structure, the kernel can find the page table which maps a given page by following the pointers through the VMA structure. 

Life gets a bit more complicated when the process forks, however. Once that happens, there will be multiple page tables pointing to the same anonymous pages and a single VMA pointer will no longer be adequate. To deal with this case, Andrea has created a new "`anon_vma`" structure which implements a linked list of VMAs. The third member of the new `struct page` union is a pointer to this structure which, in turn, points to all VMAs which might contain the page. The structure now looks like: 

![\[anonvma\]](https://static.lwn.net/images/ns/anonvma2.png)

If the kernel needs to unmap a page in this scenario, it must follow the linked list and examine every VMA it finds. Once the page is unmapped from every page table found, it can be freed. 

There are some memory costs to this scheme: the VMA structure requires a new `list_head` structure, and the `anon_vma` structure must be allocated whenever a chain must be formed. One VMA can refer to thousands of pages, however, so a per-VMA cost will be far less than the per-page costs incurred by the existing rmap code. 

This approach does incur a greater computational cost. Freeing a page requires scanning multiple VMAs which may or may not contain references to the page under consideration. This cost will increase with the number of processes sharing a memory region. Ingo Molnar, who is fond of O(1) solutions, [is nervous](<https://lwn.net/Articles/75225/>) about object-based schemes for this reason. According to Ingo, losing the possibility of creating an O(1) page unmapping scheme is a heavy cost to pay for the prize of making large amounts of memory work on obsolete hardware. 

The solution that Ingo would like to see, instead, is to reduce the per-page memory overhead by reducing the number of pages. The means to that end is [page clustering](<https://lwn.net/Articles/23785/>) \- grouping adjacent hardware pages into larger virtual pages. Page clustering would reduce rmap overhead, and reduce the size of the main kernel memory map as well. The available page clustering patch is even more intrusive than object-based reverse mapping, however; it seems seriously unlikely to be considered for 2.6.
