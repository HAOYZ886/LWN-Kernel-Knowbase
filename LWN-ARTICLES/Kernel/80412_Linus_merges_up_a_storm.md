---
title: Linus merges up a storm
url: https://lwn.net/Articles/80412/
date: "April 14, 2004"
category: Development model
author: 
---

While Linus took a week off, Andrew Morton maintained a "merge candidate" tree full of patches which were to be added to the mainline on Linus's return. Linus is back; he has been quiet on linux-kernel, but his BitKeeper repository shows that he has been busy: over 700 patches have been merged in the first half of this week. Quite a few of these are significant; there will be a lot of changes in the 2.6.6 kernel. Here's a quick list of some of the more important additions. 

  * The usual pile of architecture updates, including x86_64, PPC, ARM, ia64, m68k-noMMU, S/390, and others. 

  * [POSIX message queue](<http://www-users.mat.uni.torun.pl/~wrona/posix_ipc/>) support. 

  * Changes to the ext2 and ext3 filesystems which provide significant speedups for the `fsync()` and `fdatasync()` calls. Various other performance improvements have been added to those filesystems as well. 

  * The addition of the `fcntl()` method to the `file_operations` structure (see the [March 24 Kernel Page](<https://lwn.net/Articles/76338/>)). 

  * The "laptop mode" patch. This patch has evolved somewhat since we last [looked at it](<http://lwn.net/Articles/65437/>), but the basic idea remains the same: avoid spinning up the disk whenever possible, but, when you do have to perform disk activity, do everything you can. 

  * 4KB kernel stacks for the i386 architecture. This patch reduces the kernel's per-process overhead, which is useful for people trying to run thousands of threads. It also removes one of the few places where the kernel needs to allocate multiple, physically-contiguous pages. In 2.6.6, there is a configuration option allowing the continued use of 8KB stacks, though the plan is to eventually remove this option. The configured stack size is stored in modules, so it will not be possible to load a module which was built for the wrong size stack. 

  * Non-executable stack support for several architectures. This is not the full "Exec shield" patch from Ingo Molnar, though parts of that patch appear here. 

  * A big reiserfs update, including `data=ordered` support, space preallocation, laptop mode support, and more. 

  * IPv6 support in SELinux. 

  * The [lightweight auditing framework](<https://lwn.net/Articles/79326/>). 

  * A mechanism which allows block drivers to respond to queries about the congestion state of their queues. This is useful for higher-level drivers (i.e. the device mapper) which have a complicated queue state. 

  * The [per-device unplugging patch](<https://lwn.net/Articles/75233/>) which makes some significant changes to the block layer, but which yields significant performance improvements. This patch has evolved a lot since it was originally posted, mostly to deal with complexities in the device mapper, RAID, and swapping code. 

  * The "completely fair queueing" (CFQ) I/O scheduler (covered here [last November](<https://lwn.net/Articles/57732/>)). This scheduler tries to evenly divide disk bandwidth among all processes on the system. The CFQ scheduler can be chosen with a configuration option, or by booting with the `elevator="cfq"` option. 

  * Some software suspend fixes, including support for systems with high memory. 

  * The external module support patch (described in a separate article below). The behavior of "`make clean`" has also been reworked to do a more thorough job while, simultaneously, leaving behind enough information to allow the building of external modules. 

  * A new configuration option allowing the building of kernels without sysfs support. Be sure to read the help text before disabling sysfs, however; without sysfs the kernel needs more explicit help in finding its root partition. 

  * Various libata (serial ATA) improvements and fixes. 

  * A long list of NFS cleanups and improvements. 

  * Some cosmetic fixes, such as running devfs and the floppy driver through `lindent`. 

  * Some significant page cache and virtual memory changes, which we will get to in the next article. 

Overall, one might be forgiven for thinking that 2.6.6 looks much like a development kernel release. In fact, most of more intrusive patches listed above have been around and tested for some time now; they have just finally made their escape from the -mm tree. With the exception of the CPU scheduler patches (which we hope to cover here next week) and, perhaps, the reverse mapping VM changes, 2.6.6 looks likely to contain the bulk of the work that most developers are still hoping to see added to 2.6. 2.6.6 contains enough big changes that its chances of containing an unpleasant surprise or two are fairly high. Within a few more releases, however, 2.6 may well have stabilized to the point that it can be more widely deployed and the bulk of developer attention can move on to 2.7.
