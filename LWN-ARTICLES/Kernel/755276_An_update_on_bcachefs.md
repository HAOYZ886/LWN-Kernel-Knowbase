---
title: An update on bcachefs
url: https://lwn.net/Articles/755276/
date: "May 23, 2018"
category: "Filesystems-bcachefs"
author: "By Jake Edge May 23, 2018 LSFMM"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jake Edge**  
May 23, 2018

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2018/>)

The [bcachefs filesystem](<https://bcachefs.org/>) has been under development for a number of years now; according to lead developer Kent Overstreet, it is time to start talking about getting the code upstream. He came to the 2018 Linux Storage, Filesystem, and Memory-Management Summit (LSFMM) to discuss that in a combined filesystem and storage session. Bcachefs grew out of [bcache](<https://bcache.evilpiepirate.org/>), which is a block layer cache that was merged into Linux 3.10 in mid-2013. 

Five or six years ago, when he was still at Google, creating bcachefs from bcache seemed like it would take a year and 15,000 lines of code, Overstreet said. Now, six years and 50,000 lines of code later, it is a real filesystem. It "turned out really well", he said. 

[ ![\[Kent Overstreet\]](https://static.lwn.net/images/2018/lsf-overstreet-sm.jpg) ](<https://lwn.net/Articles/755291/>)

Bcachefs is a general-purpose copy-on-write filesystem with lots of features, including checksumming for blocks, compression, encryption, multiple device support, and, of course, caching. Jens Axboe asked if there was still a clean separation between bcachefs and bcache. Overstreet said that there was; roughly 80% of the code is shared. He has taken out the bcache interfaces in his development tree because there is no need for them as bcachefs can handle all of what bcache can do (and more). 

Hannes Reinecke asked about the long-term expectation for bcache and bcachefs; will they coexist or will bcache be removed in favor of bcachefs. Overstreet said that bcache is the prototype for all of the ideas in bcachefs. As part of developing bcachefs, the B-tree code has been fleshed out and polished. Bcache was fast in most cases, but there were some corner cases where it was not; all of that has been fixed in bcachefs. 

He said that he would like get users off of bcache and onto bcachefs. The filesystem has an `fsck` available to detect and repair problems. A block layer cache does not get the same level of testing that a full filesystem does. By creating and upstreaming bcachefs, he will in some sense be turning it into a real project. 

He would prefer not have both the block layer and filesystem interfaces, since that doesn't really provide anything extra. One major disadvantage of bcache is that writes to the backing device are not copy on write so there are cache coherency issues. Bcache had ways to deal with those problems, but bcachefs simply eliminates them entirely. 

Ted Ts'o asked how many users of bcache there are; how much of a problem is it to get rid of bcache? Axboe said that there are users and a community has formed to develop and maintain it. Ts'o said he would be in favor of eliminating bcache, but if there are users of the feature, that really cannot happen. Reinecke said that SUSE supports bcache in its distributions, so it will need to be maintained for a few years. 

The on-disk format is different between bcache and bcachefs, similar to how ext2, ext3, and ext4 have evolved, Overstreet said. If he brought back the block device interfaces into bcachefs, then the filesystem could be a drop-in replacement for bcache. Ts'o noted that before ext3 and ext2 could be dropped, ext4 was able to handle the other two; if bcachefs can support the older bcache devices, the same could be done. Axboe said that perhaps an offline conversion tool could be written. Reinecke said that SUSE will still need bcache as a device for some time, but doesn't care if it is provided by the bcache code or by bcachefs. 

Amir Goldstein asked about support for reflink, but Overstreet said that bcachefs does not have that yet. It is one of the easier things on the to-do list, however. Other things on that list include erasure coding and then snapshots further out. The reflink feature uses the same design as is in XFS, he said. Dave Chinner said that reflink is a major feature to be missing from a filesystem these days. Overstreet said that he has gotten much of it working, but space accounting is not right yet. 

Chinner asked if there would be an on-disk format changes that would require "forklift upgrades". The snapshot feature will require on-disk format changes, Overstreet said, but the other features should not. There has not been a need to change the on-disk format for quite some time, which is part of why he thinks it is ready to go upstream. 

Chinner wondered where bcachefs is aimed; what are its target users? Overstreet said that the killer feature is performance. The latency tail is "really really good", he said. In tests, it has gotten 14GB/sec writes without major CPU impact and mixed read/write workloads also do well. On every workload the project can find, bcachefs performs as fast as the hardware should go. 

Both small and large users will benefit from the filesystem, he said. He has been using it as his root filesystem for several years, there are users running it on servers, and the company that is funding him to work on bcachefs is using it on NAS boxes with up to 60 spindles. He was asked about shingled magnetic recording (SMR) support; both bcache and bcachefs do file data allocation in terms of 1-2MB buckets, which they write to once. That should be fairly SMR-friendly, but he has not worked out how to deal with metadata on SMR devices yet. 

Ts'o wondered about the diversity of devices that had been used in the benchmarking; that would be useful in determining what the strengths and weaknesses of bcachefs are. Has it been tried on older hardware, low-end flash devices, small disks, etc.? From what he has heard, it "starts to sound like snake oil". It has been tested on big RAID devices, high-end NVMe devices, and various other options, but has not been tested on some of the lower-end devices that were asked about, Overstreet said. 

The discussion then shifted to whether it was time to get bcachefs into the mainline and how that process would work. Axboe was concerned that the on-disk format may still change to support snapshots and wondered if it made sense to wait until that work was completed. But filesystems can support multiple on-disk formats; Btrfs does it, as Josef Bacik pointed out, and XFS has been doing it for 20 years, Chinner said. Overstreet said that filesystems using the current on-disk format would still be fully supported, just that they would not be able to take snapshots. 

Ts'o asked about xfstests and Overstreet said that he uses them all the time; there is a 30-line patch needed to support bcachefs. Once that is added, Ts'o said, he would be happy to add bcachefs to his automated testing regime. 

Bacik said that the filesystem and storage developers need to see the code and know that he will be around to maintain it, at least until there are others who will pick it up. He said that Overstreet had hit all the high points, so Bacik said he was comfortable with starting the review process. 

Overstreet said he would [post his patches](<https://lwn.net/Articles/753979/>) shortly after LSFMM, but that it is 50,000 lines of code. Chinner said that it needs to be broken up into sane chunks. Bacik agreed, saying that he mostly cared about the interfaces, not the internal B-tree stuff. Chinner said that the user-space APIs and the on-disk format were two places to start; people make "obvious mistakes" in those areas. Next would be the interface to the VFS; generally, reviewers are going to be most interested in things at the periphery. Ts'o suggested that since Overstreet knows the code best, he should highlight places where he is making assumptions about various other parts of the kernel (e.g. the dentry cache, the memory-management subsystem); that would allow reviewers to scrutinize that code.
