---
title: Online defragmentation for ext4
url: https://lwn.net/Articles/317787/
date: "February 4, 2009"
category: "Filesystems-ext4"
author: "By Jonathan Corbet February 4, 2009"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
February 4, 2009

Any filesystem designed for use with rotating media must pay careful attention to the layout of files on the disk. If a file's blocks can be placed sequentially on the device, they can be read or written as a unit, without the need for performance-destroying head seeks in the middle. Even the most careful filesystem will sometimes fail to lay out files in a minimal number of contiguous extents, though. If a file grows, for example, and the blocks just past the previous end are not available, the filesystem has no choice other than placing the new blocks somewhere else. Depending on how full the filesystem is, those blocks could end up far away indeed. This sort of fragmentation can result in filesystems slowing down over time. 

Fragmentation problems can be fixed up after the fact. The most obvious way to defragment a disk is to make a new filesystem on it; after all, empty filesystems tend not to have fragmentation problems. But the new filesystem will have less fragmentation even after its old contents have been restored onto it. When the ultimate size of every file is known in advance, it's relatively easy to make good layout decisions. Knowing this, system administrators have used backup-and-restore cycles as a way of cleaning up overly fragmented disks for many years. 

There is, of course, a problem with this approach which goes beyond the risk of discovering that one's backup is not quite as good as one had thought. The downtime associated with rewriting a disk can be unwelcome to users; a filesystem which is down responds even more slowly than a filesystem with fragmentation problems. So it would be nice to have a way to defragment a filesystem while keeping it online and available. This online defragmentation capability has been on the ext4 "planned features" list for a long time; it is, at this point, about the only planned feature which has not yet been merged into the mainline. 

Some attempts at online defragmentation have been made in the past, but they have not, yet, gotten through review. Now Akira Fujita has come forward with [a new ext4 online defragmentation patch](<http://lwn.net/Articles/317584/>) which, by virtue of a different view of the problem, might just make it into the mainline. Previous attempts exposed an interface whereby a user-space application could ask the filesystem to defragment a specific file by allocating new (contiguous) blocks to it. That turned out to be a bit too much work to put into the kernel; so, with this patch, Akira has created an interface which moves a bit more of the work into user space. 

In the new scheme, a user-space defragmentation daemon will pick a file which, in its opinion, is too spread out on the disk. The daemon will then set about creating a new, less-fragmented file to replace it. That is done by creating a new, temporary file on the same filesystem, then unlinking it (while holding the file descriptor open). Calls to `fallocate()` can then be used to add the requisite number of blocks to the new file. Once the new file is up to the correct size, the daemon can use the [`FS_IOC_FIEMAP` `ioctl()`](<http://lwn.net/Articles/260795/>) to query the number of extents (fragments) it contains. If the new file is not an improvement over the old one, the daemon should just close it and give up; the filesystem simply does not have enough contiguous storage available. 

The daemon could, at this point, simply copy the old file into the new one, then put the newly defragmented version in the place of the old one. The problems with that approach include performance (all that data must be copied through user space) and robustness. If some other process changes the file while the copy is happening, the new file may lose those changes. Indeed, if some process has the old file open, it may never notice that the replacement has happened. So something smarter is needed. 

Akira's patch addresses these problems with the creation of a new, magic `ioctl()` call for ext4. The defragmentation application must fill out a structure like: 

```
struct move_extent {
    	int org_fd;		/* original file descriptor */
    	int dest_fd;		/* destination file descriptor */
    	ext4_lblk_t start;	/* logical offset of org_fd and dest_fd*/
    	ext4_lblk_t len;	/* exchange block length */
        };
```

This structure, when passed to the new `EXT4_IOC_DEFRAG` `ioctl()`, expresses a request to the kernel to move `len` blocks from the original file to the new one, starting at `start`. Essentially, it copies an extent's worth of data into the (fully allocated, nicely contiguous) space in the new file, then performs a magic block swap. The contiguous blocks from the new file are patched into the old file, while the fragmented blocks are, instead, put into the new file. Once the entire file has been treated in this way, the file will have been defragmented without having been visibly moved. 

The final step is to delete the "new" file, which now contains the "old" file's blocks. Since the file had been unlinked, that will cause the filesystem to recover the old blocks and the task will be complete. For the curious, Akira has posted [the source for a user-space defragmentation tool](<https://lwn.net/Articles/317803/>) which shows how this interface can be used. 

There have not been a whole lot of objections to the new code. Chris Mason did [point out](<https://lwn.net/Articles/317801/>) that the system will do unfortunate things if the layout of a swap file changes. He has clearly thought about the problem - to an extent: 

Btrfs is currently getting around this by dropping bmap support, so swapfiles on btrfs won't work at all. A real long term solution is required ;) 

Beyond that, there are some minor issues, such as the definition of the ABI in terms of types like `int` instead of architecture-independent types. Requests for separate source and destination block numbers have been made; that feature would help developers working on hierarchical storage systems. The ability to guide the allocation of blocks would be useful in situations where performance can be improved by grouping related files together on the disk. 

There could also be value in finding a way to move much of this functionality into the VFS layer where it could be used with other filesystems as well; that could prove to be a difficult task, though, and ext4 maintainer Ted Ts'o [has little desire](<https://lwn.net/Articles/317911/>) to take on that job. 

Those little issues notwithstanding, it does appear that the ext4 filesystem may be closer to getting the much-requested online defragmentation feature.
