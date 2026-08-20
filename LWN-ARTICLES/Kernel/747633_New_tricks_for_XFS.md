---
title: New tricks for XFS
url: https://lwn.net/Articles/747633/
date: "February 21, 2018"
category: "Filesystems-XFS"
author: "By Jake Edge February 21, 2018 linux.conf.au"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jake Edge**  
February 21, 2018

* * *

[linux.conf.au](<https://lwn.net/Archives/ConferenceByYear/#2018-linux.conf.au>)

The XFS filesystem has been in the kernel for fifteen years and was used in production on [IRIX](<https://en.wikipedia.org/wiki/IRIX>) systems for five years before that. But it might just be time to teach that "old dog" of a filesystem some new tricks, Dave Chinner said, at the beginning of his linux.conf.au 2018 presentation. There are a number of features that XFS lacks when compared to more modern filesystems, such as snapshots and subvolumes; but he has been thinking—and writing code—on a path to get them into XFS. 

#### Some background

XFS is the "original [B-tree](<https://en.wikipedia.org/wiki/B-tree>) filesystem" since everything that the filesystem stores is organized in B-trees. They are not actually a traditional B-tree, rather they are a form of [B* tree](<https://en.wikipedia.org/wiki/B-tree#Variants>). A difference is that each node has a sibling pointer, which allows horizontal traversal of the tree. That kind of traversal is important when looking at features like copy on write (CoW). 

An XFS filesystem is split into allocation groups, "which are like mini-filesystems"; they have their own free-space index B-trees, inode B-trees, reverse-mapping B-trees, and so on. File data is referenced by extents, with the help of B-trees. "Directories and attributes are more B-trees"; the directory B-tree is the most complex as it is a "virtually mapped, multiple index B-tree with all sorts of hashing" for scalability. 

XFS uses writeahead journaling for crash resistance. It has checkpoint-based journaling that is meant to reduce the write amplification that can result from changing blocks that are already in the journal. 

He followed that with a quick overview of CoW filesystems. When a CoW filesystem writes to a block of data or metadata, it first makes a copy of it; in doing so, it needs to update the index tree entries to point to the new block. That leads to modifying the block that holds those entries, which necessitates another copy, thus a modification to the parent index entry, and so on, all the way up to the root of the filesystem. All of those updates can be written together anywhere in the filesystem, which allows lots of optimizations to be done. It also provides consistent on-disk images, since the entire update can be written prior to making an atomic change to the root-level index. 

All of that is great for crash recovery, he said, but the downside is that it requires that space be allocated for these on-disk updates. That allocation process requires metadata updates, which means a metadata tree update, thus more space needs to be allocated for that. That leads to the problem that the filesystem does not know exactly how much space is going to be needed for a given CoW operation. "That leads to other problems in the future." 

These index tree updates are what provide many of the features that are associated with CoW filesystems, Chinner said, "sharing, snapshots, subvolumes, and so on". They are all a natural extension of having an index tree structure that reference-counts objects; that allows multiple indexes to point to the same object by just increasing the reference count on it. Snapshots are simply keeping around an index tree that has been superseded; that can be done by taking a reference to that tree. Replication is done by creating a copy of the tree and all of its objects, which is a complicated process, but "does give us the send-receive-style replication" that users are familiar with. 

CoW in XFS is different. Because of the B* trees, it cannot do the leaf-to-tip update that CoW filesystems do; it would require updating laterally as well, which in the worst case means updating the entire filesystem. So CoW in XFS is data-only. 

Data-only CoW limits the functionality that XFS can provide; features like deduplication and file cloning are possible, but others are not. The features it does provide are useful for projects like overlayfs and NFS, Chinner said. The advantage of data-only CoW is that there is no impact on non-shared data or metadata. In addition, XFS can always calculate how much space is needed for a CoW operation because only the data is copied; the metadata is updated in place. 

But, since the metadata updates are not done with CoW, crash resiliency is a bit more difficult—it is not a matter of simply writing a new tree branch and then switching to it atomically. XFS has implemented "deferred operations", which are a kind of "intent logging mechanism", Chinner said. Deferred operations were used for freeing extents in the past, but have been extended to do reference count and reverse B-tree mapping updates. That allows replaying CoW updates as part of recovery. 

#### What is a subvolume?

Thinking about all of that led Chinner to a number of questions about what can be done with data-only CoW. Everyone seems to want subvolume snapshots, but that seems to require CoW operations for metadata. How can the problem be repackaged so that there is a way to implement the same functionality? That is the ultimate goal, of course. He wondered how much of a filesystem was actually needed to implement a subvolume. There are other implementations to look at, so we can learn from them, he said. "What should we avoid? What do they do right?" The good ideas can be stolen—copied—"because that's the easy way". 

Going back to first principles, he asked: "what is a subvolume? What does it provide?" From what he can tell, there are three attributes that define a subvolume. It has flexible capacity, so it can grow or shrink without any impact. A subvolume is also a fully functioning filesystem that allows operations like punching holes in files or cloning. The main attribute, though, is that a subvolume is the unit of granularity for snapshots. Everything else is built on top of those three attributes. 

He asked: could subvolumes be implemented as a namespace construct that sits atop the filesystem? Bind mounts and mount namespaces already exist in VFS, he wondered if those could be used to create something that "looks like and smells like a subvolume". If you add a directory hierarchy quota on top of a bind mount, it will result in a kind of flexible space management. If you "squint hard enough", that is something like a subvolume, he said. 

Similarly, a recursive copy operation using `--reflink=always` can create a kind of snapshot. It still replicates the metadata, but the vast majority of the structure has been cloned without copying the data. Replication can be done with `rsync` and `tar`; "sure, it's slow", but there are tools to do that sort of thing. It doesn't really resemble a Btrfs subvolume, for example, but it can still provide the same functionality, Chinner said. In addition, overlayfs copies data and replicates metadata, so it shows that you can provide "something that looks like a subvolume using data-only copy on write". 

Another idea might be to implement the subvolume below the filesystem with a device construct of some sort. In fact, we already have that, he said. A filesystem image can be stored in a sparse file, then loopback mounted. That image file can be cloned with data-only CoW, which allows for fast snapshots. The space management is "somewhat flexible", but is limited by what the block layer provides and what filesystems implement. Replication is a simple file copy. 

What this shows "is that what we think of as a subvolume, we're already using", Chinner said. The building blocks are there, they are just being used in ways that do not make people think of subvolumes. 

The loopback filesystem solution suffers from the classic `ENOSPC` problem, however. If the filesystem that holds the image file runs out of space, it will communicate that by returning `ENOSPC`, but the filesystem inside the image will not be prepared to handle that failure and things break horribly: "blammo!". This is same problem that thin provisioning has. It is worse than the CoW filesystem `ENOSPC` problem, because you can't predict when it will happen and you can't recover when it does, he said. 

He returned to the idea of learning from others at that point. Overlayfs and, to a lesser extent, Btrfs have taught us that specifying subvolumes via mount options is "really, really clunky", Chinner said. Btrfs subvolumes share the same superblock, which can cause some subtle issues about how they are treated by various tools like `find` or backup programs. A subvolume needs to be implemented as an independent VFS entity and not just act like one. "There's only so much you can hide by lying." 

The `ENOSPC` problem is important to solve. The root of the problem is that upper and lower volumes (however defined) have a different view of free-space availability and those two layers do not communicate about it. This problem has been talked about many times at [LSFMM](<https://lwn.net/Archives/ConferenceIndex/#Storage_Filesystem__Memory_Management>) (for example, [in 2017](<https://lwn.net/Articles/718803/>) and [in 2016](<https://lwn.net/Articles/684465/>)) without making any real progress. But a while back, Christoph Hellwig came up with a file layout interface for the [Parallel NFS](<http://www.pnfs.com/>) (pNFS) server running on top of XFS; it allowed the pNFS client to remotely map files from the server and to allocate blocks from the server. The actual data lives elsewhere and the client does its reads and writes to those locations; so the client is doing its filesystem allocation on the server and then doing the I/O to somewhere else. This provides a model for a cross-layer communication of space accounting and management that is "very instructive". 

#### A new kind of subvolume

He has been factoring all of this into his thinking on a new type of subvolume; one that acts the same as the subvolumes CoW filesystems have, but is implemented quite differently. The kernel could be changed so that it can directly mount image files (rather than via the loopback device) and a device space-management API could be added. If a filesystem implements both sides of that API, image files of the same filesystem type can be used as subvolumes. The API can be used to get the mapping information, which will allow the subvolume to do its I/O directly to the host filesystem's block device. This breaks the longstanding requirement that filesystems must use block devices; with his changes, they can now use files directly. 

But this mechanism will still work for block devices, which will make it useful for thin provisioning as well. The thin-provisioned block device (such as dm-thin) can implement the host side of the space-management API; the filesystem can then use the client-side API for space accounting and I/O mapping. That way the underlying block device will report `ENOSPC` before the filesystem has modified its structures and issued I/O. That is something of a bonus, he said, but if his idea solves two problems at once, that gives him reason to think he is on the right track. 

Snapshots are "really easy in this model". The subvolume is frozen and the image file is cloned. It is fast and efficient. In effect, the subvolume gets CoW metadata even though its filesystem does not implement it; the data-only CoW of the filesystem below (where the image file resides) provides the metadata CoW. 

Replication could be done by copying the image files, but there are better ways to do it. Two image files can be compared to determine which blocks have changed between two snapshots. It is quite simple to do and does not require any knowledge of what is in the files being replicated. He implemented a prototype using XFS filesystems on loopback devices in 200 lines of shell script using `xfs_io`. "It's basically a delta copy" that is independent of what is in the filesystem image; if you had two snapshots of ext4 filesystems, the same code would work, he said. 

There are features that people are asking for that the current CoW filesystems (e.g. Btrfs, ZFS) cannot provide, but this new scheme could. Right now, there is a lot of data shared between files on disk that is not shared once it gets to the page cache. If you have 500 containers based on the same golden image, you can have multiple snapshots being used but each container has its own version of the same file in the cache. "So you have 500 copies of `/bin/bash` in memory", he said. Overlayfs does this the right way since it shares the one cached version of the unmodified Bash between all of the containers. 

His goal is to get that behavior for this new scheme as well. That requires sharing the data in shared extents in the page cache. It is a complex and difficult problem, Chinner said, because the page cache is indexed by file and offset, whereas the only information available for the shared extents is their physical location in the filesystem (i.e. the block number). Instead of doing an exhaustive search in the page cache to see if a shared extent is cached, he is proposing adding a buffer cache that is indexed by block number. XFS already has a buffer cache, but it doesn't have a way to share pages between multiple files. Chinner indicated that Matthew Wilcox was working on solving that particular problem; that solution would be coming "maybe next week", he said with a grin. 

For a long time people have been saying that you don't need encryption for subvolumes because containers are isolated, but then came Meltdown and Spectre, which broke all that isolation. He thinks that may lead some to want more layers of defense to make it harder to steal their data when that isolation breaks down. Adding the generic VFS file-encryption API to XFS will allow encrypting the image files and/or individual files within a subvolume. There might be something to be gained by adding key management into the space-management API as well. 

It is looking like XFS could offer "encrypted, snapshottable, cloned subvolumes with these mechanisms", Chinner said. There is still a lot of work to do to get there, of course; it is still in its early stages. 

The management interface that will be presented to users is not nailed down yet; he has been concentrating on getting the technology working before worrying about policy management. How subvolumes are represented, what the host volume looks like to users, and whether everything is a subvolume are all things that need to be worked out. There is also a need to integrate this work with tools like Anaconda and Docker. 

None of the code has had any review yet; it all resides on his laptop and servers. Once it gets posted, there will be lots of discussion about the pieces he will need to push into the kernel as well as the XFS-specific parts. There will probably be "a few flame wars around that, a bit of shouting, all the usual melodrama that goes along with doing controversial things". He recommended popcorn. 

He then gave a demo (starting around 36:56 in the [YouTube video](<https://www.youtube.com/watch?v=wG8FUvSGROw>) of the talk) of what he had gotten working so far. It is a fairly typical early stage demo, but managed to avoid living up to the names of the subvolume and snapshot, which were "blammo" and "kaboom". 

After the demo, Chinner summarized the talk (and the work). He started out by looking at how to get the same functionality as subvolumes, but without implementing copy on write for metadata. The "underlying revelation" was to use files as subvolumes and to treat subvolumes as filesystems. That gives the same functionality as a CoW filesystem for that old dog XFS. 

[I would like to thank LWN's travel sponsor, the Linux Foundation, for travel assistance to Sydney for LCA.]
