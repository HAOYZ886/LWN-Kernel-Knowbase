---
title: Online filesystem scrubbing and repair
url: https://lwn.net/Articles/718800/
date: "April 5, 2017"
category: Filesystems
author: "By Jake Edge April 5, 2017 LSFMM 2017"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jake Edge**  
April 5, 2017

* * *

[LSFMM 2017](<https://lwn.net/Articles/lsfmm2017/>)

In his traditional LSFMM session to "whinge about various things", Darrick Wong mostly discussed his recent work on online filesystem repair for XFS, but also strayed into some other topics. Online filesystem scrubbing for XFS was one of those, as was a new `ioctl()` command to determine block ownership.

He started with the [`GETFSMAP` `ioctl()` command](<https://lwn.net/Articles/714985/>), which allows querying a filesystem to determine how its blocks are being used (which blocks contain data or metadata and which are free). He has patches for XFS that are reviewed and ready to go for 4.12. Ext4 support is out for review. Chris Mason said that there are `ioctl()` commands for Btrfs that can be remapped to `GETFSMAP`. 

[ ![\[Darrick Wong\]](https://static.lwn.net/images/2017/lsfmm-wong-sm.jpg) ](<https://lwn.net/Articles/718794/>)

Wong then described an online scrubbing tool for XFS that he has been working on. Right now, it examines the metadata in the filesystem and complains if there is something that is "clearly insane". Part two will be posted soon and will cross-check metadata with other metadata to make sure they are all in agreement. The tool will also use `GETFSMAP` to read all file-data blocks using direct I/O to check to see if any give read errors. 

Damien Le Moal suggested that the RAID rebuild assist feature of some storage devices could be used to either get the data or get a failure quickly. Wong asked if there was a way to get a list of LBAs with read errors, but that isn't supported. Fred Knight said that using RAID rebuild assist will cause the device not to try any "heroic recovery" and simply return errors. Wong said that he hasn't done much work on reading from the device yet as he has been concentrating on the filesystem side. 

For online repair for XFS, he has been working on using the reverse mappings to reconstruct the primary metadata for corrupted filesystems. He does not want to use the metadata that is left over because it may be corrupted too. There is a problem, though, if you have to recreate the secondary metadata (e.g. the reverse mapping) because you run into a deadlock situation. 

The deadlock is avoided by freezing the filesystem so there is no access to the inodes. Inode by inode, this restriction is relaxed as the reverse mapping is rebuilt. The block device is not frozen, just the filesystem, but he wondered if that was a sane thing to do. It is not the intended use case for filesystem freezing, but it seems to work. That is the only part of online repair where the filesystem "has to come to a screeching halt", he said. 

There is no secondary metadata for directories right now, so the online repair just moves corrupted directories to `lost+found` or removes them if they are too corrupted. XFS doesn't have parent directory pointers that might be used to help repair directory corruption. 

There is still no good way to remove open files, Wong said. He thought it might be possible to rip the file's pages out of the page cache, replace the inode operations and mapping fields with dummy values that just return errors, and to try to put the inode on the orphan list. Matthew Wilcox asked if writeback would be done before the pages are removed from the page cache; Wong said that was an open question. 

Mason said that Facebook does no online repair. If a filesystem gets corrupted, it is taken out of production and restored to a working state before being returned to service. Various aspects of the filesystem change (e.g. latencies) while online repair is in progress, so the company just does not bother. Ted Ts'o said that the cloud world tends to have enough redundancy that doing online repair isn't really needed, but in other worlds that is not true. Wong said that the feature is targeted at users who are willing to trade "some to a lot of latency" for the ability to not have to take a volume offline. 

A way to specify a "desperation" flag in block I/O requests is on his wish list, Wong said. Setting that would mean that the block layer should go read RAID-1 mirrors or invoke stronger error recovery to read a block. He would like to use that when a block read succeeds but the checksum doesn't match. 

He asked the room if anyone was making progress on filesystem freezing before suspend operations. Mason said that the use of the work queue `WQ_FREEZABLE` flag was "all wrong" as described in a Kernel Summit [session](<https://lwn.net/Articles/705269/>). The code for the filesystems had all been cut and pasted and was simply wrong. He had thought that someone was planning to fix that, but was not sure where that effort had gotten.
