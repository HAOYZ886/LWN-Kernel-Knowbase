---
title: "User-space page fault handling"
url: https://lwn.net/Articles/636226/
date: "March 11, 2015"
category: Memory management; remap anon pages; userfaultfd
author: "By Jonathan Corbet March 11, 2015 LSFMM 2015"
---

By **Jonathan Corbet**  
March 11, 2015

* * *

[LSFMM 2015](<https://lwn.net/Articles/lsfmm2015/>)

Andrea Arcangeli's [`userfaultfd()`](<https://lwn.net/Articles/615086/>) patch set has been in development for a couple of years now; it has the look of one of those large memory-management changes that takes forever to find its way into the mainline. The good news in this case was announced at the beginning of this session in the memory-management track of the 2015 Linux Storage, Filesystem, and Memory Management Summit: there is now the beginning of an agreement with Linus that the patches are in reasonable shape. So we may see this code merged relatively soon. 

The `userfaultfd()` patch set, in short, allows for the handling of page faults in user space. This seemingly crazy feature was originally designed for the migration of virtual machines running under KVM. The running guest can move to a new host while leaving its memory behind, [![\[Andrea Arcangeli\]](https://static.lwn.net/images/conf/2015/lsfmm/AndreaArcangeli-sm.jpg)](<https://lwn.net/Articles/636228/>) speeding the migration. When that guest starts faulting in the missing pages, the user-space mechanism can pull them across the net and store them in the guest's address space. The result is quick migration without the need to put any sort of page-migration protocol into the kernel. 

Andrea was asked whether the kernel, rather than implementing the file-descriptor-based notification mechanism, could just use `SIGBUS` signals to indicate an access to a missing page. That will not work in this case, though. It would require massively increasing the number of virtual memory areas (VMAs) maintained in the kernel for the process, could cause system calls to fail, and doesn't handle the case of in-kernel page faults resulting from `get_user_pages()` calls. What's really needed is for a page fault to simply block the faulting process while a separate user-space process (the "monitor") is notified to deal with the issue. 

Pavel Emelyanov stood up to talk about his use case for this feature, which is the live migration of containers using the [checkpoint-restore in user space (CRIU)](<http://criu.org/Main_Page>) mechanism. While the KVM-based use case involves having the monitor running as a separate thread in the same process, the CRIU case requires that the monitor be running in a different process entirely. This can be managed by sending the file descriptor obtained from `userfaultfd()` over a socket to the monitor process. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

There are, Pavel said, a few issues that come up when `userfaultfd()` is used in this mode. The user-space fault handling doesn't follow a `fork()` (it remains attached to the parent process only), so faults in the child process will just be resolved with zero-filled pages. If the target process moves a VMA in its virtual address space with `mremap()`, the monitor will see the new virtual addresses and be confused by them. And, after a fork, existing memory goes into the copy-on-write mode, making it impossible to populate pages in both processes. The conversation did not really get into possible solutions for these problems, though. 

Andrea talked a bit about the `userfaultfd()` API, which has evolved in the past months. There is now a set of `ioctl()` calls for performing the requisite operations. The `UFFDIO_REGISTER` call is used to tell the kernel about a range of virtual addresses for which faults will be handled in user space. Currently the system only deals with page-not-present faults. There are plans, though, to deal with write-protect faults as well. That would enable the tracking of dirtied pages which, in turn, would allow live snapshotting of processes or the active migration of pages back to a "memory node" elsewhere on the network. 

With regard to the potential live-snapshotting feature, most of the needed mechanism is already there. There is one little problem in that, should the target modify a page that is currently resident on the swap device, the resulting swap-in fault will make the page writable. So `userfaultfd()` will miss the write operation and the page will not be copied. Some changes to the swap code will be needed to add a write-protect bit to swap entries before this feature will work properly. 

Earlier versions of the patch introduced a `remap_anon_pages()` system call that would be used to slot new pages into the target process's address space. In the current version, that operation has been turned into another `ioctl()` operation. Actually, there is more than one; there are now options to either copy a page into the target process or to remap the page directly. Zero-copy operation has a certain naive appeal, but it turns out that the associated translation lookaside buffer (TLB) flush is more expensive than simply copying the data. So the remap option is of limited use and unlikely to make it upstream. 

Andrew Lutomirski worried that this feature was adding "weird semantics" to memory management. Might it be better, he said, to set up `userfaultfd()` as a sort of device that could then be mapped into memory with `mmap()`? That would isolate the special-case code and not change how "normal memory" behaves. The problem is that doing things this way would cause the affected memory range to lose access to many other useful memory-management features, including swapping, transparent huge pages, and more. It would, Pavel said, put "weird VMAs" into a process that really just "wants to live its own life" after migration. 

As the discussion headed toward a close, Andrea suggested that `userfaultfd()` could perhaps be used to implement the long-requested "[volatile ranges](<https://lwn.net/Kernel/Index/#Volatile_ranges>)" feature. First, though, there is a need to finalize the API for this feature and get it merged; it is currently blocking the addition of the post-copy migration feature to KVM.
