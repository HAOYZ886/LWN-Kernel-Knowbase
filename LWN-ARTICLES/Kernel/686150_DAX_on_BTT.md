---
title: DAX on BTT
url: https://lwn.net/Articles/686150/
date: "May 4, 2016"
category: "DAX; Memory management-Nonvolatile memory"
author: "By Jake Edge May 4, 2016 LSFMM 2016"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jake Edge**  
May 4, 2016

* * *

[LSFMM 2016](<https://lwn.net/Articles/lsfmm2016/>)

In the final plenary session of the 2016 Linux Storage, Filesystem, and Memory-Management Summit, much of the team that works on the [DAX direct-access mechanism](<https://lwn.net/Articles/610174/>) led a discussion on how DAX should interact with the [block translation table](<https://www.kernel.org/doc/Documentation/nvdimm/btt.txt>) (BTT)—a mechanism aimed at making persistent memory have the atomic sector-write properties that users expect from block devices. Dan Williams took the role of ringleader, but Matthew Wilcox, Vishal Verma, and Ross Zwisler were also on-stage to participate. 

Williams noted that Microsoft has adopted DAX for persistent memory and is even calling it DAX. Wilcox said that it was an indication that Microsoft is "listening to customers; they've changed". 

[ ![\[Matthew Wilcox, Vishal Verma, Ross Zwisler, and Dan Williams\]](https://static.lwn.net/images/2016/lsf-daxbtt-sm.jpg) ](<https://lwn.net/Articles/686127/>)

BTT is a way to put block-layer-like semantics onto persistent memory, which handles writes at a cache-line granularity (i.e. 64 bytes), so that 512-byte (sector) writes are atomic. This eliminates the problem of "sector tearing", where a power or other failure causes a partial write to a sector resulting in a mixture of old and new data—a situation that applications (or filesystems) are probably not prepared to handle. Microsoft supports DAX on both BTT and non-BTT block devices, while Linux only supports it for non-BTT devices. Williams asked: "should we follow them [Microsoft] down that rabbit hole?" 

The problem is that BTT is meant to fix a problem where persistent memory is treated like a block device, which is not what DAX is aimed at. Using BTT only for filesystem metadata might be one approach, Zwisler said. But Ric Wheeler noted that filesystems already put a lot of work into checksumming metadata, so using BTT for that would make things much slower for little or no gain. 

Jeff Moyer pointed out that sector tearing can happen on block devices like SSDs, which is not what users expect. Joel Becker suggested that something like the SCSI atomic write command could be used by filesystems or applications that are concerned about torn sectors. That command guarantees that the sector is either written in full or not at all. There is no way to "magically save applications from torn sectors" unless they take some kind of precaution, he said. 

There is a bit of a "hidden agenda" in supporting BTT, though, Williams said. Currently, the drivers are not aware of when DAX mappings are established and torn down, but that would change for BTT support. Wilcox said he has a patch series that addresses some parts of that by making the radix tree the source for that information.
