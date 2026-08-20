---
title: "Linux Storage and Filesystem Workshop, day 2"
url: https://lwn.net/Articles/327740/
date: "April 8, 2009"
category: "Block layer; Filesystems-Workshops"
author: "By Jonathan Corbet April 8, 2009"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
April 8, 2009

The second and final day of the Linux Storage and Filesystem Workshop was held in San Francisco, California on April 7\. Conflicting commitments kept your editor from attending the entire event, but he was able to participate in sessions on solid-state device support, storage topology information, and more. 

### Supporting SSDs

The solid-state device topic was the most active discussion of the morning. SSDs clearly stand to change the storage landscape, but it often seems that nobody has yet figured out just how things will change or what the kernel should do to make the best use of these devices. Some things are becoming clearer, though. The kernel will be well positioned to support the current generation SSDs. Supporting future products, though, is going to be a challenge. 

[![\[Group photo\]](https://static.lwn.net/images/conf/lsf2009/lsf2009-group-sm.jpg)](<https://lwn.net/Articles/327842/>) Matthew Wilcox, who led the discussion, started by noting that Intel SSDs are able to handle a large number of operations in parallel. The parallelism is so good, in fact, that there is really little or no advantage in delaying operations. I/O requests should be submitted immediately; the block I/O subsystem shouldn't even attempt to merge adjacent requests. This message was diluted a bit later on, but the core message is clear: the kernel should, when driving an SSD, focus on getting out of the way and processing operations as quickly as possible. 

It was asked: how do these drives work internally? This would be nice to know; the better informed the kernel developers are, the better they can do at driving the devices better. It seems, though, that the firmware in these devices - the part that, for now, makes Intel devices work better than most of the alternatives - is laden with Valuable Intellectual Property, and not much information will be forthcoming. Solid-state devices will be black boxes for the foreseeable future. 

In any case, current-generation Intel SSDs are not the only type of device that the kernel will have to work with. Drives will differ greatly in the coming years. What the kernel really needs to know is a few basic parameters: what kind of request alignment works best, what request sizes are fastest, etc. It would be nice if the drives could export this information to the operating system. There is a mechanism by which this can be done, but current drives are not making much information available. 

One clear rule holds, though: bigger requests are better. They might perform better in the drive itself, but, with high-quality SSDs, the real bottleneck is simply the number of requests which can be generated and processed in a given period of time. Bundling things into larger requests will tend to increase the overall bandwidth. 

A related rule has to do with changes in usage patterns. It would appear that the Intel drives, at least, observe the requests issued by the computer and adapt their operation to improve performance. In particular, they may look at the typical alignment of requests. As a result, it is important to let the drive know if the usage pattern is about to change - when the drive is repartitioned and given a new filesystem, for example. The way to do this, evidently, is to issue an ATA "secure erase" command. 

From there, the conversation moved to discard (or "trim") requests, which are used by the host to tell the drive that the contents of specific blocks are no longer needed. Judicious use of trim requests can help the drive in its garbage collection work, improving both performance and the overall life span of the hardware. But what constitutes "judicious use"? Doing a trim when a new filesystem is made is one obvious candidate. When the kernel initializes a swap file, it trims the entire file at the outset since it cannot contain anything of use. There is no controversy here (though it's amusing to note that mkfs does not, yet, issue trim commands). 

But what about when the drive is repartitioned? It was suggested that the portion of the drive which has been moved from one partition to another could be trimmed. But that raises an immediate problem: if the partition table has been corrupted and the "repartitioning" is really just an attempt to restore the drive to a working state, trimming that data would be a fatal error. The same is true of using trim in the fsck command, which is another idea which has been suggested. In the end, it is not clear that using trim in either case is a safe thing to do. 

The other obvious place for a trim command is when a file is deleted; after all, its data clearly is no longer needed. But some people have questioned whether that is a good time too. Data recovery is one issue; sometimes people want to be able to get back the contents of an erroneously-deleted file. But there is also a potential performance issue: on ATA drives, trim commands cannot be issued as tagged commands. So, when a trim is performed, all normal operations must be brought to a halt. If that happens too often, the throughput of the drive can suffer. This problem could be mitigated by saving up trim operations and issuing them all together every few minutes. But it's not clear that the real performance impact is enough to justify this effort. So some benchmarking work will be needed to try to quantify the problem. 

An alternative which was suggested was to not use trim at all. Instead, a similar result could be had by simply reusing the same logical block numbers over and over. A simple-minded implementation would always just allocate the lowest-numbered free block when space is needed, thus compressing the data toward the front end of the drive. There are a couple of problems with this approach, though, starting with the fact that a lot of cheaper SSDs have poor wear-leveling implementations. Reusing low-numbered blocks repeatedly will wear those drives out prematurely. The other problem is that allocating blocks this way would tend to fragment files. The cost of fragmentation is far less than with rotating storage, but there is still value in keeping files contiguous. In particular, it enables larger I/O operations, and, thus, better performance. 

There was a side discussion on how the kernel might be able to distinguish "crap" drives from those with real wear-leveling built in. There's actually some talk of trying to create value-neutral parameters which a drive could use to export this information, but there doesn't seem to be much hope that the vendors will ever get it right. No drive vendor wants its hardware to self-identify as a lower-quality product. One suggestion is that the kernel could interpret support for the trim command as an indicator that it's dealing with one of the better drives. That led to the revelation that the much-vaunted Intel drives do not, currently, support trim. That will change in future versions, though. 

A related topic is a desire to let applications issue their own trim operations on portions of files. A database manager could use this feature to tell the system that it will no longer be interested in the current contents of a set of file blocks. This is essentially a version of the long-discussed `punch()` system call, with the exception that the blocks would remain allocated to the file. De-allocating the blocks would be correct at one level, but it would tend to fragment the file over time, force journal transactions, and make `O_DIRECT` operations block while new space is allocated. Database developers would like to avoid all of those consequences. So this variant of `punch()` (perhaps actually a variant of `fallocate()`) would discard the data, but keep the blocks in place. 

From there, the discussion went to the seemingly unrelated topic of "thin provisioning." This is an offering from certain large storage array vendors; they will sell an array which claims to be much larger than the amount of storage actually installed. When the available space gets low, the customer can buy more drives from the vendor. Meanwhile, from the point of view of the system, the (apparently) large array has never changed. 

Thin provisioning providers can use the trim command as well; it lets them know that the indicated space is unused and can be allocated elsewhere. But that leads to an interesting problem if trim is used to discard the contents of some blocks in the middle of the file. If the application later writes to those blocks - which are, theoretically, still in place - the system could discover that the device is out of space and fail the request. That, in turn, could lead to chaos. 

The truth of the matter is that thin provisioning has this problem regardless of the use of the trim command. Space "allocated" with `fallocate()` could turn out to be equally illusory. And if space runs out when the filesystem is trying to write metadata, the filesystem code is likely to panic, remount the filesystem read-only, and, perhaps, bring down the system. So thin provisioning should be seen as broken currently. What's needed to fix it is a way for the operating system to tell the storage device that it intends to use specific blocks; this is an idea which will be taken back to the relevant standards committees. 

Finally, there was some discussion of the CFQ I/O scheduler, which has a lot of intelligence which is not needed for SSDs. There's a way to bypass CFQ for some SSD operations, but CFQ still adds an approximately 3% performance penalty compared to the no-op I/O scheduler. That kind of cost is bearable now, but it's not going to work for future drives. There is real interest in being able to perform 100,000 operations per second - or more - on an SSD. That kind of I/O rate does not leave much room for system overhead. So, at some point, we're going to see a real effort to streamline the block I/O paths to ensure that Linux can continue to get the best out of solid-state devices. 

### Storage topology

Martin Petersen introduced the storage topology issue by talking about the coming 4K-sector drives. The sad fact is that, for all the talk of SSDs, rotating storage will be with us for a while yet. And the vendors of disk drives intend to shift to 4-kilobyte sectors by 2011. That leads to a number of interesting support problems, most of which were covered in [this LWN article](<http://lwn.net/Articles/322777/>) in March. In the end, the kernel is going to have to know a lot more about I/O sizes and alignment requirements to be able to run future drives. 

To that end, Martin has prepared a set of patches which export this information to the system. The result is a set of directories under `/sys/block/_drive_ /topology` which provide the sector size, needed alignment, optimal I/O flag, and more. There's also a "consistency flag" which tells the user whether any of the other information actually matches reality. In some situations (a RAID mirror made up of drives with differing characteristics, for example), it is not possible to provide real information, so the kernel has to make something up. 

There was some wincing over this use of sysfs, but the need for this kind of information is clear. So these patches will probably be merged into the 2.6.31 kernel. 

### readdirplus()

There was also a session on the proposed `readdirplus()` system call. This call would function much like `readdir()` (or, more likely, like `getdents()`), but it would provide file metadata along with the names. That, in turn, would avoid the need for a separate `stat()` call and, hopefully, speed things considerably in some situations. 

Most of the discussion had to do with how this new system call would be implemented. There is a real desire to avoid the creation of independent `readdir()` and `readdirplus()` implementations in each filesystem. So there needs to be a way to unify the internal implementation of the two system calls. Most likely that would be done by using only the `readdirplus()` function if a filesystem provides one; this callback would have a "no stat information needed" flag for the case when normal `readdir()` is being called. 

The creation of this system call looks like an opportunity to leave some old mistakes behind. So, for example, it will not support seeking within a directory. There will also probably be a new `dirent` structure with 64-bit fields for most parameters. Beyond that, though, the shape of this new system call remains somewhat cloudy. Somebody clearly needs to post a patch. 

### Conclusion

And there ends the workshop - at least, the part that your editor was able to attend. There were a number of storage-related sessions which, beyond doubt, covered interesting topics, but it was not possible to be in both rooms at the same time (though, with luck, your editor will soon receive another attendee's notes from those sessions). The consensus among the attendees was that it was a highly successful and worthwhile event; the effects should be seen to ripple through the kernel tree over the next year.
