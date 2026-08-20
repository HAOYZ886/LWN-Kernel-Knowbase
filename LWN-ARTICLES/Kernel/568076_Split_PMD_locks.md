---
title: Split PMD locks
url: https://lwn.net/Articles/568076/
date: "September 25, 2013"
category: "Memory management-Scalability; Scalability"
author: "By Jonathan Corbet September 25, 2013"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
September 25, 2013

Once upon a time, the standard response to scalability problems in the kernel was the introduction of finer-grained locking. That approach has its problems, though: the cache-line bouncing that locking activity creates can be a scalability problem in its own right. So much of the scalability work in the kernel has, in recent years, been focused on lockless algorithms instead. But, sometimes, there is little alternative to the introduction of finer-grained locks; a current memory management patch set illustrates one of those situations, with some additional complications. 

Page tables hold the mapping between a virtual address in some process's address space and the physical location of the memory behind that address. It is easy to think of the page table as a simple linear array indexed by the page frame number, but the reality is more complicated: page tables are implemented as a sparse tree with up to four levels. Various subfields of a virtual address are used to progress through the tree as shown here: 

> ![\[Virtual address translation\]](https://static.lwn.net/images/2013/address-translation.png)

Some systems do not have all four levels; no 32-bit system has the PUD ("page upper directory") level, for example, and some 32-bit systems may still get by with two-level page tables. Kernel code is written to deal with all four levels, though; the extra code will vanish in the compilation state for configurations with fewer levels. 

Changes to page tables can be made frequently; every movement of a page into or out of RAM must be reflected there, as must changes to the virtual address space (such as those made via an `mmap()` call). If the page table is not shared across processes, there is little potential for contention (and, thus, for scalability problems), since only one process will be making changes there. Sharing of the page tables, as happens most frequently in threaded workloads, changes the picture, though; it is not uncommon for threads to be making concurrent page table changes. The more concurrently running threads there are, the higher the potential for contention becomes. 

In some configurations, the entire page table is protected by a single spinlock (called `page_table_lock`) in the process's `mm_struct` structure. That lock was recognized as a scalability problem years ago; in response, locking for the lowest level of the page table tree (the PTE — "page table entry" — pages) was made per-PTE-page for multiprocessor configurations. But all of the other layers of the page table tree are still protected by `page_table_lock`; in general, changes at those levels are rare enough that more sophisticated locking is not worth the trouble. 

There is only one problem: as Kirill A Shutemov has [pointed out](<https://lwn.net/Articles/566770/>), that is not always true. When huge pages are in use, the PTE level of the page table tree is omitted. Instead, the entry in the next level up — the "page middle directory" or PMD — points directly to a much larger page. So, in effect, huge pages prune the page table tree back to three levels, with the PMD becoming the lowest level. The elimination of one level of translation is one of the reasons why huge pages can improve performance, though this effect is likely overshadowed by the large increase in the coverage of the translation lookaside buffer (TLB), which avoids a lot of address translations altogether. 

What Kirill has noted is that highly threaded workloads slow down considerably when the [transparent huge pages](<https://lwn.net/Articles/423584/>) feature is in use. Given that huge pages are meant to increase performance, this result is seen as surprising and undesirable. The problem is contention for the `page_table_lock`; the use of lots of huge pages greatly increases the number of changes made at the PMD level and, thus, increases contention. To address this problem, Kirill has put together a patch set that pushes the locking down to the PMD level, eliminating much of that contention. 

Locks are normally embedded within the data structures they protect, so one might be inclined to put a spinlock into the PMD. But the PMD is a hardware-defined structure; it is simply a page full of pointers to PTE pages or huge pages, with some status bits. There is no place there for an added spinlock, so that lock must go somewhere else. When fine-grained locking was implemented at the PTE level, the same problem was encountered; the solution was to shoehorn the lock into the already overcrowded `struct page`, which is the core structure for tracking the system's physical memory. (See [this article](<https://lwn.net/Articles/565097/>) for details on how `struct page` is put together). Kirill's patch replicates the approach used at the PTE level, putting the lock into `struct page`. 

The results would appear to be reasonably convincing. A benchmark designed to demonstrate the problem runs in 36.5 seconds with transparent huge pages off. When transparent huge pages are turned on in an unmodified kernel, the number of page faults drops from over 24 million to 50,000, but the run time increases to 49.9 seconds — not the speed improvement that one might hope for. Adding the patch, though, cuts the run time to 33.9 seconds, significantly faster than an unmodified kernel without transparent huge pages. By getting rid of the cost of the locking contention at the PMD level, Kirill's patch allows the benchmark to enjoy the performance benefits that come from using huge pages. 

There is one remaining problem, as [pointed out](<https://lwn.net/Articles/568090/>) by Peter Zijlstra: the patch as written will not work with the realtime preemption patch set. In the realtime world, spinlocks are sleeping locks; that makes them bigger, to the point that they will no longer fit into the tight space available in `struct page`. That structure will grow to accommodate the larger lock, but, given that there is one `page` structure for every page in the system, the memory cost of that growth is difficult to accept. The realtime developers resolved this problem at the PTE level by allocating the lock separately and putting a pointer into `struct page`. 

Something similar can certainly be done for the PMD-level locking. But, as Peter pointed out, the lock allocation means that the initialization of PMD pages is now subject to out-of-memory failures, complicating the code considerably. He hoped that the new code could be written with the assumption that PMD construction could fail so that the realtime tree would not have to carry a complicated add-on patch. Kirill is not required to cater to the needs of an out-of-tree patch set, but it's still nicer to avoid making life difficult for the realtime people if it can be avoided. So chances are, there will be another version of this set coming in the near future. 

Beyond that, though, this work appears to be mostly complete and in good shape. It could, thus, find its way into a mainline kernel sometime in the relatively near future.
