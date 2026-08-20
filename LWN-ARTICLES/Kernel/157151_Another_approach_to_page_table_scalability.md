---
title: Another approach to page table scalability
url: https://lwn.net/Articles/157151/
date: "October 26, 2005"
category: "Memory management-Scalability"
author: 
---

Scalability - making Linux perform on ever-larger systems - is a constant theme in kernel development. Some may feel that this work only benefits the very small percentage of users who have big-iron systems, but the fact remains that today's big iron is tomorrow's laptop. Remember that supporting 1GB of memory (and beyond) was once a big-iron issue. 

One scalability issue which has been receiving attention for a while is the single page table lock used to protect all operations on an address space's tables. Christoph Lameter's [page fault scalability patches](<http://lwn.net/Articles/114596/>) were covered here last year; that patch minimized the use of this lock, and introduced a number of atomic page table operations which could eliminate locking altogether in some situations. Those patches have never made it into the mainline, due to concerns over architecture support and general usefulness. The issue has not gone away, however. 

Hugh Dickins, who has been thrashing up the -mm tree with memory management patches for the last few weeks, has now posted [a new approach](<https://lwn.net/Articles/156884/>) to paging scalability. Rather than play tricks to minimize page table lock hold times, Hugh has taken the classic approach of going to finer-grained locking. So, with his patch, the address space page table lock no longer controls access to individual pages within the tables. Instead, each page gets its own lock. 

Pushing the lock down to individual page-table pages will eliminate much of the contention for the lock on large, multi-processor systems. It should work especially well for multi-threaded processes (which share the same address space) on those systems. Splitting the lock also enables the kernel to work at reclaiming pages in one part of an address space while pages are being faulted into another part. So, in some situations, this split should be a big performance win. 

There is, however, the little problem of where to store the lock. Putting it into the page tables themselves is not an option; the format of page tables tends to be driven by the underlying hardware architecture, and CPU designers do not usually make provisions for in-table locks. One could create an array of locks elsewhere in the system, but a large system can contain a great many page table pages. The space overhead of a large lock array could thus get painful. Using a smaller, hashed array, as is done in other parts of the kernel, is an option, but Hugh didn't go that way. Instead, he put the lock into the `page` structures representing the page table pages in the system memory map. Expanding that structure is not an option, but it seems that the `private` field of `struct page` is not currently used on page table pages. So, with a bit of preprocessor trickery, that field becomes a spinlock for page table pages. 

This finer-grained locking should be helpful on larger systems, but it is likely to just be more overhead on uniprocessor or small SMP systems. So it is only enabled on kernels configured for four CPUs or more. Depending on the results from wider testing, that threshold may be raised before the patch is proposed for merging into the mainline.
