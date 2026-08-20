---
title: A short history of btrfs
url: https://lwn.net/Articles/342892/
date: "July 22, 2009"
category: "Btrfs; Filesystems-Btrfs"
author: "July 22, 2009 This article was contributed by Valerie Aurora"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

July 22, 2009

This article was contributed by Valerie Aurora

You probably have heard of the cool new kid on the file system block, [btrfs](<http://btrfs.wiki.kernel.org>) (pronounced "butter-eff-ess") - after all, Linus Torvalds is using it as his root file system on one of his laptops. But you might not know much about it beyond a few high-level keywords - copy-on-write, checksums, writable snapshots - and a few sensational rumors and stories - the [Phoronix benchmarks](<http://www.phoronix.com/scan.php?page=article&item=btrfs_benchmarks&num=1>), btrfs is a ZFS ripoff, btrfs is a secret plan for Oracle domination of Linux, etc. When it comes to file systems, it's hard to tell truth from rumor from vile slander: the code is so complex, the personalities are so exaggerated, and the users are so angry when they lose their data. You can't even settle things with a battle of the benchmarks: file system workloads vary so wildly that you can make a plausible argument for why any benchmark is either totally irrelevant or crucially important. 

In this article, we'll take a behind-the-scenes look at the design and development of btrfs on many levels - technical, political, personal - and trace it from its origins at a workshop to its current position as Linus's root file system. Knowing the background and motivation for each step will help you understand why btrfs was started, how it works, and where it's going in the future. By the end, you should be able to hand-wave your way through a description of btrfs's on-disk format. 

**Disclaimer** : I have two huge disclaimers to make: One, I worked on ZFS for several years while at Sun. Two, I have already been subpoenaed and deposed for the various Sun/NetApp patent lawsuits and I'd like to avoid giving them any excuse to subpoena me again. I'll do my best to be fair, honest, and scrupulously correct. 

#### btrfs: Pre-history

Imagine you are a Linux file system developer. It's 2007, and you are at the [Linux Storage and File systems workshop](<http://www.usenix.org/event/lsf07/>). Things are looking dim for Linux file systems: Reiserfs, plagued with quality issues and an unsustainable funding model, has just lost all credibility with the arrest of Hans Reiser a few months ago. ext4 is still in development; in fact, it isn't even [called ext4 yet](<http://lkml.org/lkml/2006/6/28/454>). Fundamentally, ext4 is just a straightforward extension of a 30-year-old format and is light-years behind the competition in terms of features. At the same time, companies are clamping down on funding for Linux development; IBM's Linux division is coming to the end of its grace period and needs to show profitability now. Other companies are catching wind of an upcoming recession and are cutting research across the board. They want projects with time to results measured in months, not years. 

Ever hopeful, the file systems developers are meeting anyway. Since the workshop is co-located with [USENIX FAST '07](<http://www.usenix.org/event/fast07/>), several researchers from academia and industry are presenting their ideas to the workshop. One of them is Ohad Rodeh. He's invented a kind of [btree that is copy-on-write (COW) friendly [PDF]](<https://btrfs.wiki.kernel.org/images-btrfs/6/68/Btree_TOS.pdf>). To start with, [btrees](<http://en.wikipedia.org/wiki/B-tree>) in their native form are wildly incompatible with COW. The leaves of the tree are linked together, so when the location of one leaf changes (via a write - which implies a copy to a new block), the link in the adjacent leaf changes, which triggers another copy-on-write and location change, which changes the link in the next leaf... The result is that the entire btree, from top to bottom, has to be rewritten every time one leaf is changed. 

Rodeh's btrees are different: first, he got rid of the links between leaves of the tree - which also "throws out a lot of the existing b-tree literature", as he says in his [slides [PDF]](<https://btrfs.wiki.kernel.org/images-btrfs/6/63/LinuxFS_Workshop.pdf>) \- but keeps enough btree traits to be useful. (This is a fairly standard form of btrees in file systems, sometimes called "B+trees".) He added some algorithms for traversing the btree that take advantage of reference counts to limit the amount of the tree that has to be traversed when deleting a snapshot, as well as a few other things, like proactive split and merge of interior nodes so that inserts and deletes don't require any backtracking. The result is a simple, robust, generic data structure which very efficiently tracks extents (groups of contiguous data blocks) in a COW file system. Rodeh successfully prototyped the system some years ago, but he's done with that area of research and just wants someone to take his COW-friendly btrees and put them to good use. 

#### btrfs: The beginning

Chris Mason took these COW-friendly btrees and ran with them. Back in the day, Chris worked on Reiserfs, where he learned a lot about what to do and what not to do in a file system. Reiserfs had some cool features - small file packing, btrees for fast lookup, flexible layout \- but the implementation tended to be haphazard and ad hoc. Code paths proliferated wildly, and along with them potential bugs. 

Chris had an insight: What if everything in the file system - inodes, file data, directory entries, bitmaps, the works - was an item in a copy-on-write btree? All reads and writes to storage would go through the same code path, one that packed the items into btree nodes and leaves without knowing or caring about the item type. Then you only have to write the code once and you get checksums, reference counting (for snapshots), compression, fragmentation, etc., for anything in the file system. 

Chris came up with the following [basic structure for btrfs](<http://btrfs.wiki.kernel.org/index.php/Btrfs_design>) ("btrfs" comes from "btree file system"). Btrfs consists of three types of on-disk structures: block headers, keys, and items, currently defined as follows: 

> 

```
>     struct btrfs_header {
>         u8 csum[32];
>         u8 fsid[16];
>         __le64 blocknr;
>         __le64 flags;
>     
>         u8 chunk_tree_uid[16];
>         __le64 generation;
>         __le64 owner;
>         __le32 nritems;
>         u8 level;
>     }
>     
>     struct btrfs_disk_key {
>         __le64 objectid;
>         u8 type;
>         __le64 offset;
>     }
>     
>     struct btrfs_item {
>         struct btrfs_disk_key key;
>         __le32 offset;
>         __le32 size;
>     }
>
```

Inside the btree (that is, the "branches" of the tree, as opposed to the leaves at the bottom of the tree), nodes consist only of keys and block headers. The keys tell you where to go looking for the item you want, and the block headers tell you where the next node or leaf in the btree is located on disk. 

![\[Btrfs block structure\]](https://static.lwn.net/images/ns/kernel/btrfs/btrfs-block.png) The leaves of the btree contain items, which are a combination of keys and data. Similarly to reiserfs, the items and data are packed in extremely space-efficient way: the item headers (that is, the item structure described above) are packed together starting at the beginning of the block, and the data associated with each item is packed together starting at the end of the block. So item headers and data grow towards each other, as shown in the diagram to the right. 

Besides being code efficient, this scheme is space and time efficient as well. Normally, file systems put only one kind of data - bitmaps, or inodes, or directory entries - in any given file system block. This wastes disk space, since unused space in one kind of block can't be used for any other purpose, and it wastes time, since getting to one particular piece of file data requires reading several different kinds of metadata, all located in different blocks in the file system. In btrfs, items are packed together (or pushed out to leaves) in arrangements that optimize both access time and disk space. You can see the difference in these (very schematic, very simplified) diagrams. Old-school filesystems tend to organize data like this: 

> ![\[Old-school\]](https://static.lwn.net/images/ns/kernel/btrfs/oldskool.png)

Btrfs, instead, creates a disk layout which looks more like: 

> ![\[new school\]](https://static.lwn.net/images/ns/kernel/btrfs/newskool.png)

In both diagrams, red blocks denote wasted disk space and red arrows denote seeks. 

Each kind of metadata and data in the file system - a directory entry, an inode, an extended attribute, file data itself - is stored as a particular type of item. If we go back to the definition of an item, we see that its first element is a key: 

> 

```
>     struct btrfs_disk_key {
>         __le64 objectid;
>         u8 type;
>         __le64 offset;
>     }
>
```

Let's start with the `objectid` field. Each object in the file system - generally an inode - has a unique objectid. This is fairly standard practice - it's the equivalent of inode numbers. What makes btrfs interesting is that the objectid makes up the most significant bits of the item key - what we use to look up an item in the btree - and the lower bits are different kinds of items related to that objectid. This results in grouping together all the information associated with a particular objectid. If you allocate adjacent objectids, then all the items from those objectids are also allocated close together. The `<objectid, type>` pair automatically groups related data close to each other regardless of the actual content of the data, as opposed to the classical file system approach, which writes separate optimized allocators for each kind of file system data. 

The `type` field tells you what kind of data is stored in the item. Is it the inode? Is it a directory entry? Is it an extent telling you where the file data is on disk? Is it the file data itself? With the combination of objectid and the type, you can look up any file system data you need in the btree. 

We should take a quick look at the structure of the btree nodes and leaves themselves. Each node and leaf is an extent in the btree - nodes are extents full of `<key, block header>` pairs, and leaves contain items. Large file data is stored outside of the btree leaves, with the item describing the extent kept in the leaf itself. (What constitutes a "large" file is tunable based on the workload.) Each extent describing part of the btree has a checksum and a reference count, which permits writable snapshots. Each extent also includes an explicit back reference to each of the extents that refer to it. 

Back references give btrfs a major advantage over every other file system in its class because now we can quickly and efficiently migrate data, incrementally check and repair the file system, and check the correctness of reference counts during normal operation. The proof is that btrfs already supports fast, efficient device removal and shrinking of the available storage for a file system. Many other file systems list "shrink file system" as a feature, but it usually ends up implemented inefficiently and slowly and several years late - or not at all. For example, ext3/4 can shrink a file system - by traversing the entire file system searching for data located in the area of the device being removed. It's a slow, fraught, bug-prone process. ZFS still [can't shrink a file system](<http://bugs.opensolaris.org/view_bug.do?bug_id=4852783>). 

The result is beautifully generic and elegant: Everything on disk is a btree containing reference counted, checksummed extents of items, organized by `<objectid, type>` keys. A great deal of the btrfs code doesn't care at all what is stored in the items, it just knows how to add or remove them from the btree. Optimizing disk layout is simple: allocate things with similar keys close together. 

#### btrfs: The politics

At the same time that Chris was figuring out the technical design of btrfs, he was also figuring out how to fund the development of btrfs in both the short and the long term. Chris had recently moved from SUSE to a special Linux group at Oracle, one that employs several high-level Linux storage developers, including Martin K. Petersen, Zach Brown, and Jens Axboe. Oracle funds a lot of Linux development, some of it obviously connected to the Oracle database (OCFS2, DIF/DIX), and some of it less so (generic block layer work, syslets). Here's how Chris put it in a recent [interview](<http://www.linux-foundation.org/weblogs/amanda/2009/06/22/a-conversation-with-chris-mason-on-btrfs-the-next-generation-file-system-for-linux/>) with [Amanda McPherson from the Linux Foundation](<http://www.linux-foundation.org/weblogs/amanda/>): 

> **Amanda:** Why did you start this project? Why is Oracle supporting this project so prominently? 
> 
> **Chris** : I started Btrfs soon after joining Oracle. I had a unique opportunity to take a detailed look at the features missing from Linux, and felt that Btrfs was the best way to solve them. 
> 
> Linux is a very important platform for Oracle. We use it heavily for our internal operations, and it has a broad customer base for us. We want to keep Linux strong as a data center operating system, and innovating in storage is a natural way for Oracle to contribute. 

In other words, Oracle likes having Linux as a platform, and is willing to invest development effort in it even if it's not directly related to Oracle database performance. Look at it this way: how many operating systems are written and funded in large part by your competitors? While it is tempting to have an operating system entirely under your control - like Solaris - it also means that you have to pay for most of the development on that platform. In the end, Oracle believes it is in its own interest to use its in-house expertise to help keep Linux strong. 

After a few months of hacking and design discussions with Zach Brown and many others, Chris [posted btrfs for review](<http://lkml.org/lkml/2007/6/12/242>). From there on out, you can trace the history of btrfs like any other open source project through the mailing lists and source code history. Btrfs is now in the mainline kernel and developers from Red Hat, SUSE, Intel, IBM, HP, Fujitsu, etc. are all working on it. Btrfs is a true open source project - not just in the license, but also in the community. 

#### btrfs: A brief comparison with ZFS

People often ask about the relationship between btrfs and ZFS. From one point of view, the two file systems are very similar: they are copy-on-write checksummed file systems with multi-device support and writable snapshots. From other points of view, they are wildly different: file system architecture, development model, maturity, license, and host operating system, among other things. Rather than answer individual questions, I'll give a short history of ZFS development and compare and contrast btrfs and ZFS on a few key items. 

When ZFS first got started, the outlook for file systems in Solaris was rather dim as well. Logging UFS was already nearing the end of its rope in terms of file system size and performance. UFS was so far behind that many Solaris customers paid substantial sums of money to Veritas to run VxFS instead. Solaris needed a new file system, and it needed it soon. 

Jeff Bonwick decided to solve the problem and started the ZFS project inside Sun. His organizing metaphor was that of the virtual memory subsystem - why can't disk be as easy to administer and use as memory? The central on-disk data structure was the slab - a chunk of disk divided up into the same size blocks, like that in the [SLAB kernel memory allocator](<http://en.wikipedia.org/wiki/Slab_allocation>), which he also created. Instead of extents, ZFS would use one block pointer per block, but each object would use a different block size - e.g., 512 bytes, or 128KB - depending on the size of the object. Block addresses would be translated through a virtual-memory-like mechanism, so that blocks could be relocated without the knowledge of upper layers. All file system data and metadata would be kept in objects. And all changes to the file system would be described in terms of changes to objects, which would be written in a copy-on-write fashion. 

In summary, btrfs organizes everything on disk into a btree of extents containing items and data. ZFS organizes everything on disk into a tree of block pointers, with different block sizes depending on the object size. btrfs checksums and reference-counts extents, ZFS checksums and reference-counts variable-sized blocks. Both file systems write out changes to disk using copy-on-write - extents or blocks in use are never overwritten in place, they are always copied somewhere else first. 

So, while the feature list of the two file systems looks quite similar, the implementations are completely different. It's a bit like [convergent evolution between marsupials and placental mammals](<http://txtwriter.com/Backgrounders/Evolution/EVpage14.html>) \- a marsupial mouse and a placental mouse look nearly identical on the outside, but their internal implementations are quite a bit different! 

In my opinion, the basic architecture of btrfs is more suitable to storage than that of ZFS. One of the major problems with the ZFS approach - "slabs" of blocks of a particular size - is fragmentation. Each object can contain blocks of only one size, and each slab can only contain blocks of one size. You can easily end up with, for example, a file of 64K blocks that needs to grow one more block, but no 64K blocks are available, even if the file system is full off nearly empty slabs of 512 byte blocks, 4K blocks, 128K blocks, etc. To solve this problem, we (the ZFS developers) invented ways to create big blocks out of little blocks ("gang blocks") and other unpleasant workarounds. In our defense, at the time btrees and extents seemed fundamentally incompatible with copy-on-write, and the virtual memory metaphor served us well in many other respects. 

In contrast, the items-in-a-btree approach is extremely space efficient and flexible. Defragmentation is an ongoing process - repacking the items efficiently is part of the normal code path preparing extents to be written to disk. Doing checksums, reference counting, and other assorted metadata busy-work on a per-extent basis reduces overhead and makes new features (such as fast reverse mapping from an extent to everything that references it) possible. 

Now for some personal predictions (based purely on public information - I don't have any insider knowledge). Btrfs will be the default file system on Linux within two years. Btrfs as a project won't (and can't, at this point) be canceled by Oracle. If all the intellectual property issues are worked out (a big if), ZFS will be ported to Linux, but it will have less than a few percent of the installed base of btrfs. Check back in two years and see if I got any of these predictions right! 

#### Btrfs: What's next?

Btrfs is heading for 1.0, a little more than 2 years since the first announcement. This is much faster than many file systems veterans - including myself - expected, especially given that during most of that time, btrfs had only one full-time developer. Btrfs is not ready for production use - that is, storing and serving data you would be upset about losing - but it is ready for widespread testing - e.g., on your backed-up-nightly laptop, or your experimental netbook that you reinstall every few weeks anyway. 

Be aware that there was a recent flag day in the btrfs on-disk format: A commit shortly after the 2.6.30 release changed the on disk format in a way that isn't compatible with older kernels. If you create your btrfs file system using the old, 2.6.30 or earlier kernel and tools, and boot into a newer kernel with the new format, you won't be able to use your file system with a 2.6.30 or older kernel any longer. Linus Torvalds [found this out the hard way.](<http://www.mail-archive.com/linux-btrfs@vger.kernel.org/msg02500.html>) But if this does happen to you, don't panic - you can find [rescue images and other helpful information](<http://btrfs.wiki.kernel.org/index.php/Rescue_me>) on the the btrfs wiki.
