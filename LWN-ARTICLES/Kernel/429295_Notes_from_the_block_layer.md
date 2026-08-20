---
title: Notes from the block layer
url: https://lwn.net/Articles/429295/
date: "February 22, 2011"
category: Block layer; Data integrity; Stable pages
author: "By Jonathan Corbet February 22, 2011"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
February 22, 2011

Over the last week or so a number of interesting topics have come up with regard to the low-level functioning of the block layer. This article will survey a few of these topics. 

**Enforcing read-only** : The block layer has a mechanism by which a driver can mark a specific device (or partition) as being read-only. This flag may be set if the physical device is write-locked; it can also be set by higher-level code (the DM or MD layers, for example) when the administrator creates a read-only device. Tejun Heo discovered an interesting thing, though: this flag is not enforced within the block layer. An attempt to open a write-protected device for write access will succeed, and the block layer will happily issue write operations to a read-only device. That struck Tejun as wrong, so he put a patch for 2.6.38 which addresses part of the problem: an attempt to open a read-only device for write access will be blocked. 

It turns out, though, that this check breaks things. Since enforcement of read-only status has never been done, developers have been careless about how they open block devices. So, with this patch in place, the loop device, device mapper, and MD all break when trying to open a read-only device, even if the ultimate goal is read-only access. Breaking things on this scale is not one of the stated goals of the 2.6.38 development cycle, so Chuck Ebbert has posted [a patch](<https://lwn.net/Articles/429299/>) reverting the change; some version of this patch is likely to be merged before the final 2.6.38 release. 

In-kernel code which is careless about open permissions can easily be fixed, but fixing the user-space utilities will take rather longer. So this check probably cannot be put into the `open()` path anytime soon. Beyond that, as [Linus pointed out](<https://lwn.net/Articles/429301/>), it may never really be the right thing to do; there are times when it may be necessary to open a read-only device for write access. Real enforcement of read-only status, if it is to be done in the block layer, probably needs to happen when operations are actually submitted to the device. How many things that would break remains to be seen. 

**Stable pages** : Linux has had support for [block data integrity checking](<https://lwn.net/Articles/290141/>) since 2008. In short, this feature takes advantage of suitably-equipped hardware to ensure that data is not corrupted between the host and its destination in persistent storage. Before writing a block to a device, the kernel will calculate a checksum and send it with the data; if the data, once written by the device, no longer matches the checksum, the device will signal an error. This mechanism can increase overall confidence that the system is storing data without corrupting it. 

There is one little problem, though. Imagine a sequence of events where the kernel calculates a checksum for a specific block, issues a write operation, then goes on to do more interesting things. Before the block controller gets around to acting on the request, some process comes along and changes the contents of the block. At this point, the checksum will no longer match, and the operation can fail. What is the best way to respond to (or, better, prevent) this outcome? 

Darrick Wong has addressed this problem with [a patch](<https://lwn.net/Articles/429305/>) which takes a possibly heavy-handed approach: when integrity checking is in use, blocks will be copied before the checksum is calculated and the I/O operation initiated. The rest of the system can then do anything it wants with the original data; the data as it existed when the write operation was queued will be written to the device. This approach will certainly work, but the cost is clear: an extra copy operation is added to the write path. That is not a cost that sits well with all developers. 

The proper way to solve this, for some value of "proper," is implementing "stable pages" within the filesystem code. In essence: a page which is under writeout becomes immutable; any process trying to change that page's contents will block until the write operation is complete. This solution is not universally popular either; it is said to have an adverse impact on at least one benchmark regardless of whether integrity checking is in use. As Jan Kara [noted](<https://lwn.net/Articles/429306/>), the best-performing approach will not be the same for everybody: 

In fact what is going to be faster depends pretty much on your system config. If you have enough CPU/RAM bandwidth compared to storage speed, you're better [off] doing copying. If you can barely saturate storage with your CPU/RAM, waiting is probably better for you. 

Some people also like the fact that the block-copying approach puts the pain on users of the integrity-checking features while not hurting other users - assuming that the cost of all those page allocations and copies doesn't affect anybody else. That said, stable pages look like they will be the approach taken in the future; as Martin Petersen [pointed out](<https://lwn.net/Articles/429307/>), there are a number of filesystem features - encryption, for example - which depend on it. Work is underway to add this capability to a number of filesystems; at the moment, only Btrfs has proper stable page support. 

**Comprehensive block I/O throttling coverage**. Last week's Kernel Page featured [hierarchical I/O scheduling](<https://lwn.net/Articles/427961/>); that work fills in an important feature, but the limitations of the (quite new) bandwidth controller don't stop there. One of its larger shortcomings is that it only really works with I/O submitted directly from process context. When I/O is initiated by the kernel (in particular, when the writeback code flushes dirty pages to disk), the controller is unable to associate the pages with the process that dirtied them. Since on many (or most) systems most block I/O writes are generated that way, it is easy to see that the block I/O controller's coverage is somewhat limited at the moment. 

Andrea Righi has posted [a patch set](<https://lwn.net/Articles/429292/>) which is meant to lift that limitation by tracking the ownership of all dirty pages in the system. There is code in the kernel now which can do that ownership tracking; the memory usage controller needs that information to do its job. So Andrea's patch generalizes the ownership tracking code and makes it serve the I/O controller's purposes as well. Half of the existing `flags` field in `struct page_cgroup` are taken to hold an index describing which control group the page belongs to. That makes the block controller different from how the memory controller uses this structure - the latter stores a direct pointer to its `mem_cgroup` structure - but it does have the advantage of not increasing the size of the `page_cgroup` structure. 

That advantage is not to be undervalued: `struct page_cgroup` shadows `struct page`, so one can exist for almost every page in the system. Even a little bit of overhead adds up quickly when such large quantities are involved. That overhead will be the biggest disadvantage of this new feature; anybody who wants to throttle block I/O bandwidth, and who is not also using the memory controller, will pay a significant cost in increased kernel memory use. The payback is that block I/O throttling actually works as intended; without page tracking, it can only give approximate results.
