---
title: "4K stacks - again"
url: https://lwn.net/Articles/160138/
date: "November 15, 2005"
category: Kernel stack; NDISwrapper
author: 
---

The question of whether the i386 architecture should move to using 4K kernel stacks by default has been raised a few times; LWN last [covered the 4K stack issue](<http://lwn.net/Articles/150580/>) in September. Adrian Bunk has started the discussion anew with [this proposal](<http://lwn.net/Articles/159934/>) that the -mm tree go to 4K stacks (only) now, with an eye toward changing the mainline for 2.6.16. 

Most of the technical issues have not changed since September, so those arguments will not be repeated here. It is worth noting that layered block devices and filesystems have mostly been fixed. In past kernels, highly stacked devices (think of a combination of RAID, encryption, and network filesystems) could end up with very long call chains in the kernel, and, as a result, overflow the kernel stack. Most of these calls have since been serialized, so block-layer stacking should not be a problem. 

The issue that remains is NDISwrapper, the glue layer which allows Windows NDIS drivers to be loaded into a Linux kernel. Windows runs with a much larger kernel stack size, so NDIS driver writers have no reason to be as careful about stack usage. And, of course, these drivers cannot be fixed to work properly with Linux. Some have argued that breaking NDISwrapper is not a possibility, since many users rely upon it to make their wireless network adapters work. But patience with this line of thought is running thin, as can be seen in [this outburst](<https://lwn.net/Articles/160146/>) from Dave Jones: 

If we continue down this path, we'll have no native wireless drivers for Linux. The answer is not to complain to linux-kernel for breaking ndiswrapper, but complain to the vendors for not releasing specifications for native drivers to be written. 

The good news is that [the wireless situation is not as bad as one might think](<https://lwn.net/Articles/160147/>). There is [documentation for Broadcom chips](<http://bcm-specs.sipsolutions.net/>) available now, and [a Broadcom driver](<http://bcm43xx.berlios.de/>) is in the works. There is also [an Atheros driver](<http://www.selenic.com/pipermail/kernel-mentors/2005-August/000347.html>) which is "nearly done." Once these drivers are complete and joined with the Intel drivers already in the mainline, Linux will have much better support for wireless devices, and far fewer systems will have any reason to use NDISwrapper. 

There are a number of reasons for going with the 4K stack mode, including better performance and higher reliability. Some distributions (e.g. Fedora Core and RHEL) have been shipping 4K kernels for a while now. So, while nobody has committed to moving the mainline (or -mm) toward 4K-only yet, chances are improving that it will happen sometime in the not-too-distant future.
