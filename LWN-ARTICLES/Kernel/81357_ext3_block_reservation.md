---
title: ext3 block reservation
url: https://lwn.net/Articles/81357/
date: "April 20, 2004"
category: "Filesystems-ext3"
author: 
---

Like most modern filesystems, ext3 tries to lay out files contiguously on the disk. This layout allows files to be read and written quickly, without a lot of disk head seeks in the middle. This strategy can be thwarted, however, by the fact that ext3 allocates blocks as they are actually needed by a file. By the time a file requests a new block, the space immediately after the file on disk may well have been allocated for some other file. At that point, a contiguous allocation will be impossible. 

Mingming Cao has attempted to fix this problem with a set of "block reservation" patches for ext3; those patches are currently part of the -mm tree. The core idea behind these patches is that the filesystem should think ahead of time about where it might place blocks for growing files and reserve that space. That way, when the file does grow, there will be blocks available in a useful part of the disk. 

To that end, the ext3 block allocator has been replaced by a reservation-oriented version. The first time a block is needed for a file, the filesystem creates a "reservation window" which sets aside a range of blocks (eight of them, initially); the actual block allocations are then taken from the window. When the window is exhausted, a new, possibly expanded window is allocated, as near as possible to the old window, to replace it. Reservations only last until the process writing the file closes it; thereafter, the blocks become free once again. 

Interestingly, nothing in the filesystem itself tracks block reservations; they are all handled by a single, in-core linked list (per filesystem). A block reservation will not actually prevent blocks inside the window from being allocated to some other file. Since the filesystem allocates out of reservation windows whenever possible, however, and those windows do not overlap, the reservations are almost always honored. In some situations (such as when all remaining free blocks are reserved) the filesystem will forget about reservations and allocate blocks from anywhere. 

Some [benchmark results](<https://lwn.net/Articles/81359/>) show significant performance improvements, especially when large numbers of processes are running. To some extent, this improvement comes about because block reservations narrow down the area of the disk that must be searched for free blocks and increase the chances that a block will be found quickly. The real benefit, however, is that the on-disk layout of the files is much improved. Unless problems turn up, this patch may find its way into the mainline fairly quickly.
