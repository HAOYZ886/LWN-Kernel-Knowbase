---
title: The end of the 3.12 merge window
url: https://lwn.net/Articles/566546/
date: "September 17, 2013"
category: "Releases-3.12"
author: "By Jonathan Corbet September 17, 2013"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
September 17, 2013

Despite [toying with](<https://lwn.net/Articles/566948/>) the idea of closing the merge window rather earlier than expected, Linus did, in the end, keep it open until September 16\. He repeated past grumbles about maintainers who send their pull requests at the very end of the merge window, though; increasingly, it seems that wise maintainers should behave as if the merge window were a single week in length. Pull requests that are sent too late run a high risk of being deferred until the next development cycle. 

In the end, 9,479 non-merge changesets were pulled into the mainline repository for the 3.12 merge window; about 1,000 of those came in after the writing of [last week's summary](<https://lwn.net/Articles/566122/>). Few of the changes merged in the final days of the merge window were hugely exciting, but there have been a number of new features and improvements. Some of the more significant, user-visible changes include: 

  * Unlike its predecessor, the 3.12 kernel will not be known as "Linux for Workgroups." Instead, for reasons that are not entirely clear, the new code name was "Suicidal Squirrel" for a few days; it then was changed to "One giant leap for frogkind." 

  * It is now possible to provide block device partition tables on the kernel command line; see [Documentation/block/cmdline-partition.txt](<https://lwn.net/Articles/566549/>) for details. 

  * The memory management subsystem has gained the ability to migrate huge pages between NUMA nodes. 

  * The Btrfs filesystem has the beginning of support for offline deduplication of data blocks. A new `ioctl()` command (`BTRFS_IOC_FILE_EXTENT_SAME`) can be used by a user-space program to inform the kernel of extents in two different files that contain the same data. The kernel will, after checking that the data is indeed the same, cause the two files to share a single copy of that data. 

  * The HFS+ filesystem now supports POSIX access control lists. 

  * The [reliable out-of-memory killer patches](<https://lwn.net/Articles/562211/#oom>) have been merged. This work should make OOM handling more robust, but it could possibly confuse user-space applications by returning "out of memory" errors in situations where such errors were not seen before. 

  * The evdev input layer has gained a new `EVIOCREVOKE` `ioctl()` command that revokes all access to a given file descriptor. It can be used to ensure that no evil processes lurk on an input device across sessions. See [this patch](<http://lists.freedesktop.org/archives/systemd-devel/2013-August/012897.html>) for an example of how this functionality can be used. 

  * New hardware support includes: 

    * **Miscellaneous** : MOXA ART real-time clocks, Freescale i.MX SoC temperature sensors, Allwinner A10/A13 watchdog devices, Freescale PAMU I/O memory management units, TI LP8501 LED controllers, Cavium OCTEON GPIO controllers, and Mediatek/Ralink RT3883 PCI controllers, 

    * **Networking** : Intel i40e Ethernet interfaces.  

Changes visible to kernel developers include: 

  * The seqlock locking primitive has gained a new "locking reader" type. Normally, seqlocks allow for data structures to be changed while being accessed by readers; the readers are supposed to detect the change (by checking the sequence number) and retry if need be. Some types of readers cannot tolerate changes to the structure, though; in current kernels, they take an expensive write lock instead. The "locking reader" lock will block writers and other locking readers, but allow normal readers through. Note that locking readers _could_ share access with each other; the fact that this sharing does not happen now is an implementation limitation. The functions for working with this type of lock are: 

```
void read_seqlock_excl(seqlock_t *sl);
            void read_sequnlock_excl(seqlock_t *sl);
```

There are also the usual variants for blocking hardware and software interrupts; the full set can be found in `<linux/seqlock.h>`. 

  * The [new shrinker API](<https://lwn.net/Articles/550463/>) has been merged. Most code using this API needed to be changed; the result should be better performance and a better-defined, more robust API. The new "LRU list" mechanism that was a part of that patch set has also been merged. 

  * The [per-CPU IDA](<https://lwn.net/Articles/555189/>) ID allocator patch set has been merged. 

Now begins the stabilization phase for the 3.12 kernel. If the usual pattern holds, the final release can be expected on or shortly after Halloween; whether it turns out to be a "trick" or a "treat" depends on how well the testing goes between now and then.
