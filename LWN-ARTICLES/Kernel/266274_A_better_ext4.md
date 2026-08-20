---
title: A better ext4
url: https://lwn.net/Articles/266274/
date: "January 23, 2008"
category: "Filesystems-ext4"
author: "By Jonathan Corbet January 23, 2008"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
January 23, 2008

[Last week's Kernel Page](<http://lwn.net/Articles/264521/>) may have been filesystem-heavy, but there was still a big omission, in the form of ext4. But ext4, being the successor to ext3, may well be the filesystem many of us are using a few years from now. Things have been relatively quiet on that front - at least, outside of the relevant mailing lists - but the ext4 developers have not been idle. Some of their work has now come to the surface with Ted Ts'o's posting of the [ext4 merge plans](<http://lwn.net/Articles/266191/>) for 2.6.25. 

One of the changes going into ext4 is a lifting of the longstanding 4KB block size limit. That does not mean that just any block size works, though, and this feature will benefit fewer people than one might think, for one specific reason: the block size must still be no larger than the page size on the host system. So those of us running x86 systems with 4KB pages will be stuck with 4KB blocks still. And, on any system, the maximum block size is now 64KB. 

One amusing effect of this change is that the size of a directory entry can now be as large as 64KB as well. But the field which holds the size of directory entries is only 16 bits wide. So a special hack has been employed to recognize 64KB directory entries and keep everything consistent. 

Some internal variables have overflow problems as well. Block numbers are stored as a signed, 32-bit quantity, and so are block group numbers. That limits the maximum size of a filesystem to a mere 256PB. In 2.6.25, these values will become unsigned long variables, eliminating that intolerably low limit. Through some trickery, the inode field which stores the number of blocks associated with a file will be expanded to 48 bits, raising the maximum size of an individual file to just under 248 512-byte blocks. 

The work does not stop there, though: another patch redefines that field to mean the number of filesystem blocks (instead of 512-byte sectors) used by the file. This is a change which has to be handled carefully, since it is an on-disk format change which could create trouble for people with existing ext4 filesystems. Everybody who is using ext4 should certainly be doing so with the knowledge that it's a development filesystem and is only suitable for storing files which are not valuable for more than about 30 minutes - Rawhide OpenOffice.org updates, say. But it still would be nice to not trash every existing ext4 filesystem out there. So the `i_blocks` field will continue, by default, to hold the number of 512-byte blocks. But, if that field exceeds 32 bits and forces the use of 48-bit numbers, it is thereafter interpreted as filesystem blocks. Since no existing filesystems are yet using 48-bit numbers, this approach successfully avoids breaking them. 

Journal checksums are another feature arriving for 2.6.25. If the system crashes, the journal is used to recover any transactions which were committed, but which did not actually make it to disk. It sure would be nice to know that the journal, as stored in the filesystem, is intact before using it to make changes elsewhere. The checksum enables the filesystem to ensure that the journal is good and avoid (further) corrupting the filesystem if it is not. An interesting side benefit is that the checksum loosens the constraints on how the journal is written to disk, since an incompletely-written journal will now be detected; that should help to improve filesystem performance slightly. 

Note that full data checksumming is still not on the agenda for ext4. But checksumming the journal is a good (if small) step in the right direction. 

Another change is a VFS API change, in that it turns the `i_version` field of the `inode` structure into an unsigned, 64-bit value on all architectures. This version number is incremented when the file is changed, and it's stored (split into two fields) in the on-disk inode. 64-bit version numbers are required by NFSv4, which uses them to provide the dreaded "stale file handle" error when things change. 

There is a new `ioctl()` (`EXT4_IOC_MIGRATE`) which can be used to explicitly request that the on-disk inode for a file be converted to the ext4 format. 

The ext4 filesystem is extent-based, and has been for some time. "Extent-based" means that it tracks block allocations by extents (first block, number of blocks) rather than storing pointers to each individual block, as is done in ext3. There are a number of performance benefits to doing things this way, especially for larger files. Those benefits disappear, though, if a file's blocks cannot be grouped into the smallest number of extents possible. 

One technique which greatly helps in optimizing block allocations for files is to allocate them in relatively large groups, rather than individually. In 2.6.25, ext4 will contain the multi-block allocator, which does exactly that. One might think that allocating a few blocks at a time would not be that big of a change, but the multi-block allocator is by far the most complex patch in the set. A lot of effort and heuristics go into deciding how many blocks to allocate, finding the optimal set of blocks, tracking the allocation, recovering blocks which end up never being used, ensuring that an application cannot read pre-allocated (but unwritten) blocks in search of leaked secrets, etc. It is quite a bit of code, but it is worth the trouble; multi-block allocation will be enabled by default in 2.6.25. 

As noted above, a number of these patches force changes to the on-disk data structure. According to Ted, though, these should be the last on-disk changes for ext4. There are some features which still will not have been merged when 2.6.25 comes around - delayed allocation and online defragmentation among them - but they should not require format changes. So ext4 is getting closer to the point where it is considered ready for production use. 

It is _not_ at that point yet, though, and people who use it are still doing so at their own risk. To help drive that point home, Ted has proposed [a new mount flag](<http://lwn.net/Articles/266269/>) (called `test_fs`) which communicates to the kernel the user's understanding that they are about to mount a developmental filesystem and will not go filing lawsuits if things go wrong. In the absence of this mount option, an ext4 filesystem will refuse to mount. One might think that child-proofing the filesystem in this way would not be necessary, but some extra care in this area can only be a good thing. Filesystem-related surprises are rarely welcome.
