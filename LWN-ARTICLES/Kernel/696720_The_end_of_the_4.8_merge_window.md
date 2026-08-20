---
title: The end of the 4.8 merge window
url: https://lwn.net/Articles/696720/
date: "August 10, 2016"
category: "Releases-4.8"
author: "By Jonathan Corbet August 10, 2016"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
August 10, 2016

By the time Linus released [4.8-rc1](<https://lwn.net/Articles/696633/>) and closed the merge window for this development cycle, 11,618 non-merge changesets had found their way into the mainline repository. That suggests that 4.8 will be a relatively busy development cycle, but not busy enough to break any records. Just over 1,000 of those changesets were pulled after [last week's summary](<https://lwn.net/Articles/696073/>) was written; some of the more interesting changes in that last set include: 

  * The Ceph filesystem now has full [RADOS namespace](<http://tracker.ceph.com/projects/ceph/wiki/Rados_namespaces>) support. This feature has been partially supported since 4.5; the final pieces were merged for 4.8. 

  * The OrangeFS filesystem has better in-kernel caching support; see [the pull-request text](<https://git.kernel.org/linus/8cbdd85bda499d028b8f128191f392d701e8e41d>) for more information. 

  * The new `printk.devkmsg` command-line parameter can be used to control the ability of user space to send data to the kernel log via `/dev/kmsg`. The default setting of `ratelimit` applies rate limiting to data from user space. Other possibilities are `on` (allowing unlimited logging, as older kernels did) and `off` to disable logging from user space entirely. 

  * M68k binaries built for systems without a memory-management unit can now be run on ordinary, MMU-equipped systems as well. That will help developers of such applications debug them on more powerful systems. 

  * The new "software RDMA over Ethernet" driver allows the use of InfiniBand remote DMA protocols over the kernel's network stack. 

  * Reverse-mapping support has been added to the XFS filesystem; this feature allows the filesystem code to track the ownership of every block on a storage device. Reverse mapping in its current form is not hugely useful, but it will be a core part of a set of intended XFS features for future development cycles; these features include [`reflink()`](<https://lwn.net/Articles/684826/>), copy-on-write data, data deduplication, much-improved bad block reporting, and better recovery from filesystem damage. As Dave Chinner [put it](<https://git.kernel.org/linus/0cbbc422d56668528f6efd1234fe908010284082>): ""There's a lot of new stuff coming along in the next couple of cycles, and it all builds in the rmap infrastructure."" 

  * The [architecture emulation containers](<https://lwn.net/Articles/679308/>) feature has been merged; it allows containers to run code built for an architecture that differs from that of the host system. 

  * The [post-init read-only memory](<https://lwn.net/Articles/666550/>) kernel-hardening feature now works with data in loadable modules as well. 

  * The [hardened usercopy patches](<https://lwn.net/Articles/695991/>) were merged after the 4.8-rc1 release. This feature adds more checking to the kernel functions that copy data between kernel and user space with the idea of making them harder to exploit. 

  * New hardware support includes: RapidIO channelized mailbox controllers, IDT RXS Gen.3 SRIO switches, IBM POWER virtual SCSI target servers, Maxim MAX6916 SPI realtime clocks, Silead I2C touchscreens, SiS 9200 family I2C touchscreens, Broadcom iProc PWM controllers, STMPE expander PWM controllers, ChromeOS EC PWM controllers, and J-Core J2 processors. 

One thing that did not make it this time around, despite being pushed during the merge window, is the "latent entropy" GCC plugin. This program instruments various kernel functions in an attempt to generate some entropy from randomness in how the hardware responds, especially during that period early in the boot process when entropy may be in short supply. Linus was [unimpressed](<https://lwn.net/Articles/696938/>) by the pull request and unconvinced by the techniques used in the plugin itself. He has [indicated](<https://lwn.net/Articles/696939/>) that he might eventually take the plugin, but not right away, so this one looks like it will wait until the 4.9 development cycle. 

If the usual schedule holds, the final 4.8 release will come out on September 25, which will place the 4.9 merge window during the Kernel Recipes and LinuxCon Europe conferences. That will thus be a busy time, but, between now and then, the work of testing this kernel and fixing the bugs needs to be done.
