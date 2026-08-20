---
title: "Page faults in user space: MADV_USERFAULT, remap_anon_range(), and userfaultfd()"
url: https://lwn.net/Articles/615086/
date: "October 7, 2014"
category: "Memory management-Virtualization; remap anon pages; userfaultfd"
author: "By Jonathan Corbet October 7, 2014"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
October 7, 2014

The quest for performance often seems to lead developers to want to move functionalities out of the kernel and back into user space, where a dedicated application can, in theory, make things happen more quickly. Networking functions are often handled in this way, for example. A desire to move memory-management functions into user space is somewhat less common, but, as can be seen from Andrea Arcangeli's [user-space page fault handling patch set](<https://lwn.net/Articles/614881/>), it is not unheard of. 

Page-fault handling usually requires fetching data from secondary storage and placing it in the correct place in the faulting process's address space. Why would one want to do that in user space? The primary use case here is the live migration of virtual machines running under KVM. Migration requires moving the virtual machine's memory, which can take a long time, but the owner of that machine would like to see as brief an outage as possible while the migration is happening. Preferably, the migration would not be noticeable at all. One way to approach that goal is to move the minimal amount of memory needed to represent the virtual machine on the new host. Once the machine starts running in the new location, it will certainly try to access pages which have not yet been moved. If the (user-space) virtual machine manager can catch the resulting page faults, it can prioritize the transfer of the pages the running machine actually needs. It is, in other words, a form of cross-host demand paging that makes migration happen with lower latency. 

Other uses — shared memory distributed across the network, for example — are possible as well. 

The patch set starts by adding a couple of new variants to the `get_user_pages()` function, which is charged with making user-space pages accessible to the kernel: 

```
long get_user_pages_locked(struct task_struct *tsk, struct mm_struct *mm,
    		               unsigned long start, unsigned long nr_pages,
    	    		       int write, int force, struct page **pages,
    			       int *locked);
        long get_user_pages_unlocked(struct task_struct *tsk, struct mm_struct *mm,
    				 unsigned long start, unsigned long nr_pages,
    				 int write, int force, struct page **pages);
```

The former version is intended to be called with the `mmap_sem` semaphore held. It may release that semaphore while running, in which case `*locked` will be set to zero. The second form, instead, assumes that `mmap_sem` is _not_ held. Using these functions in the kernel improves performance by allowing `mmap_sem` to be dropped while page-fault handling is in progress. That is useful even in current kernels, but, if handling of faults is going to be entrusted to user space, it will become necessary. Holding `mmap_sem` while calling out to user space would not be a recipe for happy times. 

The next step is to add the `MADV_USERFAULT` flag to the `madvise()` system call. If that flag is set on a region of memory, the kernel will no longer attempt to resolve page faults in that region. Instead, in the absence of other measures (described below), the faulting process will receive a `SIGBUS` signal. That, of course, leaves the process in the position of having to resolve the page fault on its own. A tool provided to help with that task is the new `remap_anon_pages()` system call: 

```
int remap_anon_pages(void *dest, void *src, unsigned long len,
        			 unsigned long flags);
```

This system call will take the pages holding `len` bytes starting at `src` and move them in the process's address space to the region starting at `dest`. A number of conditions must be met for this operation to succeed, starting with the fact that the full range in `dest` must currently be unmapped — `remap_anon_pages()` will not overwrite an existing page mapping. The range in `src`, instead, must all be present and mapped, and the pages cannot be shared with other processes. All of these rules exist to simplify the implementation, but also to try to catch race conditions in user-space fault handling. 

If `src` is a huge page, and `len` is a multiple of 2MB, then the full huge page(s) will be moved to `dest` without being split. 

With this mechanism in place, an application's `SIGBUS` signal handler can respond to a fault by allocating memory, filling it with the needed contents, and mapping it into the proper location with `remap_anon_pages()`. Once the signal handler returns, the page fault will be retried, but, this time, the needed memory will be in place, so application execution will continue. 

Anybody who has worked with signal handlers on Unix-like systems is probably thinking at this point that all that work does not belong in such a handler. And, indeed, signal handlers are not the way that processes are expected to deal with page-fault handling. To make life easier, Andrea adds another system call: 

```
int userfaultfd(int flags);
```

This call will return an open file descriptor which may be used to communicate with the kernel about page fault handling. The `flags` argument is mostly unused, though `O_NONBLOCK` may be provided to request non-blocking behavior. 

The first step after acquiring the file descriptor is for the application to write a 64-bit integer indicating which version of the userfault protocol it understands. The kernel will respond with the same number if the protocol is supported, `-1` otherwise. Once agreement has been reached in that area, the application can read a 64-bit address whenever a page fault occurs. It should resolve the fault, then write back two pointers indicating the range of memory which has been mapped in response to the fault. 

The idea here is that a process can dedicate a thread to page-fault handling. Whenever a fault occurs, the faulting thread will pause while the handler thread puts things in place. No `SIGBUS` signals will be delivered if `userfaultfd()` has been called. So, for the faulting thread, life just continues as usual, with the possible exception that some page faults may take longer to handle than one might expect. 

As was mentioned above, there might be multiple use cases for user-space page fault handling. What if a single application wishes to exercise more than one of those cases? To that end, the application can open more than one file descriptor with `userfaultfd()` and restrict each to a specific range of memory. That restriction is requested by writing two pointers indicating the range to be covered; the least-significant bit should be set on the start pointer. Thereafter, only faults within the given range will be directed to that file descriptor. The application must still set `MADV_USERFAULT` on the ranges in question. Multiple ranges can be set up to go to a single file descriptor, but a given range of memory can only have its faults handled by a single descriptor. 

The bulk of the commentary on the patch set has been around the `remap_anon_pages()` system call. Linus initially [wondered](<https://lwn.net/Articles/615097/>) whether `remap_anon_pages()` made more sense than `remap_file_pages()`, which he called an ""unmitigated disaster"" and which [may be removed](<https://lwn.net/Articles/597632/>) in the near future. Later he [added](<https://lwn.net/Articles/615228/>) that he would prefer an interface where the fault handler process would simply `write()` the data to the page of interest, causing it to be allocated and mapped. Andrea [responded](<https://lwn.net/Articles/615229/>) that such an interface might be possible; the handler would write the data to the `userfaultfd()` file descriptor and the kernel would handle the rest. But he worried about losing the zero-copy behavior that was carefully designed into the current interface. Linus's [answer](<https://lwn.net/Articles/615231/>) to that made it clear that he was not concerned about zero-copy behavior, which, he said, is almost never worth the cost of implementing it. 

What we may see is that the `get_user_pages()` optimizations will find their way in relatively soon, though Linus [wasn't entirely happy](<https://lwn.net/Articles/615098/>) with those either. The remaining work will take a while longer, and the end result seems unlikely to include `remap_anon_pages()`. But, given that the use case is real, a significant improvement to live migration is going to be hard to turn down in the long run.
