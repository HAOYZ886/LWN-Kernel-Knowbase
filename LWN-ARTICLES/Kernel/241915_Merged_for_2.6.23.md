---
title: Merged for 2.6.23
url: https://lwn.net/Articles/241915/
date: "July 18, 2007"
category: "Releases-2.6.23"
author: 
---

Some 2600 changesets have been merged into the mainline kernel repository since [last week's summary](<http://lwn.net/Articles/241311/>). The shape of 2.6.23 is now becoming clearer; this kernel will include: 

  * New drivers for Dallas DS1682 elapsed time recorder chips, PMC-Sierra MSP71xx i2c controllers, Renesas M66592 USB peripheral controllers, Renesas R8A66597 USB host controllers, OTi-6858 USB-to-RS232 bridge controllers, Samsung S3C24xx SoC USB device controllers, Intel iop32x, iop33x, and iop13xx DMA engines, Xilinx SystemACE compact flash interfaces, BCM1250 dual UART devices, OMAP24xx multichannel SPI controllers, Atmel AVR32 AT32AP700x real-time clocks, ST M41T80 and ST M48T59 real-time clocks, Dallas DS1216 real-time clocks, TI OMAP framebuffers, display controllers, and LCD controllers (along with a support for a number of panels), Atmel AT32AP700X watchdog devices, IBM z/VM virtual card readers and punches, Afatech AF9005 demodulators. 

  * After years of work, the core Xen i386 implementation has been merged. Xen is finally a part of the mainline kernel. (Anybody who is tempted to believe that predictions found in LWN are worth anything may be amused by [Dave Jones poking fun](<http://kernelslacker.livejournal.com/87437.html>) at a suggestion, published in 2004, that Xen could be merged sometime soon). 

  * The [`fallocate()`](<http://lwn.net/Articles/240571/>) system call has been merged, but without the deallocation options. 

  * The developmental ext4 filesystem has gained a number of new features, including `fallocate()` support, nanosecond timestamps, and support for directories containing more than 65,000 other directories. 

  * The new "macvlan" driver allows the system administrator to create virtual interfaces mapped to and from specific MAC addresses. 

  * A number of virtual drivers for Sun logical domains (on the SPARC64 architecture) have been added. LDOM CPU hotplug support has also been added. 

  * The bsg code - a new generic SCSI device driver based on the block layer - has been merged. 

  * IPV4 multipath cached routing support has been dropped; this code never did work very well, and never got out of the experimental state. 

  * Basic, experimental support for PPP over L2TP sockets has been added. 

  * A device model extension (marked experimental) can export a laptop's desktop management information (DMI) data through sysfs. This will allow distributors to load just the drivers needed for a specific laptop instead of the "load them all and let the hardware sort them out" technique which is often used now. 

  * The highly experimental "USB persist" feature attempts to maintain the state of USB devices when they lose power. The driving motivation between this patch is to be able to suspend a system containing filesystems on USB storage and still have those filesystems mounted and working at resume time. 

  * As scheduled, the speedstep-centrino CPU governor has been removed in favor of the acpi-cpufreq code. 

  * The XFS filesystem now has a "stream of files" concept which allows it to place related files (a series of frames in a video stream, for example) contiguously on disk. 

  * The AFS filesystem now has file locking support. 

  * The raw block driver has been un-deprecated since it appears it will not be going away anytime soon. 

  * The [`O_CLOEXEC`](<http://lwn.net/Articles/236843/>) open flag has been added. 

  * There is a new `clone()` flag - `CLONE_NEWUSER` \- which creates a new user namespace for the process; it is intended for use with container systems. 

  * The long-debated [memory fragmentation avoidance](<http://lwn.net/Articles/224829/>) patches have been merged at last; the associated [lumpy reclaim](<http://lwn.net/Articles/211505/>) code has been merged as well. 

  * The kernel virtual machine (KVM) code can now support SMP guests. 

Changes visible to kernel developers include: 

  * Much of the x86 startup code has been rewritten in C. There should be little in the way of changes for anybody who does not actually get into the code, but, for those folks, the new version should be much easier to work with. 

  * There is a new rtnetlink API for managing software network devices. 

  * The networking core can now work with devices which have more than one transmit queue. This is a feature which was needed to properly support some wireless devices. 

  * The sysfs core has been significantly rewritten to weaken the connection between sysfs entries and internal kobjects. The new code should make life easier for driver writers who will have fewer object lifecycle issues to worry about. 

  * The never-used `enable_wake()` PCI driver method has been removed. 

  * Drivers wanting to get the revision ID from the PCI config space should now just use the value found in the new `revision` member of the `pci_dev` structure. All in-tree drivers have been changed to use this new approach. 

  * The SCSI layer has picked up a couple of scatter/gather accessor functions - `scsi_dma_map()` and `scsi_dma_unmap()` \- in preparation for chained scatter/gather lists and bidirectional requests. Most drivers in the kernel have been updated to use these functions. 

  * The idr code has a couple of new helper functions: `idr_for_each()` and `idr_remove_all()`. 

  * Much of the kernel build system has been converted over to "menuconfig" objects, making it easy to turn whole groups of options on or off at once. 

  * `sys_ioctl()` is no longer exported to modules. 

  * The page table helper functions `ptep_establish()`, `ptep_test_and_clear_dirty()` and `ptep_clear_flush_dirty()` have been removed - they had no in-kernel users. 

  * Kernel threads are non-freezable by default; any kernel thread which should be frozen for a suspend-to-disk operation must now call `set_freezable()` to arrange for that to happen. 

  * The SLUB allocator is now the default. 

  * The new function `is_owner_or_cap(inode)` tests for access permission based on the current fsuid and capabilities; it replaces the open-coded test previously found in several filesystems. 

  * There is a new utility function: 

```
char *kstrndup(const char *s, size_t max, gfp_t gfp);
```

This function duplicates a string along the lines of the user-space `strndup()`. 

It's worth noting a couple of things which will _not_ be in 2.6.23. The first is the [process containers](<http://lwn.net/Articles/236038/>) patch, which is not quite considered to be ready yet. Some other features (notably CFS group scheduling) are waiting for process containers, so chances are good that this code will be in shape for merging by 2.6.24. 

The other big omission is the x86_64 clockevents, dynamic tick, and high-resolution timers code. This patch is considered by its authors to be ready (and your editor has been running it without ill effect), but, after the troubles caused by the integration of the i386 version of this code in 2.6.21, there is a desire felt by some developers to go a bit more slowly and carefully. The result was a somewhat unhappy discussion on the mailing lists and a plan to better split these patches so they can be carefully reviewed for the next development cycle.
