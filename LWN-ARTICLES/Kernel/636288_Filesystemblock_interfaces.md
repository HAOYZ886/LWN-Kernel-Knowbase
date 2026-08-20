---
title: Filesystem/block interfaces
url: https://lwn.net/Articles/636288/
date: "March 17, 2015"
category: Block layer
author: "By Jake Edge March 17, 2015 LSFMM 2015"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jake Edge**  
March 17, 2015

* * *

[LSFMM 2015](<https://lwn.net/Articles/lsfmm2015/>)

In his session at the 2015 LSFMM Summit, Steven Whitehouse wanted to try to pull together lots of individual projects that are affecting the interfaces between the filesystem and block layers. There may be certain commonalities between them, so it would be good if the projects know about each other. When looking at making interface changes, it is also important for the storage and filesystem maintainers to consider the needs of all of these related projects rather than to just look at them piecemeal. 

[ ![\[Steven Whitehouse\]](https://static.lwn.net/images/2015/lsf-whitehouse-sm.jpg) ](<https://lwn.net/Articles/636339/>)

These projects come under one of three broad headings: dynamic devices, innovative I/O, and snapshots. Dynamic devices refers to "intelligent storage" devices; normally, a block device has the same characteristics throughout its life, but dynamic devices change capacity or other attributes over time. Innovative I/O refers to working with devices like shingled magnetic recording (SMR) and persistent memory devices as well as supporting data integrity features like checksums. Snapshots could fit in either of the other two headings, but he thought it was best to pull them out on their own. 

Dynamic devices are those that have changes made to the device post-mount. For example, thin provisioning changes the capacity in the underlying devices in response to less available disk space—up to the capacity the kernel believes that it has. But dynamic devices may require a different kind of interface for error reporting so that filesystems can distinguish between temporary and permanent errors. Topology changes for multipath devices are another dynamic change. If Btrfs exeriences checksum failures while trying to read data, it may want to be able to ask for a different mirror or to change the path to the data. He asked, what information is needed from the block layer and how do the filesystems get that information? 

There is a difference between informational reporting and error reporting, James Bottomley said. One contains hints that filesystems might want to use, while the other means the filesystem needs to do something about the event. Another question is how applications would want to get that kind of information, Ted Ts'o said, though it is clear that most applications won't change to take advantage of this kind of information. 

Hannes Reineke said that there have been some attempts to use udev notifications to provide information to user space. The problem with that is there is no device information available for udev to attach the information to. Even if the information is available, there needs to be a way to transport it, he said. 

But it is the filesystems that really need to know about changes in the block layer, Ts'o said. Maybe there needs to be a callback added to `struct super` that the block layer can make use of to alert filesystems to changes. Even a simple "something changed" message would be helpful. 

There are a variety of new features that require different ways to communicate between the filesystems and the block layer, Whitehouse said in transitioning to the innovative I/O topic. SMR devices need to provide ways for the filesystem to find out where the write pointer is and the layout of the zones in the device. Data integrity (e.g. [DIF/DIX](<https://access.redhat.com/solutions/41548>)) requires ways for checksums and/or checksum failures to be communicated between the block and filesystem layers. If the filesystem wants to read from a specific disk in a mirror, to provide hints to the block layer, or to initiate a copy offload operation, there needs to be an interface available to do so. He wondered if the same sorts of mechanisms could be used to support all of these kinds of operations. 

The short answer would seem to be "no". Ts'o said that there are too many differences for all of those to be able to share much. But too much specificity in the interfaces won't be good either, Ric Wheeler said. Sometimes the right thing to request is for the block layer to "do something different than you did last time" when there is an problem, he continued. Christoph Hellwig agreed that "try again" can be the right approach for both disk failures and transport failures, while Dave Chinner suggested that adding some kind of "retry as hard as you can" operation might be helpful. 

The problem comes back to error reporting and distinguishing transient from permanent errors, which is a recurring topic in the storage and filesystems tracks at LSFMM. The kernel is currently limited to the POSIX-defined errors, Chinner said. What is really needed are more fine-grained errors that give more information than just `ENOSPC`. A proper error interface from the block layer is really needed, he said. 

Getting consistency between the snapshot operations across various devices was Whitehouse's last topic. Trying to take a filesystem snapshot on a single device is much different than doing so on a thin-provisioned array that may involve multiple underlying block devices. There are different granularities for snapshots as well. It could be that a single-file snapshot or application snapshot (which might include files on multiple filesystems) is desired. 

For this topic, though, there was little time for discussion. Whitehouse was able to at least introduce the problem a bit for consideration down the road. 

[I would like to thank the Linux Foundation for travel support to Boston for the summit.]
