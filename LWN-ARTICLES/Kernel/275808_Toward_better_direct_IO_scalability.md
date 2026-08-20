---
title: Toward better direct I/O scalability
url: https://lwn.net/Articles/275808/
date: "March 31, 2008"
category: "Memory management-Scalability; Scalability; fast gup"
author: "By Jonathan Corbet March 31, 2008"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
March 31, 2008

Linux enthusiasts like to point out just how scalable the system is; Linux runs on everything from pocket-size devices to supercomputers with several thousand processors. What they talk about a little bit less is that, at the high end, the true scalability of the system is limited by the sort of workload which is run. CPU-intensive scientific computing tasks can make good use of very large systems, but database-heavy workloads do not scale nearly as well. There is a lot of interest in making big database systems work better, but it has been a challenging task. Nick Piggin appears to have come up with a logical next step in that direction, though, with a relatively straightforward set of core memory management changes. 

For some time, Linux has supported direct I/O from user space. This, too, is a scalability technology: the idea is to save processor time and memory by avoiding the need to copy data through the kernel as it moves between the application and the disks. With sufficient programming effort, the application should be able to make use of its superior knowledge of its own data access patterns to cache data more effectively than the kernel can; direct I/O allows that caching to happen without additional overhead. Large database management systems have had just that kind of programming effort applied to them, with the result that they use direct I/O heavily. To a significant extent, these systems use direct I/O to replace the kernel's paging algorithms with their own, specialized code. 

When the kernel is asked to carry out a direct I/O operation, one of the first things it must do is to pin all of the relevant user-space pages into memory and locate their physical addresses. The function which performs this task is `get_user_pages()`: 

```
int get_user_pages(struct task_struct *tsk, 
                           struct mm_struct *mm, 
    		       unsigned long start,
    		       int len,
    		       int write,
    		       int force,
    		       struct page **pages, 
    		       struct vm_area_struct **vmas);
```

A successful call to `get_user_pages()` will pin `len` pages into memory, those pages starting at the user-space address `start` as seen in the given `mm`. The addresses of the relevant `struct page` pointers will be stored in `pages`, and the associated VMA pointers in `vmas` if it is not `NULL`. 

This function works, but it has a problem (beyond the fact that it is a long, twisted, complex mess to read): it requires that the caller hold `mm->mmap_sem`. If two processes are performing direct I/O on within the same address space - a common scenario for large database management systems - they will contend for that semaphore. This kind of lock contention quickly kills scalability; as soon as processors have to wait for each other, there is little to be gained by adding more of them. 

There are two common approaches to take when faced with this sort of scalability problem. One is to go with more fine-grained locking, where each lock covers a smaller part of the kernel. Splitting up locks has been happening since the initial creation of the Big Kernel Lock, which is the definitive example of coarse-grained locking. There are limits to how much fine-grained locking can help, though, and the addition of more locks comes at the cost of more complexity and more opportunities to create deadlocks. 

The other approach is to do away with locking altogether; this has been the preferred way of improving scalability in recent years. That is, for example, what all of the work around read-copy-update has been doing. And this is the direction Nick has chosen to improve `get_user_pages()`. 

Nick's core observation is that, when `get_user_pages()` is called on a normal user-space page which is already present in memory, that page's reference count can be increased without needing to hold any locks first. As it happens, this is the most common use case. Behind that observation, though, are a few conditions. One is that it is not possible to traverse the page tables if those tables are being modified at the same time. To be guaranteed that this will not happen, the kernel must, before heading into the page table tree, disable interrupts in the current processor. Even then, the kernel can only traverse the currently-running process's page tables without holding `mmap_sem`. 

Lockless operation also will not work whenever pages which are not "normal" are involved. Some cases - non-present pages, for example - are easily detected from the information found in the page tables themselves. But others, such as situations where the relevant part of the address space has been mapped onto device memory with `mmap()`, are not readily apparent by looking at the associated page table entries. In this case, the kernel must look back at the controlling `vm_area_struct` (VMA) structure to see what is going on - and that cannot be done without holding `mmap_sem`. So it looks like there is no way to find out whether lockless operation is possible without first taking the lock. 

The solution here is to grab a free bit in the page table entry. The PTE for a page which is present in memory holds the physical page frame address. In such addresses, the bottom 12 bits (for architectures using 4096-byte pages) will always be zero, so they can be dedicated to other purposes. One of them is used to indicate whether the page is present in memory at all; others indicate writability, whether it's a user-space page, whether it is dirty, etc. Nick's patch grabs one of the few remaining bits and calls it "`PAGE_BIT_SPECIAL`," indicating "special" pages. These are pages which, for whatever reason, do not have a readily-accessible `struct page` associated with them. Marking "special" pages in the page tables can help in a number of places; one of those is making it possible to determine whether lockless `get_user_pages()` is possible on a given page. 

Once these pages are properly marked in the page tables, it is possible to write a function which makes a good attempt at a lockless `get_user_pages()`. Nick's [proposal](<http://lwn.net/Articles/275724/>) is called `fast_gup()`: 

```
int fast_gup(unsigned long start, int nr_pages, 
                     int write, struct page **pages);
```

This function has a much simpler interface than `get_user_pages()` because it does not handle many of the cases that `get_user_pages()` can deal with. It only works with the current process's address space, and it cannot return pointers to VMA structures. But it _can_ iterate through a set of page tables, testing each page for presence, writability, and "non-specialness," and incrementing each page's reference count (thus pinning it into physical memory) in the process. If it works, it's very fast. If not, it undoes things then falls back to `get_user_pages()` to do things the slow, old-fashioned way. 

How much is this worth? Nick claims a 10% performance improvement running "an OLTP workload" (one of those unnameable benchmark suites, perhaps) using IBM's DB2 DBMS system on a two-processor (eight-core) system. The performance improvement, he says, may be greater on larger systems. But even if it remains at "only" 10%, this work is a clear step in the right direction for this kind of workload. 

[**Update** : this interface was merged for the 2.6.27 kernel; the name was changed to `get_user_pages_fast()` but it is otherwise the same.]
