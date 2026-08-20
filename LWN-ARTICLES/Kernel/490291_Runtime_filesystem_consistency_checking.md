---
title: Runtime filesystem consistency checking
url: https://lwn.net/Articles/490291/
date: "April 3, 2012"
category: Filesystems
author: "By Jake Edge April 3, 2012"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jake Edge**  
April 3, 2012

This year's edition of the Linux Storage, Filesystem, and Memory Management Summit took place in San Francisco April 1-2, just prior to the Linux Foundation Collaboration Summit. Ashvin Goel of the University of Toronto was invited to the summit to discuss the work that he and others at the university had done on consistency checking as filesystems are updated, rather than doing offline checking using tools like `fsck`. One of the students who had worked on the project, Daniel Fryer, was also present to offer his perspective from the audience. Goel said that the work is not ready for production use, and Fryer echoed that, noting that the code is not 100% solid by any means. They are researchers, Goel said, so the community should give them some leeway, but that any input to make their work more relevant to Linux would be appreciated. 

Filesystems have bugs, Goel said, producing a list of bugs that caused filesystem corruption over the last few years. Existing solutions can't deal with these problems because they start with the assumption that the filesystem is correct. Journals, RAID, and checksums on data are nice features but they depend on offline filesystem checking to fix up any filesystem damage that may occur. Those solutions protect against problems below the filesystem layer and not against bugs in the filesystem implementation itself. 

But, he said, offline checking is slow and getting slower as disks get larger. In addition, the data is not available while the `fsck` is being done. Because of that, checking is usually only done after things have obviously gone wrong, which makes the repair that much more difficult. The example given was a file and directory inode that both point to the same data block; how can the checker know which is correct at that point? 

James Bottomley asked if there were particular tools that were used to cause various kinds of filesystem corruption, and if those tools were available for kernel hackers and others to use. Goel said that they have tools for both ext3 and btrfs, while audience members chimed in with other tools to cause filesystem corruptions. Those included fsfuzz, mentioned by Ted Ts'o, which will do random corruptions of a filesystem. It is often used to test whether malformed filesystems on USB sticks can be used to crash or subvert the kernel. There were others, like fswreck for the OCFS2 filesystem, as well as similar tools for XFS noted by Christoph Hellwig and another that Chris Mason said he had written for btrfs. Bottomley's suggestion that the block I/O scheduler could be used to pick blocks to corrupt was met with a response from another in the audience joking that the block layer didn't really need any help corrupting data--widespread laughter ensued. 

Returning to the topic at hand, Goel stated that doing consistency checking at runtime is faced with the problem that consistency properties are global in nature and are therefore expensive to check. To find two pointers to the same data block, one must scan the entire filesystem, for example. In an effort to get around this difficulty, the researchers hypothesized that global consistency properties could be transformed into local consistency invariants. If only local invariants need to be checked, runtime consistency checking becomes a more tractable problem. 

They started with the assumption that the initial filesystem is consistent, and that something below the filesystem layer, like checksums, ensures that correct data reaches the disk. At runtime, then, it is only necessary to check that the local invariants are maintained by whatever data is being changed in any metadata writes. This checking happens before those changes become "durable", so they reason by induction that the filesystem resulting from those is also consistent. By keeping any inconsistent state changes from reaching the disk, the "Recon" system makes filesystem repair unnecessary. 

As an example, ext3 maintains a bitmap of the allocated blocks, so to ensure consistency when a block is allocated, Recon needs to test that the proper bit in the bitmap flips from zero to one and that the pointer used is the correct one (i.e. it corresponds to the bit flipped). That is the "consistency invariant" for determining that the block has been allocated correctly. A bit in the bitmap can't be set without a corresponding block pointer being set and vice versa. Additional checks are done to make sure that the block had not already been allocated, for example. That requires that Recon maintain its own block bitmap. 

These invariants (they came up with 33 of them for ext3) are checked at the transaction commit point. The design of Recon is based on a fundamental mistrust of the filesystem code and data structures, so it sits between the filesystem and the block layer. When the filesystem does a metadata write, Recon records that operation. Similarly, it caches the data from metadata reads, so that the invariants can be validated without excessive disk reads. When the commit of a metadata update is done, the read cache is updated if the invariants are upheld in the update. 

When filesystem metadata is updated, Recon needs to determine what logical change is being performed. It does that by examining the metadata block to determine what type of block it is, and then does a "logical diff" of the changes. The result is a "logical change record" that records five separate fields for each change: block type, ID, the field that changed, the old value, and the new value. As an example, Goel listed the change records that might result from appending a block to inode 12: 

> Type| ID| Field| Old| New  
> ---|---|---|---|---  
> inode| 12| blockptr[1]| 0| 501  
> inode| 12| i_size| 4096| 8192  
> inode| 12| i_blocks| 8| 16  
> bitmap| 501| \--| 0| 1  
> bgd| 0| free_blocks| 1500| 1499  
  
Using those records, the invariants can be checked to ensure that the block pointer referenced in the inode is the same as the one that has its bit set in the bitmap, for example. 

Currently, when any invariant is violated, the filesystem is stopped. Eventually there may be ways to try to fix the problems before writing to disk, but for now, the safe option is to stop any further writes. 

Recon was evaluated by measuring how many consistency errors were detected by it vs. those caught by `fsck`. Recon caught quite a few errors that were not detected by `fsck`, while it only missed two that `fsck` caught. In both cases, the filesystem checker was looking at fields that are not currently used by ext3. Many of the inconsistencies that Recon found and `fsck` didn't were changes to unallocated data, which are not important from a consistency standpoint, but still should not be changed in a correctly operating filesystem. 

There are some things that neither `fsck` nor Recon can detect, like changes to filenames in directories or time field changes in inodes. In both cases, there isn't any redundant information to do a consistency check against. 

The performance impact of Recon is fairly modest, at least in terms of I/O operations. With a cache size of 128MB, Recon could handle a web server workload with only a reduction of approximately 2% I/O operations/second based on a graph that was shown. The cache size was tweaked to find a balance based on the working set size of the workload so that the cache would not be flushed prematurely, which would otherwise cause expensive reads of the metadata information. The tests were run on a filesystem on a 1TB partition with 15-20GB of random files according to Fryer, and used small files to try to stress the metadata cache. 

No data was presented on the CPU impact of Recon, other than to say that there was "significant" CPU overhead. Their focus was on the I/O cost, so more investigation of the CPU cost is warranted. Based on comments from the audience, though, some would be more than willing to spend some CPU in the name of filesystem consistency so that the far more expensive offline checking could be avoided in most cases. 

The most important thing to take away from the talk, Goel said, is that as long as the integrity of written block data is assured, all of the ext3 properties that can checked by `fsck` can instead be done at runtime. As Ric Wheeler and others in the audience pointed out, that doesn't eliminate the need for an offline checker, but it may help reduce how often it's needed. Goel agreed with that, and noted that in 4% of their tests with corrupted filesystems, `fsck` would complete successfully, but that a second run would find more things to fix. Ts'o was very interested to hear that and asked that they file bugs for those cases. 

There is ongoing work on additional consistency invariants as well as things like reducing the memory overhead and increasing the number of filesystems that are covered. Dave Chinner noted that invariants for some filesystems may be hard to come up with, especially for filesystems like XFS that don't necessarily do metadata updates through the page cache. 

The reaction to Recon was favorable overall. It is an interesting project and surprised some that it was possible to do runtime consistency checking at all. As always, there is more to do, and the team has limited resources, but most attendees seemed favorably impressed with the work. 

[Many thanks are due to Mel Gorman for sharing his notes from this session.]
