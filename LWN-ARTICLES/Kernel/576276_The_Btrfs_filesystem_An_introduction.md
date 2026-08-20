---
title: "The Btrfs filesystem: An introduction"
url: https://lwn.net/Articles/576276/
date: "December 11, 2013"
category: "Btrfs-LWNs guide to; Filesystems-Btrfs"
author: "By Jonathan Corbet December 11, 2013"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
December 11, 2013

The Btrfs filesystem has been through almost every part of the hype cycle at one point or another in its short history. For a time, it was the next-generation filesystem that was going to solve many of our problems; distributors were racing to see who could be the first to ship it by default. Then it became clear that all those longtime filesystem developers weren't totally out to lunch when they warned that it takes many years to get a filesystem to a point where it can be trusted with important data. At this point, it is possible to go to a conference and hear almost nothing positive about Btrfs; disillusionment appears to have set in. By the time Btrfs is truly ready, some seem to think, it will be thoroughly obsolete. 

The truth may not be quite so grim. Development on Btrfs continues, with a strong emphasis on stability and performance. Problems _are_ getting fixed, and users are beginning to take another look at this promising filesystem. More users are beginning to play with it, and openSUSE [considered](<https://lwn.net/Articles/576280/>) the idea of using it by default back in September. Your editor's sense is that the situation may be bottoming out, and that we may, slowly, be heading into a new phase where Btrfs takes its place — still slowly — as one of the key Linux filesystems. 

This article is intended to be the first in a series for users interested in experimenting with and evaluating the Btrfs filesystem. We'll start with the basics of the design of the filesystem and how it is being developed; that will be followed by a detailed look at specific Btrfs features. One thing that will _not_ appear in this series, though, is benchmark results; experience says that proper filesystem benchmarking is hard to do right; it's also highly workload- and hardware-dependent. Poor-quality results would not be helpful to anybody, so your editor will simply not try. 

#### What makes Btrfs different?

Not that long ago, Linux users were still working with filesystems that had evolved little since the Unix days. The ext3 filesystem, for example, was still using block pointers: each file's inode (the central data structure holding all the information about the file) contained a list of pointers to each individual block holding the file's data. That design worked well enough when files were small, but it scales poorly: a 1GB file would require 256K individual block pointers. More recent filesystems (including ext4) use pointers to "extents" instead; each extent is a group of contiguous blocks. Since filesystems work to store data contiguously anyway, extent-based storage greatly reduces the overhead of managing a file's space. 

Naturally, Btrfs uses extents as well. But it differs from most other Linux filesystems in a significant way: it is a "copy-on-write" (or "COW") filesystem. When data is overwritten in an ext4 filesystem, the new data is written on top of the existing data on the storage device, destroying the old copy. Btrfs, instead, will move overwritten blocks elsewhere in the filesystem and write the new data there, leaving the older copy of the data in place. 

The COW mode of operation brings some significant advantages. Since old data is not overwritten, recovery from crashes and power failures should be more straightforward; if a transaction has not completed, the previous state of the data (and metadata) will be where it always was. So, among other things, a COW filesystem does not need to implement a separate journal to provide crash resistance. 

Copy-on-write also enables some interesting new features, the most notable of which is snapshots. A snapshot is a virtual copy of the filesystem's contents; it can be created without copying any of the data at all. If, at some later point, a block of data is changed (in either the snapshot or the original), that one block is copied while all of the unchanged data remains shared. Snapshots can be used to provide a sort of "time machine" functionality, or to simply roll back the system after a failed update. 

Another important Btrfs feature is its built-in volume manager. A Btrfs filesystem can span multiple physical devices in a number of RAID configurations. Any given volume (collection of one or more physical drives) can also be split into "subvolumes," which can be thought of as independent filesystems sharing a single physical volume set. So Btrfs makes it possible to group part or all of a system's storage into a big pool, then share that pool among a set of filesystems, each with its own usage limits. 

Btrfs offers a wide range of other features not supported by other Linux filesystems. It can perform full checksumming of both data and metadata, making it robust in the face of data corruption by the hardware. Full checksumming is expensive, though, so it remains likely to be used in only a minority of installations. Data can be stored on-disk in compressed form. The [send/receive feature](<https://lwn.net/Articles/506244/>) can be used as part of an incremental backup scheme, among other things. The online defragmentation mechanism can fix up fragmented files in a running filesystem. The 3.12 kernel saw the addition of an offline de-duplication feature; it scans for blocks containing duplicated data and collapses them down to a single, shared copy. And so on. 

It is worth noting that the copy-on-write approach is not without its costs. Obviously, some sort of garbage collection is required or all those block copies will quickly eat up all of the available space on the filesystem. Copying blocks can take more time than simply overwriting them as well as significantly increasing the filesystem's memory requirements. COW operations will also have a tendency to fragment files, wrecking the nice, contiguous layout that the filesystem code put so much effort into creating. Fragmentation hurts less with solid-state devices than on rotational storage, but, even in the former case, fragmented files will not be as quick to access. 

So all this shiny new Btrfs functionality does not come for free. In many settings, administrators may well decide that the costs associated with Btrfs outweigh the benefits; those sites will stick with filesystems like ext4 or XFS. For others, though, the flexibility and feature set provided with Btrfs are likely to be quite appealing. Once it is generally accepted that Btrfs is ready for real-world use, chances are it will start popping up on a lot of systems. 

#### Development

One concern your editor has heard in conference hallways is that the pace of Btrfs development has slowed. For the curious, here's the changeset count history for the Btrfs code in the kernel, grouped into approximately one-year periods: 

> Year| Changesets| Developers  
> ---|---|---  
> 2008 (2.6.25—29)| 913| 42  
> 2009 (2.6.30—33)| 279| 45  
> 2010 (2.6.34—37)| 193| 33  
> 2011 (2.6.38—3.2)| 610| 67  
> 2012 (3.3—8)| 773| 63  
> 2013 (3.9—13)| 671| 68  
  
These numbers, on their own, do not demonstrate a slowing of development; there was an apparent slow period in 2010, but the number of changesets and the number of developers contributing them has held steady thereafter. That said, there are a couple of things to bear in mind when looking at those numbers. One is that the early work involved the addition of features to a brand-new filesystem, while work in 2013 is almost entirely fixes. So the _size_ of the changes has shrunk considerably, but one could easily argue that things should be just that way. 

The other relevant point is that contributions by Btrfs creator Chris Mason have clearly fallen in recent years. Partly that is because he has been working on the user-space btrfs-progs code — work which is not reflected in the above, kernel-side-only numbers — but it also seems clear that he has been busy with other work-related issues. It will be interesting to see how things change now that Chris and prolific Btrfs contributor Josef Bacik have [found a new home](<https://lwn.net/Articles/575105/>) at Facebook. 

In summary, the amount of new code going into Btrfs has clearly fallen in recent years, but that will be seen as good news by anybody hoping for a stable filesystem anytime soon. There is still some significant effort going into this filesystem, and chances are good that developer attention will increase as distributors look more closely at using Btrfs by default. 

#### What's next

All told, Btrfs still looks interesting, and it seems like the right time to take a closer look at what is still the next generation Linux filesystem. Now that the introductory material is out of the way, the next article in this series will start to actually play with Btrfs and explore its feature set. Those articles (appearing here as they are published) are: 

  * [Getting started](<https://lwn.net/Articles/577218/>): where to get the software, and the basics of creating and using a Btrfs filesystem. 

  * [Working with multiple devices](<https://lwn.net/Articles/577961/>): a Btrfs filesystem is not limited to a physical device; instead, the filesystem supports multiple-device filesystems in a number of RAID and RAID-like configurations. This article describes that functionality and how to make use of it. 

  * [Subvolumes and snapshots](<https://lwn.net/Articles/579009/>): creating multiple filesystems on a single storage volume, along with the associated snapshot mechanism. 

  * [Send/receive and `ioctl()`](<https://lwn.net/Articles/581558/>): using the send/receive feature for incremental backups, a brief overview of functionality available with `ioctl()`, and concluding notes. 

By the end of the series, we plan to have a reasonably comprehensive introduction to Btrfs in place; stay tuned.
