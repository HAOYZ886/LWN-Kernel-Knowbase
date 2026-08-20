---
title: The lockless page cache
url: https://lwn.net/Articles/291826/
date: "July 29, 2008"
category: "Lockless page cache; Memory management-Scalability; Scalability; fast gup"
author: "By Jonathan Corbet July 29, 2008"
---

By **Jonathan Corbet**  
July 29, 2008

One of the biggest problems in kernel development is dealing with concurrency. In a system where more than one thing can be happening at once, one must always take care to keep multiple threads of control from interfering with each other and corrupting the system as a whole. In the same way that two roads become more dangerous when they intersect, connecting two or more processors to the same memory greatly increases their potential for the creation of mayhem. 

Travelers to the US are often amused (or irritated) by the often-favored solution to roadway concurrency: putting in traffic lights. Such a light will indeed (if observed) eliminate the potential for a number of unpleasant race conditions within intersections, but at a performance cost: traffic going through the intersection must often stop and wait. This solution also scales poorly; as more roads (or lanes with different destinations) feed into the same intersection, each of them experiences more red-light time. 

In kernel programming, the first tool for controlling concurrency - locks in various forms - are directly analogous to traffic lights. It is not coincidental that the name for a common locking primitive (semaphore) matches the name for a traffic light (semaforo) in a number of Latin-derived languages. Locks enforce exclusive access to a kernel resource in the same way that a traffic light enforces exclusive access to an intersection, and with many of the same costs. When too many processors end up waiting at the same lock, the performance of the system as a whole can suffer significantly. 

There are two common approaches to mitigating scalability problems with locks. For many years after the 2.0 kernel came out, these problems were addressed through the creation of more locks, each controlling a smaller resource. Lock proliferation is effective, in that it reduces the chance that two processors will be trying to acquire the same lock at the same time. Since it works so well, this approach has led to the creation of thousands of locks in the Linux kernel. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

Proliferation has its limits, though. Adding locks increases complexity; in particular, with more locks, the chances of creating occasional deadlock situations increase. Deadlocks can be avoided through the careful observation of rules on the acquisition of locks, and the order in which they are acquired in particular. But nobody will ever be able to sort out \- and document - the proper relative locking order for thousands of locks. So kernel developers must make do with rules for some of the most important locks and the vigilance of the lockdep tool to find any remaining problems. 

The other problem with lock proliferation is harder to get around, though. The acquisition of a lock requires writing a value to a location in shared memory. As each processor acquires a lock, it must change that value, which causes that processor to acquire exclusive access to the cache line holding the lock variable. The cache lines for heavily-used locks will fly around the system in a way that badly hurts performance, even if no processor ever has to wait for another to release the lock. Adding more locks will not fix this problem; instead, it will just create more bouncing cache lines and make things worse. 

So, as the number of processors grows, the path to continued scalability must not include the wholesale creation of new locks; indeed, it requires the removal of locks in the most performance-critical paths. And that is what this whole long-winded introduction leads up to: the 2.6.27 kernel will include some changes by Nick Piggin which implement lockless operation in some important parts of the virtual memory subsystem. And those, in turn, will lead to faster operation on multiprocessor systems. 

The first of these changes is a new function for obtaining direct access to user-space pages from the kernel: 

```
int get_user_pages_fast(unsigned long start, int nr_pages, int write,
    			        struct page **pages);
```

This function works much like `get_user_pages()`, but, in exchange for some limits on its operation, it is able to do its job without acquiring the mmap semaphore; that, in turn, can lead to a 10% performance boost on "a threaded database workload." The details of how this function works were [covered here last March](<http://lwn.net/Articles/275808/>) (though the function was called `fast_gup()` back then), so we'll not repeat that discussion here. 

The other big change is a set of patches which Nick has been carrying for quite some time: the lockless page cache. The page cache holds in-memory copies of pages from files on disk; its purpose is to improve performance by minimizing disk I/O. Looking up pages in the page cache is a common activity; it happens as a result of file I/O, page faults, and more. So it needs to be fast. In 2.6.26 kernels, each mapping (each connection between the page cache and a specific file in a filesystem somewhere) has its own lock. So processors will not normally contend for the locks unless they are operating on the same file. But locks for commonly-accessed files (shared libraries, for example) are likely to be frequently bounced between processors. 

Most page cache operations are lookups - read-only operations which make no changes. In the lookup operation, the lock protects a few aspects of the task, including: 

  1. A given page within the mapping must be looked up in the mapping's [radix tree](<http://lwn.net/Articles/175432/>) to find its location in memory (if any). 

  2. If the page is resident in the page cache, it must have its reference count increased so that it will not be evicted before the code performing the lookup has done whatever it needs to do. 

The radix tree, itself, is a complicated data structure; it must be protected from modification while the lookup is being performed. For certain, performance-critical parts of the radix-tree code, that protection is done through (1) some rules on what can be called when, and (2) the use of read-copy-update (RCU). As a result, the radix tree lookup can be done in a lockless manner. 

There is still a problem, though: a given page may be evicted from the page cache (or simply moved) between steps (1) and (2) above. Should that happen, the second step will increment the reference count for a page which now belongs to a different mapping, and return an incorrect pointer. The kernel developers have, through lots of experience over many years, learned that system crashes resulting from data corruption are quite hard on throughput. So true scalability requires that this kind of scenario be avoided; thus the mapping semaphore, which prevents page cache changes from being made until the reference count has been properly updated. 

Nick made an interesting observation here: it actually doesn't matter if the wrong reference count gets incremented as long as one ensures that the specific page mapping is still valid afterward. The result is a new, low-level page cache function: 

```
int page_cache_get_speculative(struct page *page);
```

If the given `page` has a reference count of zero, then the page has been removed from the page cache; in that case this function return zero and the reference count will not be changed. If the reference count is non-zero, though, it will be increased and a non-zero value will be returned. 

Incrementing a page's reference count will prevent that page from being evicted or moved until the count goes back to zero. So kernel code which has incremented a specific page's reference count will thereby ensure that the page stays in its current state. In the page cache case, the code can obtain a speculative reference to a page found in a mapping's radix tree. But it does not, yet, know whether it actually got a reference to the page it was looking for - something may have happened between the radix tree lookup and the obtaining of the reference. So it must check - after the reference has been acquired - to be sure that it has the right page. If not, it releases the reference and tries again. Eventually it will either pin down the right page or verify that the relevant part of the file is not resident in memory. 

Lockless operation forces a bit more care on the part of the page reclaim code, which is trying to get a page's reference count down to zero so that it can remove the page. Since there is no locking around the reference count now, the reclaim code must set it to zero while checking, in an atomic manner, that nobody else has incremented it. That is the purpose of the `atomic_cmpxchg()` function, which will only perform the operation if it does not collide with another processor. Since `page_cache_get_speculative()` will not increment the reference count if it is zero, the reclaim code knows that, by getting that count to zero, it now has exclusive control of the page. 

The end result of all this is that a set of locking operations has been removed from the core of the page cache, improving the scalability of that code. There is, of course, a cost, in the form of trickier code with a more complex set of rules which must be followed. Chances are that we will see more of this kind of code, though, as the number of processors in our systems increases.
