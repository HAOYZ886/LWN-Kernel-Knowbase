---
title: The future of DAX
url: https://lwn.net/Articles/717953/
date: "March 27, 2017"
category: "DAX; Memory management-Nonvolatile memory"
author: "By Jonathan Corbet March 27, 2017 LSFMM 2017"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
March 27, 2017

* * *

[LSFMM 2017](<https://lwn.net/Articles/lsfmm2017/>)

DAX is the mechanism that enables direct access to files stored in persistent memory arrays without the need to copy the data through the page cache. At the 2017 Linux Storage, Filesystem, and Memory-Management Summit, Ross Zwisler led a plenary session on the future of DAX. Development in this area offers a number of interesting trade-offs between data safety and enabling the highest performance. 

The biggest issue for next year, Zwisler said, is finding the best way to handle flushing of data from user space. Data written to persistent memory by the CPU may look like it is permanently stored but, most likely, it has only made it as far as the cache; that data can still be lost in the event of a crash, power failure, or asteroid strike. For pages in the page cache, user space can use `msync()` to flush the data to persistent storage, but DAX pages explicitly avoid the page cache. So flushing data to permanent storage requires going through the radix tree, finding the dirty pages, and flushing the associated cache lines. Intel provides some instructions for performing this flushing quickly; the kernel will use those instructions to ensure that data is durable after an `msync()` call. So far, so good. 

The problem is that there are use cases where `msync()` is too slow, so users want to avoid it. Instead, they would like to write and flush [![\[Ross Zwisler\]](https://static.lwn.net/images/conf/2017/lsfmm/RossZwisler-sm.jpg)](<https://lwn.net/Articles/717958/>) individual chunks of data themselves without calling into the kernel. This method can be quite a bit faster, since the application knows which data it has written, while the kernel lacks the information to flush data at the individual cache-line level. 

This technique works as long as no file-data allocations have been done in the write path. Otherwise, there will be changed filesystem metadata that also needs to be flushed, and that will not happen in this scenario. As a result, data can be lost in a crash. A number of solutions to this problem have been proposed, but, according to Zwisler, Dave Chinner has called them all "crazy". A safer approach, [Chinner said last September](<https://lkml.org/lkml/2016/9/11/159>), is to simply require that files be completely preallocated before writing begins; at that point, there should be no metadata changes and the problem goes away. 

Rik van Riel suggested that applications should be required to open files with the `O_SYNC` option if they intend to access them in this mode, but Zwisler responded that the situation is not that simple. Jan Kara said that the problem could come from other applications performing operations in the filesystem that create metadata changes; those applications may be completely unaware of the other users and will not be concerned with flushing their changes out. Getting around that problem would require some sort of state stored at the inode level and not, like `O_SYNC`, at the file-descriptor level. 

But even then, the filesystem itself can destabilize the metadata by, for example, performing deduplication. In the end, Kara said, the only way for an application to know that a filesystem is in a consistent state on-disk is to call `fsync()`. Moving control of flushing to user space breaks a lot of assumptions; there will need to be a way to prevent filesystems from messing with things. 

Zwisler said that Chinner's proposal had anticipated this problem and, as a result, came with a lot of caveats. It would be necessary to turn off [reflink](<https://lwn.net/Articles/684826/>) functionality and other filesystem features, for example. Zwisler also said that device DAX, which presents persistent memory as a character device without a filesystem, exists for this kind of thing; device DAX gives the user total control. For the filesystem implementation, it might be best to just go with the preallocation idea, he said, while making it painful enough that there will be an incentive not to use it. But the incentives _to_ use it will also be there: by avoiding system calls, the user-controlled method is always going to be faster. 

Kara said that history shows that, if somebody is interested in a feature, businesses will work to provide it. With enough motivation, these problems can be solved. Zwisler said that there is a strong desire to have a filesystem in place on persistent memory; filesystems provide or enable nice features like naming, backups, and more. What is really needed is a new filesystem that was designed for persistent memory from the beginning, but that is not a short-term solution. Even if such a filesystem were to appear tomorrow, it's a rare user who is willing to trust production data to a brand-new filesystem. So we are going to have to get by with what we have now for some time yet. 

The group concluded that, for now, users will have to get by with limiting metadata updates or using device DAX. With luck, adventurous users will experiment with other ideas out of tree and better solutions will eventually emerge. 

The next question had to do with platforms that support "flush on fail" functionality — the ability to automatically flush data to persistent memory after a crash. On such hardware, there is no need to worry about doing explicit cache flushes; indeed, doing so will just slow things down. The big problem here is that there is no discovery method for this feature, so the user must ask for flushes to be turned off if they know that their hardware will do flush on fail. A feature to allow that will be provided; it is seen as being similar to the ability to turn off writeback caching on hard drives. 

Currently DAX is still marked as an experimental feature in the kernel, and mounting a filesystem with DAX enabled results in a warning in the log. When, Zwisler asked, can this be turned off? Support for the [reflink](<https://lwn.net/Articles/684826/>) feature, or at least the ability to "not collide with it" seems to be one remaining requirement; it is evidently being worked on. Dan Williams noted that DAX is currently turned off if `page` structures are not available for the persistent-memory array. It is possible to operate without those structures, but there is no support for huge pages, `fork()` will fail if persistent memory is mapped, and it's not possible to use a debugger on programs that have that memory mapped. He asked whether this was worth fixing, noting that it would not be a small job. Interest in addressing the issue seemed relatively low in the room. 

Zwisler said that the filesystem mount options for DAX are currently inconsistent. With ext4, DAX either works for all files or it doesn't work at all; XFS, instead, can enable or disable DAX on a per-inode basis. It would be better, he said, to have consistent behavior across filesystems before proclaiming the feature to be stable. 

Another wishlist feature is support for 1GB extra-huge pages. Device DAX can use such pages now, but they are not available when there is a filesystem involved. Fixing that problem would be relatively complex; among other things, it would require filesystems to lay out files in 1GB-aligned extents, which none do now. It is not clear that there is a use case for this feature, so nobody seems motivated to make it work now. 

The session concluded with a review of the changes needed to remove the "experimental" tag from DAX. More testing was added to the list; it's not clear if the test coverage is as good as it need to be yet or not. The concerns about interaction with reflink need to be addressed, and making the mount options consistent is also on the list (though some developers would like to just see the mount options go away entirely). That list is long enough that the future of DAX seems to include "experimental" status for a little while yet.
