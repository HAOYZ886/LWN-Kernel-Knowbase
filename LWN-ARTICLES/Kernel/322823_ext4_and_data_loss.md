---
title: ext4 and data loss
url: https://lwn.net/Articles/322823/
date: "March 11, 2009"
category: "Filesystems-ext4"
author: "By Jonathan Corbet March 11, 2009"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
March 11, 2009

The ext4 filesystem offers a number of useful features. It has been stabilizing quickly, but that does not mean that it will work perfectly for everybody. Consider this example: Ubuntu's bug tracker contains [an entry titled "ext4 data loss"](<https://bugs.edge.launchpad.net/ubuntu/+source/linux/+bug/317781>), wherein a luckless ext4 user reports: 

Today, I was experimenting with some BIOS settings that made the system crash right after loading the desktop. After a clean reboot pretty much any file written to by any application (during the previous boot) was 0 bytes. 

Your editor had not intended to write (yet) about this issue, but quite a few readers have suggested that we take a look at it. Since there is clearly interest, here is a quick look at what is going on. 

Early Unix (and Linux) systems were known for losing data on a system crash. The buffering of filesystem writes within the kernel, while being very good for performance, causes the buffered data to be lost should the system go down unexpectedly. Users of Unix systems used to be quite aware of this possibility; they worried about it, but the performance loss associated with synchronous writes was generally not seen to be worth it. So application writers took great pains to ensure that any data which really needed to be on the physical media got there quickly. 

More recent Linux users may be forgiven for thinking that this problem has been entirely solved; with the ext3 filesystem, system crashes are far less likely to result in lost data. This outcome is almost an accident resulting from some decisions made in the design of ext3. What's happening is this: 

  * By default, ext3 will commit changes to its journal every five seconds. What that means is that any filesystem _metadata_ changes will be saved, and will persist even if the system subsequently crashes. 

  * Ext3 does not (by default) save data written to files in the journal. But, in the (default) `data=ordered` mode, any modified data blocks are forced out to disk before the metadata changes are committed to the journal. This forcing of data is done to ensure that, should the system crash, a user will not be able to read the previous contents of the affected blocks - it's a security feature. 

  * The end result is that `data=ordered` pretty much guarantees that data written to files will actually be on disk five seconds later. So, in general, only five seconds worth of writes might be lost as the result of a crash. 

In other words, ext3 provides a relatively high level of crash resistance, even though the filesystem's authors never guaranteed that behavior, and POSIX certainly does not require it. As Ted put it in [his excruciatingly clear and understandable explanation of the situation](<https://bugs.edge.launchpad.net/ubuntu/+source/linux/+bug/317781/comments/45>): 

Since ext3 became the dominant filesystem for Linux, application writers and users have started depending on this, and so they become shocked and angry when their system locks up and they lose data --- even though POSIX never really made any such guarantee. 

Accidental or not, the avoidance data loss in a crash seems like a nice feature for a filesystem to have. So one might well wonder just what would have inspired the ext4 developers to take it away. The answer, of course, is performance - and delayed allocation in particular. 

"Delayed allocation" means that the filesystem tries to delay the allocation of physical disk blocks for written data for as long as possible. This policy brings some important performance benefits. Many files are short-lived; delayed allocation can keep the system from writing fleeting temporary files to disk at all. And, for longer-lived files, delayed allocation allows the kernel to accumulate more data and to allocate the blocks for data contiguously, speeding up both the write and any subsequent reads of that data. It's an important optimization which is found in most contemporary filesystems. 

But, if blocks have not been allocated for a file, there is no need to write them quickly as a security measure. Since the blocks do not yet exist, it is not possible to read somebody else's data from them. So ext4 will not (cannot) write out unallocated blocks as part of the next journal commit cycle. Those blocks will, instead, wait until the kernel decides to flush them out; at that point, physical blocks will be allocated on disk and the data will be made persistent. The kernel doesn't like to let file data sit unwritten for too long, but it can still take a minute or so (with the default settings) for that data to be flushed - far longer than the five seconds normally seen with ext3. And that is why a crash can cause the loss of quite a bit more data when ext4 is being used. 

The real solution to this problem is to fix the applications which are expecting the filesystem to provide more guarantees than it really is. Applications which frequently rewrite numerous small files seem to be especially vulnerable to this kind of problem; they should use a smarter on-disk format. Applications which want to be sure that their files have been committed to the media can use the `fsync()` or `fdatasync()` system calls; indeed, that's exactly what those system calls are for. Bringing the applications back into line with what the system is really providing is a better solution than trying to fix things up at other levels. 

That said, it would be nice to improve the robustness of the system while we're waiting for application developers to notice that they have some work to do. One possible solution is, of course, to just run ext3. Another is to shorten the system's writeback time, which is stored in a couple of sysctl variables: 

```
/proc/sys/vm/dirty_expire_centisecs
        /proc/sys/vm/dirty_writeback_centisecs
```

The first of these variables (`dirty_expire_centiseconds`) controls how long written data can sit in the page cache before it's considered "expired" and queued to be written to disk; it defaults to 30 seconds. The value of `dirty_writeback_centiseconds` (5 seconds, default) controls how often the `pdflush` process wakes up to actually flush expired data to disk. Lowering these values will cause the system to flush data to disk more aggressively, with a cost in the form of reduced performance. 

A third, partial solution exists in a set of patches queued for 2.6.30; they add a set of heuristics which attempt to protect users from being badly burned in certain situations. They are: 

  * [A patch](<http://git.kernel.org/?p=linux/kernel/git/tytso/ext4.git;a=commitdiff;h=3bf3342f394d72ed2ec7e77b5b39e1b50fad8284>) adding a new `EXT4_IOC_ALLOC_DA_BLKS` `ioctl()` command. When issued on a file, it will force ext4 to allocate any delayed-allocation blocks for that file. That will have the effect of getting the file's data to disk relatively quickly while avoiding the full cost of the (heavyweight) `fsync()` call. 

  * [The second patch](<http://git.kernel.org/?p=linux/kernel/git/tytso/ext4.git;a=commitdiff;h=6645f8c3bc3cdaa7de4aaa3d34d40c2e8e5f09ae>) sets a special flag on any file which has been truncated; when that file is closed, any delayed allocations will be forced. That should help to prevent the "zero-length files" problem reported at the beginning. 

  * Finally, [this patch](<http://git.kernel.org/?p=linux/kernel/git/tytso/ext4.git;a=commitdiff;h=dbc85aa9f11d8c13c15527d43a3def8d7beffdc8>) forces block allocation when one file is renamed on top of another. This, too, is aimed at the problem of frequently-rewritten small files. 

Together, these patches should mitigate the worst of the data loss problems while preserving the performance benefits that come with delayed allocation. They have not been proposed for merging at this late stage in the 2.6.29 release cycle, though; they are big enough that they will have to wait for 2.6.30. Distributors shipping earlier kernels can, of course, backport the patches, and some may do so. But they should also note the lesson from this whole episode: ext4, despite its apparent stability, remains a very young filesystem. There may yet be a surprise or two waiting to be discovered by its early users.
