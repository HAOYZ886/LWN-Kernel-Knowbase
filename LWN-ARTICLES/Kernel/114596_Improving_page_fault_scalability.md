---
title: Improving page fault scalability
url: https://lwn.net/Articles/114596/
date: "December 7, 2004"
category: "Memory management-Scalability"
author: 
---

Beyond doubt, many LWN readers have been concerned with how page fault performance might be improved on their 512-processor desktop systems. Christoph Lameter has been working on the answer for some months now; [his page fault scalability patches](<https://lwn.net/Articles/113870/>) are reaching a point where they will likely be considered for inclusion after 2.6.10 comes out. This patch is an interesting example of the kind of changes which must be made to support large numbers of processors. 

One of the virtual memory subsystem's core data structures is `struct mm_struct`. This structure tracks the virtual address space used by one or more processes. It contains pointers to the page tables, to the virtual memory area (VMA) structures, and more. Processes typically have their own `struct mm_struct`, but threads which share memory also share the same `mm_struct`. 

Access to this structure is serialized by two mechanisms. A semaphore (`mmap_sem`) controls access to the `mm_struct` structure itself, and a spinlock (`page_table_lock`) guards the page tables. When the status of a page must be changed in the page tables, the kernel must first take the `page_table_lock` to avoid creating confusion with the other processors on the system. When he looked at the scalability of the kernel's page fault handling code, Christoph identified this lock as a problem. When many processors are trying to simultaneously make changes to a single set of page tables, they end up spending a lot of time busy-waiting for the page table lock. Improving the performance of this code thus requires reducing the use of that lock. 

The first step in this process is [a patch](<http://lwn.net/Articles/113871/>) which causes the VM subsystem to hold `page_table_lock` for shorter periods of time. The lock is dropped for portions of the code which have no need of it, and later reacquired if needed. It is a fairly straightforward exercise in lock breaking which helps scalability, but does not solve the whole problem. 

The core of the patch is [a set of atomic page table entry functions](<https://lwn.net/Articles/113875/>) which can modify individual PTEs with no locking required. Rather than acquiring `page_table_lock`, making a PTE change, then dropping the lock, the kernel can simply make a call to: 

```
int ptep_cmpxchg(struct vm_area_struct *vma, unsigned long address, 
                         pte_t *ptep, pte_t oldval, pte_t newval);
```

This function uses the `cmpxchg` instruction (or whatever variant or emulation may be available, depending on the architecture) to compare the page table entry pointed to by `ptep` with `oldval`; if the two match, the entry is set to `newval` and `oldval` is returned. If the two do not match, the current thread lost a race with another processor which changed the PTE first; in that case, the PTE is not modified further and the function returns zero. Kernel code which uses `cmpxchg` typically will retry a modification when this sort of race occurs; Christoph's code, instead, is able to assume that the competing thread did the same thing as the one it raced against: marked the page as being present in memory. So no retries are needed. 

With that change, pages can be brought into the working set and made available without having to take the `page_table_lock` \- except for one last place. The `mm_struct` structure contains two fields (`rss` and `anon_rss`) which track the total number of in-memory pages referenced by this address space (the "resident set size"). When a page is brought in (or forced out), these fields must be incremented or decremented accordingly. Access to `rss` and `anon_rss` is controlled by `page_table_lock`. Getting rid of that last use of the lock has required a surprising amount of work on Christoph's part. 

The first implementation turned the RSS fields into `atomic_t` variables, so that they could be operated on without locking. This solution worked, but it had some shortcomings: (1) they could only be 32-bit variables, since not all architectures support 64-bit atomic types, (2) the atomic operations are still relatively expensive, and (3) having all processors on the system updating a single pair of variables caused a great deal of cache line bouncing, which hurt performance. 

The next attempt was called "sloppy_rss." Essentially, the sloppy approach retains the old `unsigned long` type for `rss` and `anon_rss`, and simply updates them without the lock. The result is incorrect RSS values, but Christoph noted that the errors tended not to exceed 1%. This approach is faster than using atomic operations. The incorrect values bugged some developers, however, and the cache bouncing problem remained. 

Another approach which was to do away with the RSS counters entirely, on the theory that these values were not actually needed very often. When an attempt to query the resident set size was made (generally by reading files in `/proc` from user space), the kernel would scan through the process's page tables and count the number of resident pages. This idea did not get very far, however; the cost of querying RSS values was simply too high. 

The current approach was [suggested by Linus](<https://lwn.net/Articles/114612/>) last month. A new set of counters is added to the task structure; when a thread brings a page into memory, that thread's counters are incremented accordingly. When a real RSS value is needed, the per-thread values are summed to yield the answer. So querying the RSS still requires a loop, but iterating through a list of tasks is much faster than walking an entire set of page tables. This algorithm avoids locking issues (since each thread takes care of its own page fault accounting and does not contend with others); it also minimizes the cache line problems. The "split RSS" approach still requires `rss` and `anon_rss` counters in the `mm_struct` itself; they are used to track pages brought in by threads which have since exited, and they are decremented when pages are forced out. This change also requires that RCU be used when freeing the `mm_struct` structure to ensure that no other processor is still trying to calculate an RSS value. 

The current version of the patch has [convinced Linus](<https://lwn.net/Articles/114614/>), so expect it to go in at some point. The biggest roadblock, at this point, may be that the four-level page table patch is at the front of the queue for 2.6.11. That patch currently conflicts with Christoph's work, and, in general, has made it hard for other VM work to get done. Once the four-level patch goes into the mainline, however, things should stabilize somewhat - at least, from the point of view of hackers working on other VM-related patches.
