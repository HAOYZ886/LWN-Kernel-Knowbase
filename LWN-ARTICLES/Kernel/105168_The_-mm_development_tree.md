---
title: "The -mm development tree"
url: https://lwn.net/Articles/105168/
date: "October 5, 2004"
category: Development model
author: 
---

Andrew Morton's -mm kernel tree now fills the role which might have once been taken by an odd-numbered development series. We don't have 2.7.x; instead, new stuff finds its way into 2.6.x-mm. So it can be interesting to step back, occasionally, and look at what patches are lurking there. 

2.6.9-rc3-mm2 contains a full 1213 patches. About half of these come from trees managed by various subsystem maintainers; seeing what those are usually requires pulling a separate BitKeeper tree and looking inside. These trees hold patches which are usually (usually!) relatively small and maintenance-oriented. The external trees brought into -mm currently include those dedicated to the ACPI, AGPGART, ALSA, i2c, IDE, IEEE 1394, input, serial ATA, networking, NTFS, driver core, PCI, USB, and SCSI subsystems. 

Among the other 654 patches in 2.6.9-rc3-mm2 are found: 

  * A change to how rlimit settings are interpreted; they become per-process settings, rather than per-thread. 

  * The [sysfs backing store patches](<https://lwn.net/Articles/69523/>) continue to languish in -mm, apparently waiting for a review from some of the core developers. 

  * Ingo Molnar's "generic IRQ subsystem" work. These patches, [posted](<https://lwn.net/Articles/104844/>) on October 2, are a big reorganization of the interrupt handling code. Over the years, much of the IRQ code had been copied from one architecture to the next, leading to a lot of duplicated functions. These patches pull the generic code out of the architecture subtrees and remove some 3000 lines of code from the kernel. 

  * Numerous kernel debugger (kgdb) patches continue to live in -mm; as always, they are unlikely to move into the mainline. 

  * They get less attention than they used to, but there are still [must-fix](<https://lwn.net/Articles/105171/>) and [should-fix](<https://lwn.net/Articles/105172/>) lists in -mm. 

  * Arjan van de Ven's patch which keeps processes from being able to overwrite kernel memory via `/dev/mem`. This patch has been shipped with Red Hat/Fedora kernels for a while, but is not yet in the mainline. 

  * An extensive set of ext3 patches implementing [block reservations](<https://lwn.net/Articles/81357/>). Stephen Tweedie has recently resumed working on these patches, so they may move forward in the near future. The ext3 online resizing patch set is also in -mm. 

  * Mikael Pettersson's performance counters patches. 

  * The -mm tree continues to be a testing ground for scheduler patches. It currently contains Peter Williams's Single Priority Array scheduler ([covered briefly](<https://lwn.net/Articles/96554/>) here last August). There is also an extensive set of scheduling domains fixes and a number of latency-reduction patches from Ingo Molnar's work. 

  * Ingo Molnar's [big kernel semaphore](<https://lwn.net/Articles/102253/>) patch. 

  * A set of PCMCIA patches adding driver model and hotplug support. 

  * A big DVD+RW support patch, which includes CDRW packet writing support. 

  * Support for in-kernel keyrings and their management. 

  * The [CacheFS](<https://lwn.net/Articles/100321/>) filesystem. 

  * The kexec patches, including support for using kexec as a kernel crash dump mechanism. 

  * The reiser4 filesystem and a large number of fixes. 

  * The [modular I/O schedulers patch](<https://lwn.net/Articles/102976/>) and the reworked "completely fair queueing" scheduler. 

  * The [`remap_page_range()` change](<https://lwn.net/Articles/104333/>) to `remap_pfn_range()`. 

  * A security module implementing the BSD "secure levels" mechanism. 

Mixed in with these big patches is the usual array of architecture updates, subsystem fixes, etc. 

In other words, -mm is a big patch; it is significantly different from the mainline kernel. For some developers, it is too far removed; David Miller recently [responded](<https://lwn.net/Articles/105177/>) to a request to test networking changes in -mm this way: 

Putting the net stuff into -mm makes debugging of networking changes harder, as -mm has a ton of experimental stuff in it as well. -mm frequently makes machines unbootable, and particularly this is felt on non-x86 platforms such as sparc64 which is where I do all of my work. 

This kind of observation is not new; many developers continued to create their patches on the 2.4 kernel long after the 2.5 branch opened because 2.5 struck them as being too unstable. When one is trying to shake out bugs in new code, it is nice to minimize the number of other unrelated, disruptive changes. That said, -mm continues to be the main staging area for much of the code going into the mainline, and many developers target it specifically with their patches. Given the number of bugs found after patches go into -mm, people are clearly running it as well.
