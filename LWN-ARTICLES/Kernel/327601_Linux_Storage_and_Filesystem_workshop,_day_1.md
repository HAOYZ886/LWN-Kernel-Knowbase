---
title: "Linux Storage and Filesystem workshop, day 1"
url: https://lwn.net/Articles/327601/
date: "April 7, 2009"
category: "Block layer; Filesystems-Workshops"
author: "By Jonathan Corbet April 7, 2009"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
April 7, 2009

The annual Linux kernel summit may gain the most attention, but the size of the kernel community makes it hard to get deeply into subsystem-specific topics at that event. So, increasingly, kernel developers gather for more focused events where some real work can be done. One of those gatherings is the Linux Storage and Filesystem workshop; the [2009 workshop](<http://events.linuxfoundation.org/events/lsf-workshop>) began on April 6\. Here is your editor's summary of the discussions which took place on the first day. 

Things began with a quick recap of the action items from the previous year. Some of these had been fairly well resolved over that time; these include power management, support for object storage devices, fibre channel over Ethernet, barriers on by default in ext4, the `fallocate()` system call, and enabling relatime by default. The record for some other objectives is not quite so good; low-level error handling is still not what it could be, "too much work" has been done with I/O bandwidth controllers while nothing has made it upstream, the union filesystem problem has not been solved, etc. As a whole, a lot has been done, but a lot remains to do. 

### Device discovery

Joel Becker and Kay Sievers led a session on device discovery. On a contemporary system, device numbers are not stable across reboots, and neither are device names. So anything in the system which must work with block devices and filesystems must somehow find the relevant device first. Currently, that is being done by scanning through all of the devices on the system. That works reasonably well on a laptop, but it is a real problem on systems with huge numbers of block devices. There are stories of large systems taking hours to boot, with the bulk of that time being spent scanning (repeatedly - once for every mount request) through known devices. 

What comes out of the discussion, of course, is that user space needs a better way to locate devices. A given program may be searching for a specific filesystem label, UUID, or something else; a good search API would support all of these modes and more. What would be best would be to build some sort of database where each new device is added at discovery time. As additional information becomes available (when a filesystem label is found, for example), it is added to the database. Then, when a specific search is done, the information has already been gathered and a scan of the system's devices is no longer necessary. 

In the simplest form, this database can be the various directories full of symbolic links that udev creates now. These directories solve much of the problem, but they can never be a complete solution for one reason: some types of devices - iSCSI targets, for example - do not really exist for the system until user space has connected to them. Multipath devices also throw a spanner into that works. For this reason, Ted Ts'o asserted that some sort of programmatic API will always be needed. 

Not a lot of progress was made toward specifying a solution; the main concern, seemingly, was coming to a common understanding of the problem. What's likely to happen is that the libblkid library will be extended to provide the needed functionality. Next year, we'll see if that has been done. 

### Asynchronous and direct I/O

Zach Brown's stated purpose in this session was to "just rant for 45 minutes" about the poor state of asynchronous I/O (AIO) support in Linux. After ten years, he says, we still have an inadequate system which has never been fixed. The problems with Linux AIO are well documented: only a few operations are truly asynchronous, the internal API is terrible, it does not properly support the POSIX AIO API, etc. There, Zach says, are people wanting to do a lot more with AIO than is currently supported by Linux. 

That said, various alternatives have been proposed over time but nobody ever tests them. 

The conversation then shifted for a bit; Jeff Moyer took a turn to complain about the related topic of direct I/O. It works poorly for applications, he says, its semantics are different for different filesystems, the internal I/O paths for direct I/O are completely different from those used for buffered I/O, and it is full of races and corner cases. Not a pretty picture. 

One of the biggest complications with direct I/O is the need for the system to support simultaneous direct and buffered I/O on the same file. Prohibiting that combination would simplify the problem considerably, but that is a hard thing to do. In particular, it would tend to break backups, which often want to read (in buffered mode) a file which is open for direct I/O. There was some talk of adding a new O_REALLYDIRECT mode which would lock out buffered operations, but it's not clear that the advantages would make this change worthwhile. 

Another thing that would help with direct I/O would be to remove the alignment restrictions on I/O buffers. That's a hard change to make, though; many disk controllers can only perform DMA to properly-aligned buffers. So allowing unaligned buffers would force the kernel to copy data internally, which rather defeats the purpose of direct I/O. There is one use case, though, where direct I/O might still make sense: some direct I/O users really only want to avoid filling the system page cache with their data. Using the `fadvise()` system call is arguably a better way of achieving that goal, but application developers are said to distrust it. 

All told, it seems from the discussion that there is not a whole lot to be done to improve direct I/O on Linux. 

Returning to the AIO problem, the developers discussed Zach's proposed [`acall()` API](<http://lwn.net/Articles/316806/>), which shifts blocking operations into special-purpose kernel threads. The use of threads in this manner promises a better AIO implementation than Linux has ever had in the past. But there is a cost: some core scheduler changes need to be made to support `acall()`. Among other things, there are some complexities related to transferring credentials between threads, propagating signals from AIO threads back to the original process, etc. The end result is that scheduler performance may well suffer slightly. The scheduler developers tend to be sensitive to even very small performance penalties, so there may well be pushback when `acall()` is proposed for mainline inclusion. 

The addition of `acall()` would also add a certain maintenance burden. Whenever a kernel developer makes a change to the task structure, that developer would have to think about whether the change is relevant to `acall()` and whether it would need to be transferred to or from worker threads. 

The conclusion was that `acall()` looks promising, and that the developers in the room thought that it could work. They also agreed, though, that a number of the relevant people were not in the room, so the question of whether `acall()` is appropriate for the kernel as a whole could not be answered. 

### RAID unification

The kernel currently contains two software RAID implementations, found in the MD and device mapper (DM) subsystems. Additionally, the Btrfs filesystem is gaining RAID capabilities of its own, a process which is expected to continue in the future. It is generally agreed that having three (or more) versions of RAID in the kernel is not an optimal situation. What a proper solution will look like, though, is not all that clear. 

The session on RAID unification started with this question: who thinks that block subsystem development should be happening in the device mapper layer? A single hand was raised. In general, it seems, the developers in the room had a relatively low opinion of the device mapper RAID code. It should be said, of course, that there were no DM developers present. 

What it comes down to is that the next generation of filesystems wants to include multiple device support. Plans for Btrfs include eventual RAID 6 support, but Btrfs developer Chris Mason has no interest in writing that code. It would be much nicer to use a generic RAID layer provided by the kernel. There are challenges, though. For example, a RAID-aware filesystem really wants to use different stripe sizes for data and metadata. Standard RAID, which knows little about the filesystems built on it, does not provide any such feature. 

So what would a filesystem RAID API look like? Christoph Hellwig is working on this problem, but he's not ready to deal with the filesystem problem yet. Instead, he's going to start by figuring out how to unify the MD and DM RAID code. Some of this work may involve creating a set of tables in the block layer for mapping specific regions of a virtual device onto real regions in a lower-level device. The block layer already does that - it's how partitions work - but incorporating RAID would complicate things considerably. But, once that's done, we'll be a lot closer to having a general-purpose RAID layer which can be used by multiple callers. 

The talk wandered into the area of error handling for a while. In particular, the tools Linux provides to administrators to deal with bad blocks are still not what they could be. There was talk about providing a consistent interface for reporting bad blocks - including tools for mapping those blocks back to the files that contain them - as well as performing passive scanning for bad blocks. 

The action items that came out of this discussion include the rework of in-kernel RAID by Christoph. After that, the process of trying to define filesystem-specific interfaces will begin. 

### Rename, fsync, and ponies

Prior to Ted Ts'o's session on `fsync()` and `rename()`, some joker filled the room with coloring-book pages depicting ponies. These pages reflected the sentiment that Ted has often expressed: application developers are asking too much of the filesystem, so they might as well request a pony while they're at it. 

Ted apologized to the room for his part in the implementation of the `data=ordered` mode for ext3. This mode was added as a way to improve the security of the filesystem, but it had the side effect of flushing many changes to the filesystem within a five-second window. That allowed application developers to "get lazy" and stop worrying about whether their data had actually hit the disk at the right times. Now those developers are resisting the idea that they should begin to worry again. 

This problem has a longer history than many people realize. The XFS filesystem first hit it back around 2001. But, Ted says, most application developers didn't understand why they were getting corrupted files after a crash. Rather than fix their applications, they just switched filesystems \- to ext3. Things worked for some time until Ubuntu users started testing the alpha "Jaunty" release, which ~~uses ext4 by default~~ makes ext4 available as an installation option. At that point, they started finding zero-length files after crashes, and they blamed ext4. 

But, Ted says, the real problem is the missing `fsync()` calls. There are a number of reasons why they are not there, including developer laziness, the problem that `fsync()` on ext3 has become very expensive, the difficulty involved in preserving access control lists and other extended attributes when creating new files, and concerns about the battery-life costs of forcing the disk to spin up. Ted had more sympathy for some of these reasons than others, but, he says, "the application developers outnumber us," so something will have to be done to meet their concerns. 

Valerie Aurora broke in to point out that application developers have been put into a position where they cannot do the right thing. A call to `fsync()` can stall the system for quite a while on ext3. Users don't like that either; witness the fuss caused by excessive use of `fsync()` by the Firefox browser. So it's not just that application developers are lazy; there are real disincentives to the use of `fsync()`. Ted agreed, but he also claimed that a lot of application developers are refusing to help fix the problem. 

In the short term, the ext4 filesystem has gained a number of workarounds to help prevent the worst surprises. If a newly-written file is renamed on top of another, existing file, its data will be flushed to disk with the next commit. Similar things happen with files which have been truncated and rewritten. There is a performance cost to these changes, but they do make a significant part of the problem go away. 

For the longer term, Ted asked: should the above-described fixes become a part of the filesystem policy for Linux? In other words, should application developers be assured that they'll be able to write a file, rename it on top of another file, omit `fsync()`, and not encounter zero-length files after a crash? The answer turns out to be "yes," but first Ted presented his other long-term ideas. 

One of those is to improve the performance of the `fsync()` system call. The ext4 workarounds have also been added to ext3 when it runs in the `data=writeback` mode. Additionally, some block-layer fixes have been incorporated into 2.6.30. With those fixes in place, it is possible to run in `data=writeback` mode, avoid the zero-length file problem, and also avoid the `fsync()` performance problem. So, Ted asked, should `data=writeback` be made the default for ext3? 

This idea was received with a fair amount of discomfort. The `data=writeback` mode brings back problems that were fixed by `data=ordered`; after a crash, a file which was being written could turn up with completely unrelated data in it. It could be somebody else's sensitive data. Even if it's boring data, the problem looks an awful lot like file corruption to many users. It seems like a step backward and a change which is hard to justify for a filesystem which is headed toward maintenance mode. So it would be surprising to see this change made. 

[After writing the above, your editor noticed that Linus had just merged a change to make `data=writeback` the default for ext3 in 2.6.30. Your editor, it seems, is easily surprised.] 

Finally, the idea of the `fbarrier()` system call was raised. Essentially, `fbarrier()` would ensure that any data written to a file prior to the call would be flushed to disk before any metadata changes made after the call. It could be implemented with `fsync()`; for ext3 `data=ordered` mode, it would do nothing at all. Ted did not try hard to sell this system call, saying that it was mainly there to address the laptop power consumption concern. Ric Wheeler claimed that it would be a waste of time; by the time people are actually using it, we'll all have solid-state drives in our laptops and the power concern will be gone. In general, enthusiasm for `fbarrier()` was low. 

So the discussion turned back to the idea of generalizing and guaranteeing the ext4 workarounds. Chris Mason asked when there might be a time that somebody would _not_ want to rename files safely; he did not get an answer. There was concern that these workarounds could not be allowed to hurt the performance of well-written applications. But the general sentiment was that these workarounds should become policy that all filesystems should implement. 

### pNFS

There was a session on supporting [parallel NFS](<http://lwn.net/Articles/313437/>) (pNFS). It was mostly a detailed, technical discussion on what sort of API is needed to allow clustered filesystems to tell pNFS about how files are distributed across servers. Your editor will confess that his eyes glazed over after a while, and his notes are relatively incoherent. Suffice to say that, eventually, OCFS2 and GFS will be able to communicate with pNFS servers and that all the people who really care about how that works will understand it. 

### Miscellaneous topics

The final session of the day related to "miscellaneous VFS topics"; the first had to do with eCryptfs. This filesystem provides encryption for individual files; it is currently implemented as a stacking filesystem using an ordinary filesystem to provide the real storage. The stacking nature of eCryptfs has long been a problem; now some Ubuntu developers are working to change it. 

In particular, what they would like to do is to move the encryption handling directly into the VFS layer. Somehow users will supply a key to the kernel, which will then transparently handle the encryption and decryption of data. To that end, some sort of transformation layer will be provided to process the data between the page cache and the underlying block device. 

One question that came up was: what happens when the user does not have a valid key? Should the VFS just provide encrypted data in that case? Al Viro raised the question of what happens when one process opens the file with a key while another one opens it without a key. At that point there will be a mixture of encrypted and clear-text pages in the cache, a situation which seems sure to lead to confusion. So it seems that the VFS will simply refuse to provide access to files if the necessary key is not provided. 

There are various problems to be solved in the creation of the transformation layer - things like not letting processes modify a page while it is being encrypted or decrypted. Chris Mason noted that he faces a similar problem when generating checksums for pages in Btrfs. These are problems which can be addressed, though. But it was clear that this kind of transformation is likely to be built into the VFS in the future. Stacking filesystems just do not work well with the Linux VFS as it exists now. 

Next up was David Brown, who works in the scientific high-performance computing field. David has an interesting problem. He runs massive systems with large storage arrays spread out across many systems. Whenever some process calls `stat()` on a file stored in that array, the entire cluster essentially has to come to a stop. Locks have to be acquired, cached pages have to be flushed out, etc., just to ensure that specific metadata (the file size in particular) is available and correct. So, if a scientist logs in and types "ls" in a large directory, the result can be 30 minutes in coming and little work gets done in the mean time. Not ideal. 

What David would like is a "stat() light" call which wouldn't cause all of this trouble. It should return the metadata to the best of its knowledge, but it would not flush caches or take cluster-wide locks to obtain this information. If that means that the size is not entirely accurate, so be it. In the subsequent discussion, the idea was modified a little bit. "Slightly inaccurate" results would not be returned; instead, the size would simply be zeroed out. It was felt that returning no information at all was better than returning something which may have no real basis in reality. 

Beyond that, there would likely be a mask associated with the system call. Initially it was suggested that the mask would be returned; it would have bits set to indicate which fields in the return `stat` structure are valid. But it was also suggested that the mask should be an input parameter instead; the call would then do whatever was needed to provide the fields requested by the caller. Using the mask as an input parameter would avoid the need for duplicate calls in the case where the necessary information is not provided the first time around. 

The actual form of the system call is likely to be determined when somebody follows Christoph Hellwig's advice to "send a bloody patch." 

The final topic of the day was union mounts. Valerie Aurora, who led this session, recently wrote [an article about union filesystems and the associated problems](<http://lwn.net/Articles/324291/>) for LWN. The focus of this session was the `readdir()` system call in particular. POSIX requires that `readdir()` provide a position within a directory which can be used by the application at any future time to return to the same spot and resume reading directory entries. This requirement is hard for any contemporary filesystem to meet. It becomes almost impossible for union filesystems, which, by definition, are presenting a combination of at least two other filesystems. 

The solution that Valerie was proposing was to simply recreate directories in the top (writable) layer of the union. The new directories would point to files in the appropriate places within the union and would have whiteouts applied. That would eliminate the need to mix together directory entries from multiple layers later on, and the `readdir()` problem would collapse back to the single-filesystem implementation. At least, that holds true for as long as none of the lower-level filesystems in the union change. Valerie proposes that these filesystems be forced to be read-only, with an unmount required before they could be changed. 

The good news is that this is how BSD union mounts have worked for a long time. 

The bad news is that there's one associated problem: inode number stability. NFS servers are expected to provide stable inode numbers to clients even across reboots. But copying a file entry up to the top level of a union will change its inode number, confusing NFS clients. One possible solution to this problem is to simply decree that union mounts cannot be exported via NFS. It's not clear that there is a plausible use case for this kind of export in any case. The other solution is to just let the inode number change. That could lead to different NFS clients having open file descriptors to different versions of the file, but so be it. The consensus seemed to lean toward the latter solution. 

And that is where the workshop concluded. Your editor will be attending most of the second and final day (minus a brief absence for a cameo appearance at the Embedded Linux Conference); a report from that day will be posted shortly thereafter.
