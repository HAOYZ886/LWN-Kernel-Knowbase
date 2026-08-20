---
title: "Linux kernel design patterns - part 3"
url: https://lwn.net/Articles/336262/
date: "June 22, 2009"
category: "Development model-Patterns"
author: "June 22, 2009 This article was contributed by Neil Brown"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

June 22, 2009

This article was contributed by Neil Brown

In this final article we will be looking at just one design pattern. We [started](<http://lwn.net/Articles/336224/>) with the fine details of reference counting, [zoomed out](<http://lwn.net/Articles/336255/>) to look at whole data structures, and now move to the even larger perspective of designing subsystems. Like every pattern, this pattern needs a name, and our working title is "midlayer mistake". This makes it sounds more like an anti-pattern, as it appears to describe something that should be avoided. While that is valid, it is also very strongly a pattern with firm prescriptive guides. When you start seeing a "midlayer" you know you are in the target area for this pattern and it is time to see if this pattern applies and wants to guide you in a different direction. 

In the Linux world, the term "midlayer" seems (in your author's mind and also in Google's cache) most strongly related to SCSI. The "scsi midlayer" went through a bad patch quite some years ago, and there was plenty of debate on the relevant lists as to why it failed to do what was needed. It was watching those discussions that provided the germ from which this pattern slowly took form. 

The term "midlayer" clearly implies a "top layer" and a "bottom layer". In this context, the "top" layer is a suite of code that applies to lots of related subsystems. This might be the POSIX system call layer which supports all system calls, the block layer which supports all block devices, or the VFS which supports all filesystems. The block layer would be the top layer in the "scsi midlayer" example. The "bottom" layer then is a particular implementation of some service. It might be a specific system call, or the driver for a specific piece of hardware or a specific filesystem. Drivers for different SCSI controllers fill the bottom layer to the SCSI midlayer. Brief reflection on the list of examples shows that which position a piece of code takes is largely a matter of perspective. To the VFS, a given filesystem is part of the bottom layer. To a block device, the same filesystem is part of the top layer. 

A midlayer sits between the top and bottom layers. It receives requests from the top layer, performs some processing common to the implementations in the bottom layer, and then passes the preprocessed requests - presumably now much simpler and domain-specific - down to the relevant driver. This provides uniformity of implementation, code sharing, and greatly simplifies that task of implementing a bottom-layer driver. 

The core thesis of the "midlayer mistake" is that midlayers are bad and should not exist. That common functionality which it is so tempting to put in a midlayer should instead be provided as library routines which can used, augmented, or ignored by each bottom level driver independently. Thus every subsystem that supports multiple implementations (or drivers) should provide a very thin top layer which calls directly into the bottom layer drivers, and a rich library of support code that eases the implementation of those drivers. This library is available to, but not forced upon, those drivers. 

To try to illuminate this pattern, we will explore three different subsystems and see how the pattern specifically applies to them - the block layer, the VFS, and the 'md' raid layer (i.e. the areas your author is most familiar with). 

#### Block Layer

The bulk of the work done by the block layer is to take 'read' and 'write' requests for block devices and send them off to the appropriate bottom level device driver. Sounds simple enough. The interesting point is that block devices tend to involve rotating media, and rotating media benefits from having consecutive requests being close together in address space. This helps reduce seek time. Even non-rotating media can benefit from having requests to adjacent addresses be adjacent in time so they can be combined into a smaller number of large requests. So, many block devices can benefit from having all requests pass through an elevator algorithm to sort them by address and so make better use of the device. 

It is very tempting to implement this elevator algorithm in a 'midlayer'. i.e. a layer just under the top layer. This is exactly what Linux did back in the days of 2.2 kernels and earlier. Requests came in to `ll_rw_block()` (the top layer) which performed basic sanity checks and initialized some internal-use fields of the structure, and then passed the request to `make_request()` \- the heart of the elevator. Not quite every request went to `make_request()` though. A special exception was made for "md" devices. Those requests were passed to `md_make_request()` which did something completely different as is appropriate for a RAID device. 

Here we see the first reason to dislike midlayers - they encourage special cases. When writing a midlayer it is impossible to foresee every possible need that a bottom level driver might have, so it is impossible to allow for them all in the midlayer. The midlayer could conceivably be redesigned every time a new requirement came along, but that is unlikely to be an effective use of time. Instead, special cases tend to grow. 

Today's block layer is, in many ways, similar to the way it was back then with an elevator being very central. Of course lots of detail has changed and there is a lot more sophistication in the scheduling of IO requests. But there is still a strong family resemblance. One important difference (for our purposes) is the existence of the function `blk_queue_make_request()` which every block device driver must call, either directly or indirectly via a call to `blk_init_queue()`. This registers a function, similar to `make_request()` or `md_make_request()` from 2.2, which should be called to handle each IO request. 

This one little addition effectively turns the elevator from a midlayer which is imposed on every device into a library function which is available for devices to call upon. This was a significant step in the right direction. It is now easy for drivers to choose not to use the elevator. All virtual drivers (md, dm, loop, drbd, etc.) do this, and even some drivers for physical hardware (e.g. umem) provide their own `make_request_fn()`. 

While the elevator has made a firm break from being a mid-layer, it still retains the appearance of a midlayer in a number of ways. One example is the `struct request_queue` structure (defined in `<linux/blkdev.h>`). This structure is really part of the block layer. It contains fields that are fundamental parts of the block interface, such as the `make_request_fn()` function pointer that we have already mentioned. However many other fields are specific to the elevator code, such as `elevator` (which chooses among several IO schedulers) and `last_merge` (which is used to speed lookups in the current queue). While the elevator can place fields in `struct request_queue`, all other code must make use of the `queuedata` pointer to store a secondary data structure. 

This arrangement is another tell-tale for a midlayer. When a primary data structure contains a pointer to a subordinate data structure, we probably have a midlayer managing that primary data structure. A better arrangement is to use the "embedded anchor" pattern from the previous article in this series. The bottom level driver should allocate its own data structure which contains the data structure (or data structures) used by the libraries embedded within it. `struct inode` is a good example of this approach, though with slightly different detail. In 2.2, `struct inode` contained a union of the filesystem-specific data structure for each filesystem, plus a pointer (`generic_ip`) for another filesystem to use. In the 2.6 kernel, `struct inode` is normally embedded inside a filesystem-specific inode structure (though there is still an `i_private` pointer which seems unnecessary). 

One last tell-tale sign of a midlayer, which we can still see hints of in the elevator, is the tendency to group unrelated code together. The library design will naturally provide separate functionality as separate functions and leave it to the bottom level driver to call whatever it needs. The midlayer will simply call everything that might be needed. 

If we look at `__make_request()` (the 2.6 entry point for the elevator), we see an early call to `blk_queue_bounce()`. This provides support for hardware that cannot access the entire address space when using DMA to move data between system memory and the device. To support such cases, data sometimes needs to be copied into more accessible memory before being transferred to the device, or to be copied from that memory after being transferred from the device. This functionality is quite independent of the elevator, yet it is being imposed on all users of the elevator. 

So we see in the block layer, and its relationship with the elevator a subsystem which was once implemented as a midlayer, but has taken a positive step away from being a midlayer by making the elevator clearly optional. It still contains traces of its heritage which have served as a useful introduction to the key identifiers of a midlayer: code being imposed on lower layer, special cases in that code, data structures storing pointers to subordinate data structures, and unrelated code being called by the one support function. 

With this picture in mind, let us move on. 

#### The VFS

The VFS (or Virtual File System) is a rich area to explore to learn about midlayers and their alternatives. This is because there is a lot of variety in filesystems, a lot of useful services that they can make use of, and a lot of work has been done to make it all work together effectively and efficiently. The top layer of the VFS is largely contained in the `vfs_` function calls which provide the entry points to the VFS. These are called by the various `sys_` functions that implement system calls, by nfsd which does a lot of file system access without using system calls, and from a few other parts of the kernel that need to deal with files. 

The `vfs_` functions fairly quickly call directly in to the filesystem in question through one of a number of `_operations` structures which contain a list of function pointers. There are `inode_operations`, `file_operations`, `super_operations` etc, depending on what sort of object is being manipulated. This is exactly the model that the "midlayer mistake" pattern advocates. A thin top layer calls directly into the bottom layer which will, as we shall see, make heavy use of library functions to perform its task. 

We will explore and contrast two different sets of services provided to filesystems, the page cache and the directory entry cache. 

#### The page cache

Filesystems generally want to make use of read-ahead and write-behind. When possible, data should be read from storage before it is needed so that, when it is needed, it is already available, and once it has been read, it is good to keep it around in case, as is fairly common, it is needed again. Similarly, there are benefits from delaying writes a little, so that throughput to the device can be evened out and applications don't need to wait for writeout to complete. Both of these features are provided by the page cache, which is largely implemented by `mm/filemap.c` and `mm/page-writeback.c`. 

In its simplest form a filesystem provides the page cache with an object called an `address_space` which has, in its `address_space_operations`, routines to read and write a single page. The page cache then provides operations that can be used as `file_operations` to provide the abstraction of a file that must be provided to the VFS top layer. If you look at the `file_operations` for a regular file in ext3, we see: 

```
const struct file_operations ext3_file_operations = {
    	.llseek		= generic_file_llseek,
    	.read		= do_sync_read,
    	.write		= do_sync_write,
    	.aio_read	= generic_file_aio_read,
    	.aio_write	= ext3_file_write,
    	.unlocked_ioctl	= ext3_ioctl,
        #ifdef CONFIG_COMPAT
    	.compat_ioctl	= ext3_compat_ioctl,
        #endif
    	.mmap		= generic_file_mmap,
    	.open		= generic_file_open,
    	.release	= ext3_release_file,
    	.fsync		= ext3_sync_file,
    	.splice_read	= generic_file_splice_read,
    	.splice_write	= generic_file_splice_write,
        };
```

Eight of the thirteen operations are generic functions provided by the page cache. Of the remaining five, the two `ioctl()` operations and the `release()` operation require implementations specific to the filesystem; `ext3_file_write()` and `ext3_sync_file` are moderately sized wrappers around generic functions provided by the page cache. This is the epitome of good subsystem design according to our pattern. The page cache is a well defined library which can be used largely as it stands (as when reading from an ext3 file), allows the filesystem to add functionality around various entry points (like `ext3_file_write()`) and can be simply ignored altogether when not relevant (as with sysfs or procfs). 

Even here there is a small element of a midlayer imposing on the bottom layer as the generic `struct inode` contains a `struct address_space` which is only used by the page cache and is irrelevant to non-page-cache filesystems. This small deviation from the pattern could be justified by the simplicity it provides, as the vast majority of filesystems do use the page cache. 

#### The directory entry cache (dcache)

Like the page cache, the dcache provides an important service for a filesystem. File names are often accessed multiple times, much more so than the contents of file. So caching them is vital, and having a well designed and efficient directory entry cache is a big part of having efficient access to all filesystem objects. The dcache has one very important difference from the page cache though: it is not optional. It is imposed upon every filesystem and is effectively a "midlayer." Understanding why this is, and whether it is a good thing, is an important part of understanding the value and applicability of this design pattern. 

One of the arguments in favor of an imposed dcache is that there are some interesting races related to directory renames; these races are easy to fail to handle properly. Rather than have every filesystem potentially getting these wrong, they can be solved once and for all in the dcache. The classic example is if `/a/x` is renamed to `/b/c/x` at the same time that `/b/c` is renamed to `/a/x/c`. If these both succeed, then 'c' and 'x' will contain each other and be disconnected from the rest of the directory tree, which is a situation we would not want. 

Protecting against this sort of race is not possible if we only cache directory entries at a per-directory level. The common caching code needs to at least be able to see a whole filesystem to be able to detect such a possible loop-causing race. So maintaining a directory cache on a per-filesystem basis is clearly a good idea, and strongly encouraging local filesystems to use it is very sensible, but whether forcing it on all filesystems is a good choice is less clear. 

Network filesystems do not benefit from the loop detection that the dcache can provide as all of that must be done on the server anyway. "Virtual" filesystems such as sysfs, procfs, ptyfs don't particularly need a cache at all as all the file names are in memory permanently. Whether a dcache hurts these filesystems is not easy to tell as we don't have a complete and optimized implementation that does not depend on the dcache to compare with. 

Of the key identifiers for a midlayer that were discussed above, the one that most clearly points to a cost is the fact that midlayers tend to grow special case code. So it should be useful to examine the dcache to see if it has suffered from this. 

The first special cases that we find in the dcache are among the flags stored in `d_flags`. Two of these flags are `DCACHE_AUTOFS_PENDING` and `DCACHE_NFSFS_RENAMED`. Each is specific to just one filesystem. The AUTOFS flag appears to only be used internally to autofs, so this isn't really a special case in the dcache. However the NFS flag is used to guide decisions made in common dcache code in a couple of places, so it clearly is a special case, though not necessarily a very costly one. 

Another place to look for special case code is when a function pointer in an `_operations` structure is allowed to be NULL, and the NULL is interpreted as implying some specific action (rather than no action at all). This happens when a new operation is added to support some special-case, and NULL is left to mean the 'default' case. This is not always a bad thing, but it can be a warning signal. 

In the `dentry_operations` structure there are several functions that can be NULL. `d_revalidate()` is an example which is quite harmless. It simply allows a filesystem to check if the entry is still valid and either update it or invalidate it. Filesystems that don't need this simply do nothing as having a function call to do nothing is pointless. 

However, we also find `d_hash()` and `d_compare()`, which allow the filesystem to provide non-standard hash and compare functions to support, for example, case-insensitive file names. This does look a lot like a special case because the common code uses an explicit default if the pointer is NULL. A more uniform implementation would have every filesystem providing a non-NULL `d_hash()` and `d_compare()`, where many filesystems would choose the case-sensitive ones from a library. 

It could easily be argued that doing this - forcing an extra function call for hash and compare on common filesystems - would be an undue performance cost, and this is true. But given that, why is it appropriate to impose such a performance cost on filesystems which follow a different standard? 

A more library-like approach would have the VFS pass a path to the filesystem and allow it to do the lookup, either by calling in to a cache handler in a library, or by using library routines to pick out the name components and doing the lookups directly against its own stored file tree. 

So the dcache is clearly a midlayer, and does have some warts as a result. Of all the midlayers in the kernel it probably best fits the observation above that they could "be redesigned every time a new requirement came along". The dcache does see constant improvement to meet the needs of new filesystems. Whether that is "an effective use of time" must be a debate for a different forum. 

#### The MD/RAID layer

Our final example as we consider midlayers and libraries, is the md driver which supports various software-RAID implementations and related code. md is interesting because it has a mixture of midlayer-like features and library-like features and as such is a bit of a mess. 

The "ideal" design for the md driver is (according to the "midlayer mistake" pattern) to provide a bunch of useful library routines which independent RAID-level modules would use. So, for example, RAID1 would be a standalone driver which might use some library support for maintaining spares, performing resync, and reading metadata. RAID0 would be a separate driver which use the same code to read metadata, but which has no use for the spares management or resync code. 

Unfortunately that is not how it works. One of the reasons for this relates to the way the block layer formerly managed major and minor device numbers. It is all much more flexible today, but in the past a different major number implied a unique device driver and a unique partitioning scheme for minor numbers. Major numbers were a limited resource, and having a separate major for RAID0, RAID1, and RAID5 etc would have been wasteful. So just one number was allocated (9) and one driver had to be responsible for all RAID levels. This necessity undoubtedly created the mindset that a midlayer to handle all RAID levels was the right thing to do, and it persisted. 

Some small steps have been made towards more of a library focus, but they are small and inconclusive. One simple example is the `md_check_recovery()` function. This is a library function in the sense that a particular RAID level implementation needs to explicitly call it or it doesn't get used. However, it performs several unrelated tasks such as updating the metadata, flushing the write-intent-bitmap, removing devices which have failed, and (surprisingly) checking if recovery is needed. As such it is a little like part of a mid-layer in that it imposes that a number of unrelated tasks are combined together. 

Perhaps a better example is `md_register_thread()` and friends. Some md arrays need to have a kernel thread running to provide some support (such as scheduling read requests to different drives after a failure). `md.c` provides library routines `md_register_thread()` and `md_unregister_thread()`, which can be called by the personality as required. This is all good. However md takes it upon itself to choose to call `md_unregister_thread()` at times rather than leaving that up to the particular RAID level driver. This is a clear violation of the library approach. While this is not causing any actual problems at the moment, it is exactly the sort of thing that could require the addition of special cases later. 

It has often been said that md and dm should be unified in some way (though it is less often that the practical issues of what this actually means are considered). Both md and dm suffer from having a distinct midlayer that effectively keeps them separate. A full understanding of the fact that this midlayer is a mistake, and moving to replace it with an effective library structure is likely to be an important first step towards any sort of unification. 

#### Wrap up

This ends our exploration of midlayers and libraries in the kernel -- except maybe to note that more recent additions include such things as libfs, which provides support for virtual filesystems, and libata, which provides support for SATA drives. These show that the tendency away from midlayers is not only on the wishlist of your author but is present in existing code. 

Hopefully it has resulted in an understanding of the issues behind the "midlayer mistake" pattern and the benefits of following the library approach. 

Here too ends our little series on design patterns in the Linux kernel. There are doubtlessly many more that could be usefully extracted, named, and illuminated with examples. But they will have to await another day. 

Once compiled, such a collection would provide invaluable insight on how to build kernel code both effectively and uniformly. This would be useful in understanding how current code works (or why it doesn't), in making choices when pursuing new development, or when commenting on design during the review process, and would generally improve visibility at this design level of kernel construction. Hopefully this could lead, in the long term, to an increase in general quality. 

For now, as a contribution to that process, here is a quick summary of the Patterns we have found. 

  * **kref** : Reference counting when the object is destroyed with the last external reference 

  * **kcref** : Reference counting when the object can persist after the last external reference is dropped 

  * **plain ref** : Reference counting when object lifetime is subordinate to another object. 

  * **biased-reference** : An anti-pattern involving adding a bias to a reference counter to store one bit of information. 

  * **Embedded Anchor** : This is very useful for lists, and can be generalized as can be seen if you explore kobjects. 

  * **Broad Interfaces** : This reminds us that trying to squeeze lots of use-cases in to one function call is not necessary - just provide lots of function calls (with helpful and (hopefully) consistent names). 

  * **Tool Box** : Sometimes it is best not to provide a complete solution for a generic service, but rather to provide a suite of tools that can be used to build custom solutions. 

  * **Caller Locks** : When there is any doubt, choose to have the caller take locks rather than the callee. This puts more control in that hands of the client of a function. 

  * **Preallocate Outside Locks** : This is in some ways fairly obvious. But it is very widely used within the kernel, so stating it explicitly is a good idea. 

  * **Midlayer Mistake** : When services need to be provided to a number of low-level drivers, provide them with a library rather than imposing them with a midlayer. 

#### Exercises

  1. Examine the "`blkdev_ioctl()`" interface to the block layer from the perspective of whether it is more like a midlayer or a library. Compare the versions in 2.6.27 with 2.6.28. Discuss. 

  2. Choose one other subsystem such as networking, input, or sound, and examine it in the light of this pattern. Look for special cases, and imposed functionality. Examine the history of the subsystem to see if there are signs of it moving away from, or towards, a "midlayer" approach. 

  3. Identify a design pattern which is specific to the Linux kernel but has not been covered in this series. Give it a name, and document it together with some examples and counter examples.
