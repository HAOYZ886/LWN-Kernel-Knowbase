---
title: A btrfs update at LinuxCon Europe
url: https://lwn.net/Articles/465160/
date: "November 2, 2011"
category: "Btrfs; Filesystems-Btrfs"
author: "By Jonathan Corbet November 2, 2011"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
November 2, 2011

In October, the btrfs user community [expressed concerns](<https://lwn.net/Articles/462543/>) about the still missing-in-action filesystem checker and repair tool. At that time, btrfs creator Chris Mason said that he hoped to demonstrate a working checker during his LinuxCon Europe session. Your editor was there as part of a standing-room-only crowd ready to see the show; we did indeed get a demonstration, but it may not have been quite what some attendees expected. 

Chris started by talking about btrfs and its goals in general; those have been well covered here and need not be repeated now. He reiterated Oracle's plan to use btrfs as the core filesystem for its RHEL-derivative Linux distribution; needless to say, supporting that role requires a rock-solid implementation. So a lot of work has been going into extensive testing of the filesystem and fixing bugs. 

The 3.2 kernel release will see the results of that work; it will contain lots of fixes. There will also be significant improvements to the logging [![\[Chris Mason\]](https://static.lwn.net/images/conf/2011/lc-europe/ChrisMason-sm.jpg)](<https://lwn.net/Articles/465180/>) code. It turns out that a lot of data was being logged more than once, greatly increasing the amount of I/O required; that has now been fixed. I/O traffic for the log, it seems, has been cut to about 25% of its previous level. 

For 3.3, the main improvement seems to be the use of larger blocks for nodes in the filesystem B-tree. Larger blocks can hold more data, of course, and, in particular, more metadata. That means that metadata that was previously scattered in the filesystem can be kept together with the relevant inode. That, in turn, leads to significant performance improvements for many filesystem operations. 

Another near-term feature, due to arrive ""right after fsck"", is the merging of Dave Woodhouse's RAID5 and RAID6 implementations. That work was [initially posted](<https://lwn.net/Articles/341026/>) in 2009; Chris apologized for taking so long to get it merged. How this feature will actually be used still needs some thought; RAID5 or 6 is quite good for data, but it can be problematic for metadata, which tends to not fill anything close to a full RAID stripe and, thus, can lead to low I/O performance. Happily, btrfs has been designed from the beginning to keep data and metadata separate; that means that things can be set up where data is protected with full RAID while metadata is managed using simple mirroring. 

Talk of protecting metadata leads naturally to the problem of recovering a filesystem when its metadata has been corrupted. That is what a filesystem checker program is for; btrfs, thus far, has been increasingly famous for it lack of a proper checker (and, more importantly, a proper filesystem repair tool). As of the LinuxCon talk, btrfs still does not have a real repair tool, but some progress has been made in that direction and a couple of other mechanisms have been provided. 

The copy-on-write nature of btrfs implies that there will be numerous old copies of the filesystem metadata on the storage device at any given time. Any change, after all, will create a new copy, leaving the previous version in place until the block is reused. Chris observed that filesystem corruptions rarely affect that older metadata, so it makes sense to use it as a primary resource in the recovery of a corrupted disk. But, first, one needs to be able to _find_ that older metadata. 

To that end, btrfs maintains an array containing the block locations of many older versions of the filesystem root. The root block, he said, is more important than the superblock when it comes to recovering data. The root is replaced often as metadata changes percolate up to the top of the directory hierarchy, so the "old root blocks" array contains pointers to what is, in effect, a set of snapshots of the very recent state of the filesystem. Clearly, this will be a valuable resource should something go badly wrong. 

One way of using that array is simply to mount the filesystem using an older version of the root. Chris demonstrated this feature by poking holes in a test filesystem, then mounting an older root to get back to where things had been before. For simple, quickly-detected problems, older root blocks should be a path toward a quick solution. 

It is not too hard to imagine situations where this approach will not work, though. If a metadata block in a rarely-changed subtree is, say, zeroed by a hardware malfunction, it could go undetected for some time. By the time the user realizes that something is wrong, there may be no older hierarchy containing the information needed to put things back together. So other solutions will be necessary. 

Obviously, one of those solutions will be the full filesystem checker and repair tool. That tool is still not ready, though. Getting a repair tool right is a hard problem; without a lot of care, a well-intentioned attempt to repair a filesystem can easily make it worse. Data that may have been recoverable before the repair attempt may no longer be so afterward. Even if a proper btrfsck were available today, it would probably be some years before it reflected enough experience to inspire confidence in users who are concerned about their data. 

So it seems that something else is required. That "something else" turns out to be a data recovery tool written by Josef Bacik. This tool has a simple (to explain) job: dig through a corrupted filesystem in read-only mode and extract as much of the data as possible. Since it makes no changes, it cannot make things worse; it seems like a worthwhile tool to have around even if a full repair tool existed. 

That tool, along with all the requisite filesystem support, is expected to be available in the 3.2 kernel time frame. Meanwhile, there is [a new btrfs-progs repository](<https://lwn.net/Articles/465175/>) that will include the recovery tool in the near future. All told, it may not be quite the btrfsck that some users were hoping for, but it should be enough to make those users feel a bit more confident about entrusting their data to a new filesystem. Judging from the size of the crowd at Chris's talk, there are a lot of people interested in doing exactly that. 

[Your editor would like to thank the Linux Foundation for funding his travel to LinuxCon Europe.]
