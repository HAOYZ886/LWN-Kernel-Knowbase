---
title: A survey of memory management patches
url: https://lwn.net/Articles/562211/
date: "August 6, 2013"
category: Huge pages; Memory management; OOM killer; drop caches
author: "By Jonathan Corbet August 6, 2013"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
August 6, 2013

Traffic on the kernel mailing lists often seems to follow a particular theme. At the moment, one of those themes is memory management. What follows is an overview of these patches, hopefully giving an idea of what the memory management developers are up to. 

#### MADV_WILLWRITE

Normally, developers expect that a write to file-backed memory will execute quickly. That data must eventually find its way back to persistent storage, but the kernel usually handles that in the background while the application continues running. Andy Lutomirski has discovered that things don't always work that way, though. In particular, if the memory is backed by a file that has never been written (even if it has been extended to the requisite size with `fallocate()`), the first write to each page of that memory can be quite slow, due to the filesystem's need to allocate on-disk blocks, mark the block as being initialized, and otherwise get ready to accept the data. If (as is the case with Andy's application) there is a need to write multiple gigabytes of data, the slowdown can be considerable. 

One way to work around this problem is to write throwaway data to that memory before getting into the time-sensitive part of the application, essentially forcing the kernel to prepare the backing store. That approach works, but at the cost of writing large amounts of useless data to disk; it might be nice to have something a bit more elegant than that. 

Andy's [answer](<https://lwn.net/Articles/562213/>) is to add a new operation, `MADV_WILLWRITE`, to the `madvise()` system call. Within the kernel, that call is passed to a new `vm_operations_struct` operation: 

```
long (*willwrite)(struct vm_area_struct *vma, unsigned long start, 
    		      unsigned long end);
```

In the current implementation, only the ext4 filesystem provides support for this operation; it responds by reserving blocks so that the upcoming write can complete quickly. Andy notes that there is a lot more that could be done to fully prepare for an upcoming write, including performing the copy-on-write needed for private mappings, actually allocating pages of memory, and so on. For the time being, though, the patch is intended as a proof of concept and a request for comments. 

#### Controlling transparent huge pages

The [transparent huge pages](<https://lwn.net/Articles/423584/>) feature uses huge pages whenever possible, and without user-space awareness, in order to improve memory access performance. Most of the time the result is faster execution, but there are some workloads that can perform worse when transparent huge pages are enabled. The feature can be turned off globally, but what about situations where some applications benefit while others do not? 

Alex Thorlton's [answer](<https://lwn.net/Articles/562034/>) is to provide an option to disable transparent huge pages on a per-process basis. It takes the form of a new operation (`PR_SET_THP_DISABLED`) to the `prctl()` system call. This operation sets a flag in the `task_struct` structure; setting that flag causes the memory management system to avoid using huge pages for the associated process. And that allows the creation of mixed workloads, where some processes use transparent huge pages and others do not. 

#### Transparent huge page cache

Since their inception, transparent huge pages have only worked with anonymous memory; there is no support for file-backed (page cache) pages. For some time now, Kirill A. Shutemov has been working on a [transparent huge page cache](<https://lwn.net/Articles/562117/>) implementation to fix that problem. The latest version, a 23-patch set, shows how complex the problem is. 

In this version, Kirill's patch has a number of limitations. Unlike the anonymous page implementation, the transparent huge page cache code is unable to create huge pages by coalescing small pages. It also, crucially, is unable to create huge pages in response to page faults, so it does not currently work well with files mapped into a process's address space; that problem is slated to be fixed in a future patch set. The current implementation only works with the ramfs filesystem — not, perhaps, the filesystem that users were clamoring for most loudly. But the [ramfs implementation](<https://lwn.net/Articles/562214/>) is a good proof of concept; it also shows that, with the appropriate infrastructure in place, the amount of filesystem-specific code needed to support huge pages in the page cache is relatively small. 

One thing that is still missing is a good set of benchmark results showing that the transparent huge page cache speeds things up. Since this is primarily a performance-oriented patch set, such results are important. The `mmap()` implementation is also important, but the patch set is already a large chunk of code in its current form. 

#### Reliable out-of-memory handling

As was described in [this June 2013 article](<https://lwn.net/Articles/552789/>), the kernel's out-of-memory (OOM) killer has some inherent reliability problems. A process may have called deeply into the kernel by the time it encounters an OOM condition; when that happens, it is put on hold while the kernel tries to make some memory available. That process may be holding no end of locks, possibly including locks needed to enable a process hit by the OOM killer to exit and release its memory; that means that deadlocks are relatively likely once the system goes into an OOM state. 

Johannes Weiner has posted [a set of patches](<https://lwn.net/Articles/562091/>) aimed at improving this situation. Following a bunch of cleanup work, these patches make two fundamental changes to how OOM conditions are handled in the kernel. The first of those is perhaps the most visible: it causes the kernel to avoid calling the OOM killer altogether for most memory allocation failures. In particular, if the allocation is being made in response to a system call, the kernel will just cause the system call to fail with an `ENOMEM` error rather than trying to find a process to kill. That may cause system call failures to happen more often and in different contexts than they used to. But, naturally, that will not be a problem since all user-space code diligently checks the return status of every system call and responds with well-tested error-handling code when things go wrong. 

The other change happens more deeply within the kernel. When a process incurs a page fault, the kernel really only has two choices: it must either provide a valid page at the faulting address or kill the process in question. So the OOM killer will still be invoked in response to memory shortages encountered when trying to handle a page fault. But the code has been reworked somewhat; rather than wait for the OOM killer deep within the page fault handling code, the kernel drops back out and releases all locks first. Once the OOM killer has done its thing, the page fault is restarted from the beginning. This approach should ensure reliable page fault handling while avoiding the locking problems that plague the OOM killer now. 

#### Logging drop_caches

Writing to the magic sysctl file `/proc/sys/vm/drop_caches` will cause the kernel to forget about all clean objects in the page, dentry, and inode caches. That is not normally something one would want to do; those caches are maintained to improve the performance of the system. But clearing the caches can be useful for memory management testing and for the production of reproducible filesystem benchmarks. Thus, `drop_caches` exists primarily as a debugging and testing tool. 

It seems, though, that some system administrators have put writes to `drop_caches` into various scripts over the years in the belief that it somehow helps performance. Instead, they often end up creating performance problems that would not otherwise be there. Michal Hocko, it seems, has gotten a little tired of tracking down this kind of problem, so he has [revived an old patch from Dave Hansen](<https://lwn.net/Articles/562215/>) that causes a message to be logged whenever `drop_caches` is used. He said: 

I am bringing the patch up again because this has proved being really helpful when chasing strange performance issues which (surprise surprise) turn out to be related to artificially dropped caches done because the admin thinks this would help... So mostly those who support machines which are not in their hands would benefit from such a change. 

As always, the simplest patches cause the most discussion. In this case, a number of developers expressed concern that administrators would not welcome the additional log noise, especially if they are using `drop_caches` frequently. But Dave [expressed a hope](<https://lwn.net/Articles/562216/>) that at least some of the affected users would get in contact with the kernel developers and explain why they feel the need to use `drop_caches` frequently. If it is being used to paper over memory management bugs, the thinking goes, it would be better to fix those bugs directly. 

In the end, if this patch is merged, it is likely to include an option (the value written to `drop_caches` is already a bitmask) to suppress the log message. That led to another discussion on exactly which bit should be used, or whether the `drop_caches` interface should be augmented to understand keywords instead. As of this writing, the simple `printk()` statement still has not been added; perhaps more discussion is required.
