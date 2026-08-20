---
title: "Write-protect for userfaultfd()"
url: https://lwn.net/Articles/787308/
date: "May 2, 2019"
category: userfaultfd
author: "By Jonathan Corbet May 2, 2019 LSFMM"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
May 2, 2019

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2019/>)

The [`userfaultfd()`](<http://man7.org/linux/man-pages/man2/userfaultfd.2.html>) system call allows one process to handle page faults for another — in user space. Its original use case was to support transparent container migration, but other uses have developed over the years. At the 2019 Linux Storage, Filesystem, and Memory-Management Summit, Andrea Arcangeli described a scheme to [add write-protection support](<https://lwn.net/Articles/783571/>) to `userfaultfd()`. After a year of lost time fighting speculative-execution problems, Arcangeli is about ready to move this feature into the mainline. 

The core idea behind `userfaultfd()` is that, when the process of interest incurs a page fault due to a non-present page, the monitoring process receives a notification and is given the opportunity to provide a page to satisfy that fault. Once pages are present, though, the monitoring process is no longer involved. The new proposal changes that by allowing the monitoring process to mark pages as being write-protected, even though the owning process has write permission to them. That allows writes to be intercepted, and some extra processing performed. 

There are a number of use cases for this feature, Arcangeli said. They include: 

  * Live snapshotting of running processes. QEMU would like this feature so that it can be sure of copying pages in a stable state. 
  * The Redis database system performs checkpointing by forking, then writing out its memory in the child process, taking advantage of the kernel's copy-on-write behavior to keep the memory stable while it is being written. Among other things, this technique requires disabling transparent huge pages in order to work, but that has a performance impact. It also potentially doubles memory use, since the parent could write to all of its data pages while the child is running. The write-protect feature could eliminate the need to fork to get a stable set of pages to write. 
  * There are schemes for implementing distributed shared memory that could use it to detect (and distribute) changes. This can, in theory, be done now by using `mprotect()` and handling segmentation-violation signals, but it's slow and creates vast numbers of virtual memory areas in the kernel. 
  * The [soft dirty feature](<https://www.kernel.org/doc/Documentation/vm/soft-dirty.txt>), used by the checkpoint-restore in user space (CRIU) mechanism, could be replaced by this write-protect mechanism. It would be far more efficient, since it would eliminate the need to scan through the page tables. 
  * Similarly, it could be used to replace the dirty bits used by language runtime systems, such as the Java virtual machine, to catch changes. 

Many other cases that use `mprotect()` now would likely run faster with `userfaultfd()`, which never needs to acquire the highly contended `mmap_sem` lock for writes. 

Processes wanting to use this feature must call `userfaultfd()` to obtain a file descriptor, then perform an `UFFDIO_REGISTER` `ioctl()` call with the `UFFDIO_REGISTER_MODE_WP` option. Then `UFFDIO_WRITEPROTECT` operations, which supply a start address [![\[Andrea
Arcangeli\]](https://static.lwn.net/images/conf/2019/lsfmm/AndreaArcangeli-sm.jpg)](<https://lwn.net/Articles/787316/>) and a size, can be used to change the write-protect bit on one or more pages. Events will show up with the `UFFD_PAGEFAULT_FLAG_WP` flag set. For now, write protection only works with anonymous pages. 

There are some loose ends that still need to be worked out. For example, the write-protect feature requires a new bit in the page-table entries; it has taken the last available bit for this purpose, which may not be entirely welcome. Arcangeli said that it may be possible to find an alternative to using that bit later on. Dave Hansen said that there might just be a couple of other bits available if one looks hard enough, but Mel Gorman warned that they might not be available on all architectures. Some of the apparently unused bits have been claimed by CPU vendors for uses that have not yet been made public, it seems. 

Another issue is that page-fault handlers can fail with a `VM_FAULT_RETRY` return code, indicating that the memory-management subsystem should restart the process of handling the fault from the beginning. But that can only happen twice for any given fault before the whole thing fails, sending a (probably fatal) signal to the faulting process. This is a problem for the write-protect feature, which can generate more retries than that. Ideally, an unlimited number of retries would be allowed, Arcangeli said. 

Currently privilege is required to use `userfaultfd()`, but the desire is to make it available for unprivileged use as well; a sysctl knob would be used to control whether privilege is needed or not, allowing "paranoid" system administrators to restrict its use. Jérôme Glisse suggested that perhaps the seccomp mechanism would be a better way to control access to this feature. 

At the end of the session, Arcangeli asked whether the patches need more review before being merged. "Always" was the inevitable answer from memory-management subsystem maintainer Andrew Morton.
