---
title: "A tale of two data-corruption bugs"
url: https://lwn.net/Articles/645720/
date: "May 24, 2015"
category: "Filesystems-ext4; RAID"
author: "By Jonathan Corbet May 24, 2015"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
May 24, 2015

There have been two bugs causing filesystem corruption in the news recently. One of them, a bug in ext4, has gotten the bulk of the attention, despite the fact that it is an old bug that is hard to trigger. The other, however, is recent and able to cause data loss on filesystems installed on a RAID 0 array. Both are interesting examples of how things can go wrong, and, thus, merit a closer look. 

#### Extent confusion in ext4

Like many reasonably modern filesystems, ext4 includes a number of performance-enhancing features. One of those is delayed allocation, wherein the filesystem will not immediately allocate specific blocks on the disk for data that an application has just written. By delaying that allocation, the filesystem gives itself some time to see if more data will be written in the near future. If so, space for all of the written data can be allocated contiguously, improving future I/O performance. Delayed allocation works, but it does leave some written data (the "delayed extent") in a sort of limbo state for a brief period where it has no fixed home on the disk. Obviously, when the time comes to flush that data out to persistent storage, the task of allocating the destination blocks can be delayed no further. 

Another performance feature is unwritten extents. An application can increase the size of a file with system calls like `truncate()` or `fallocate()`. These calls do not actually write data to any new extents added to the file. Allowing anybody to read blocks that have not been written to is clearly a bad idea; at best, the result will be garbage, while, at worst, another user's sensitive information could be disclosed. The filesystem could avoid this problem by writing zeroes to all new blocks once they are allocated, but that's an inefficient use of CPU time and I/O bandwidth, given that the blocks are ordinarily going to be overwritten with real data in the near future. The alternative is to mark the new blocks as being explicitly unwritten until that real data comes along. Attempts to read unwritten blocks will just result in a zero-filled buffer. 

Ext4 keeps track of both delayed and unwritten extents in a data structure called the "extent status tree". Something interesting happens, though, if an unwritten extent is added when there are already delayed allocation blocks in the same block range. The entire unwritten extent ends up being marked as delayed as well, because the extent status tree can't track the fact that only part of the extent is delayed. 

For example, consider a file that is currently 100 blocks long — blocks 0-99 are written and present on disk. The application writes blocks 100 and 101; the filesystem responds by putting them into the extent status tree as delayed-allocation blocks. The application then uses `fallocate()` to tell the filesystem to allocate blocks 100-109. Those unwritten blocks also go into the tree, but, since there are already two delayed blocks in that range, the entire range 100-109 is marked delayed as well. 

A delayed extent is removed from the tree when the delayed buffers are actually written to disk. But, in this case, there _are_ no delayed buffers for blocks 102-109; as a result, the extent remains as a delayed extent in the tree, even after the actual delayed portion (blocks 100 and 101) has been allocated and written out. There it will stay until another write to one of the affected blocks comes along. At that point, the entire block will be reallocated (because it is still marked as delayed), losing the previously delayed data that had already been written. That is about the point where alcohol consumption by both administrators and users increases unhealthily. 

This bug has been present in the ext4 filesystem for some time; nobody seems to be quite sure when it was introduced. It has remained undetected because it is quite hard to hit; the process, as [described](<https://lwn.net/Articles/645723/>) by Ted Ts'o, is: 

It requires the combination of (a) writing to a portion of a file that was not previously allocated using buffered I/O, (b) an fallocate of a region of the file which is a superset of region written in (a) before it has chance to be written to disk, (c) waiting for the file data in (a) to be written out to disk (either via fsync or via the writeback daemons), and then (d) before the extent status cache gets pushed out of memory, another random write to a portion of the file covered by (a) -- in which case that specific portion of (a) could be replaced by all zeros. 

Nonetheless, corruption bugs are bad news. This one has been fixed by [this patch](<https://lwn.net/Articles/645722/>) from Lukas Czerner which was merged for 4.1-rc2. The fix also found its way into the 4.0.3, 3.18.14, 3.14.42, and 3.10.78 stable updates. 

#### Discard discrepancies

About the time the above bug was being fixed, some users started reporting problems with RAID 0 arrays based on ext4; many assumed that they had been a victim of that bug. The truth of the matter was somewhat worse than that, though; they had found a nastier, easier-to-trigger bug that was the result of an overly hasty fix. 

Back in April, Joe Landman [reported](<https://lwn.net/Articles/645725/>) a problem with RAID 0 volumes on the XFS filesystem. After some back-and-forth, Neil Brown [tracked it down](<https://lwn.net/Articles/645726/>) to [a change](<http://git.kernel.org/linus/20d0189b1012a37d2533a87fb451f7852f2418d1>) merged for the 3.14 release. The code in question calculates the number of sectors that fit in the next RAID 0 chunk — in other words, the portion of the I/O request that maps to a single underlying drive. In simplified form, this calculation looks like: 

```
unsigned sectors = chunk_sects - sector_div(sector, chunk_sects);
```

The call to `sector_div()` returns the remainder of the division. What it also does, though, may be surprising: it replaces the value of `sector` with `sector/chunk_sects`. In other words, `sector_div()` is a macro that modifies one of its arguments. The code did not take that modification into account, with the result that it used the wrong value of `sector` from then on. Neil's fix was to simply reinitialize `sector` from the `bio` structure describing the operation in question. 

There was only one little problem: by then the `bio` pointer had been advanced, so the new `sector` value was wrong. When the RAID 0 code then proceeded to map the sector in the RAID device to a sector in one of the underlying physical devices, it would end up in the wrong place — likely as not, on the wrong device entirely. In practice, the code path that goes wrong in this way would be executed relatively rarely; it requires an I/O operation that crosses multiple RAID 0 chunks. So it is perhaps not entirely surprising that it seems to manifest itself most often with "discard" requests, which can be applied to an entire file at once. 

Discard is a mechanism for telling a storage device that a particular range of blocks no longer contains needed data. Its use can improve performance on solid-state drives, which can benefit from the knowledge that a certain range of blocks does not need to be preserved during wear-leveling operations. If told to do so, the ext4 filesystem will generate DISCARD requests when a file is deleted, and the RAID 0 code will pass those requests down to the underlying drives. When this particular bug hits, though, the discard request will go to the wrong place, resulting in the discarding of some random, unrelated data. 

This unfortunate "fix" was merged into the mainline during the 4.1 merge window. If it had stayed with the 4.1 kernel, its impact would have been limited; 4.1 has still not seen an official release. But this patch also went into the 4.0.2, 3.19.8, 3.18.14, and 3.14.41 stable updates. The [fix to the fix](<http://git.kernel.org/linus/a81157768a00e8cf8a7b43b5ea5cac931262374f>) (written by Eric Work) has been pulled into the mainline kernel, but has not, as of this writing, found its way into any stable updates. One assumes that will happen soon, but it is worth noting that 3.19.8 is the end of the 3.19 stable series, so there may be no updated kernel for 3.19 users. 

The good news is that the problem was caught reasonably quickly; there should not be huge numbers of users who have updated to one of the affected kernels. The bad news is that, for those users who are affected, there could be silent data corruption that will not be discovered for some time. Anybody who is running one of the affected kernels will — after moving to a safe kernel, of course — want to check the contents of their RAID 0 arrays against a backup. 

Keeping data safely is one of the fundamental obligations of an operating system kernel, so data-corruption bugs can shake one's confidence in the whole structure. But, as has been seen here, bugs happen. Sometimes they lurk for years without causing trouble, and sometimes they make their presence known quickly. Such bugs are, fortunately, rare with Linux; with luck, these are the last we will see for a while.
