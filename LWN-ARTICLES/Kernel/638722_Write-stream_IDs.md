---
title: "Write-stream IDs"
url: https://lwn.net/Articles/638722/
date: "April 7, 2015"
category: Block layer
author: "By Jonathan Corbet April 7, 2015"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
April 7, 2015

Storage devices with large physical block sizes — solid-state storage devices (SSDs), for example — are subject to a problem known as "write amplification" that can affect both the performance and lifetime of the device. Applications often have information about the data they write that can be helpful in reducing write amplification problems, but there is currently no way to communicate that information to the relevant parts of the kernel. A new proposed addition to the block-layer API may help to solve that problem in the near future, though. 

The kernel typically performs block I/O in units of 4KB, but a typical SSD has an erase-block size of many times that. The firmware in the drive itself performs the impedance matching between the small sector size exposed to the host computer and the real requirements imposed by the hardware. Whenever a sector is written, the firmware must find a home for it in a new erase block, leaving an empty space where the sector used to be. Occasionally, sectors must be shifted and coalesced during a garbage-collection pass to free up the empty spaces for new writes. 

"Write amplification" refers to this extra work that must be performed when data is overwritten. It gets worse if short-lived data is mixed with long-lived data in the same erase blocks; garbage collection must happen more often and more data must be moved around. On the other hand, if short-lived data can be kept together, the rewriting of erase blocks and garbage-collection work can be minimized. The kernel could perhaps do this kind of separation for some types of filesystem metadata, but it has no knowledge of how user space plans to use the data that it writes to the filesystem. So, if long-lived user-space data is to be separated from the short-lived variety, user space is going to have to help with the job. That is where Jens Axboe's [write-stream IDs patch set](<https://lwn.net/Articles/637687/>) comes in. 

A write-stream ID is simply an eight-bit integer value assigned to block data as it is written. The kernel does not interpret that value in any way other than as a hint that data with the same ID is likely to have approximately the same lifetime. Low-level storage drivers can use this ID to place data with the same life expectancy together on the media, hopefully reducing write-amplification problems in the process. 

At the lowest level in the block layer, the stream ID is stored in eight bits of the `bi_flags` field in the `bio` structure. It can be set with `bio_set_streamid()` and queried with `bio_get_streamid()`. A call to `bio_streamid_valid()` can be used to determine whether a given `bio` structure has had its stream ID set; low-level block drivers can use a valid stream ID to instruct the hardware to group similar data on the physical media. 

At the user-space level, the stream ID for an open file can be set with the new `fadvise(POSIX_FADV_STREAMID)` operation. Interestingly, this value is stored in two places: the `file` structure associated with the given file descriptor, and the `inode` structure representing the file itself. That might seem like an interesting choice, given that both structures are heavily used and bloating both of them with a new field is not something to be done lightly, but there is a reason for it. 

When an application performs direct I/O, the data being written will be placed in a `bio` structure immediately and passed to the block layer. The `file` structure corresponding to the file descriptor passed by user space is available then, so the stream ID stored in that `file` structure can be copied directly into the `bio` structure. 

That option is not available for buffered I/O, though. A buffered `write()` will simply copy the data into the page cache and mark the relevant page(s) dirty; the actual I/O on those pages will happen at some future time. By the time that the writeback code gets around to those pages, the `file` structure used to initiate the write may no longer exist. Even if it is still around, though, it is not readily accessible at that level of the kernel. But the `inode` structure is accessible. So, in this case, the stream ID must be taken from the `inode` structure. 

One might ask why the `inode`-stored stream ID is not used all of the time. The patches are silent on this point, but the probable answer is that the more direct control afforded by storing the ID in the `file` structure is worth having when it is possible. It allows the stream ID to be changed from one I/O operation to the next; different file descriptors referring to the same on-disk file can also have different stream IDs. This flexibility is not available when doing buffered I/O and using the stream ID stored in the `inode` structure; since it's not possible to know when the actual writeback will happen, the stream ID cannot be changed between writes without the likelihood of affecting writes intended to go under a different ID. 

There would be clear value in a closer association between stream IDs and specific buffered-write operations. Getting there would require storing the stream ID with each dirtied page, though; that, in turn, almost certainly implies shoehorning the stream ID into the associated `page` structure. That would not be an easy task; it is not surprising that it is not a part of this patch set. Should the lack of per-buffered-write stream IDs prove to be a serious constraint in the future, somebody will certainly be motivated to try to find a place to store another eight bits in `struct page`. 

Meanwhile, there does not appear to be any real opposition to the patch set in its current form. Unless that situation changes, write-stream IDs would appear to be a feature headed for the mainline in a near-future development cycle.
