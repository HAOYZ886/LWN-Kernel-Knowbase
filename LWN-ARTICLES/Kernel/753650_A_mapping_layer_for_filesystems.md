---
title: A mapping layer for filesystems
url: https://lwn.net/Articles/753650/
date: "May 9, 2018"
category: Block layer
author: "By Jake Edge May 9, 2018 LSFMM"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jake Edge**  
May 9, 2018

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2018/>)

In a plenary session on the second day of the Linux Storage, Filesystem, and Memory-Management Summit (LSFMM), Dave Chinner described his ideas for a virtual block address-space layer. It would allow "space accounting to be shared and managed at various layers in the storage stack". One of the targets for this work is for filesystems on thin-provisioned devices, where the filesystem is larger than the storage devices holding it (and administrators are expected to add storage as needed); in current systems, running out of space causes huge problems for filesystems and users because the filesystem cannot communicate that error in a usable fashion. 

His talk is not about block devices, he said; it is about a layer that provides a managed logical-block address (LBA) space. It will allow user space to make `fallocate()` calls that truly reserve the space requested. Currently, a filesystem will tell a caller that the space was reserved even though the underlying block device may not actually have that space (or won't when user space goes to use it), as in a thin-provisioned scenario. He also said that he would not be talking about his ideas for a snapshottable subvolume for XFS that was the subject of his [talk at linux.conf.au 2018](<https://lwn.net/Articles/747633/>). 

The new layer will provide the address space, which is a representation of an LBA range. There will be a set of interfaces to manage the backend storage for that range. A filesystem will usually be the client of the interface, while a block device or a separate filesystem can be the supplier of the storage for the layer. 

[ ![\[Dave Chinner\]](https://static.lwn.net/images/2018/lsf-chinner-sm.jpg) ](<https://lwn.net/Articles/753664/>)

The filesystem does not treat the virtual block address layer any differently than it does a block device from a space-management perspective. The supplier provides allocation and space reservation; it could also provide copy-on-write (CoW) to the upper layer, which would allow for snapshots at that level. In order to read and write data, however, a mapping must be done to turn the virtual LBA into a real LBA and block device for the I/O. It is similar to the export blocks feature of Parallel NFS (pNFS). 

When the client wants to do I/O, it first maps the virtual LBA, then does the operation directly to the block device where the data is stored. Jan Kara asked if it is simply a remapping layer for filesystems; Chinner agreed that it was. He was looking at adding this ability to XFS but realized it was more widely applicable. It is similar to what is done for loopback devices, but he has chopped some layers out of that; instead of going through the block device interface, it is going through the remapping layer. 

One of the problems with space reservation is that there may be a delay between the write of data and its associated metadata. But it is important that space reserved for that metadata does not disappear when it comes time to write the metadata. The upper layer filesystem needs to be able to ensure that a later writeback does not get an `ENOSPC` error for something that it believes it can write. 

Under this new scheme, the filesystem can ask the supplier for a reservation, which will result in an opaque cookie that the filesystem can use to indicate portions of the reservation. Every object modification has the cookie associated with it; when all of those modifications are done, the reference count on the cookie drops to zero and any extra reservation goes back to the backend. 

This allows allocation based on the I/O that the filesystem is building. It also can allow for write combining that is optimal for the thin-provisioned devices. Overall, it allows for optimal I/O for the underlying structures, he said. 

The client does not know anything about what the underlying backing store actually does. Similarly, the supplier does not know what the client is doing; it is just allocating and mapping. The idea is just to create an abstraction that allows two different layers in the stack to manage blocks in a way that can report errors properly. 

When the BIO is formed for a read operation, the filesystem does everything it does now, but it also calls out to the mapping layer to find out which block device to do the read on. It will issue I/O directly to the underlying device, taking a shortcut around all of the layers that a loopback device would use, he said. 

A write operation would use a two-phase write that is similar to what XFS uses for direct I/O. It would get the block device and LBA from the mapping layer and it would also attach any needed reservation cookies to the BIO. If the target area is a hole, the system first allocates for those blocks; if it is a CoW supplier, it allocates new blocks and returns the mapping and reservation for those. All of that behavior would be hidden in the lower layers. The BIOs are built and sent down to the block device; when the write completes, the supplier must run its completion routines first, then the client runs its completions to finish its two-phase write. 

At no time does the client know anything about what the underlying backing store actually does, Chinner reiterated. Similarly, the supplier does not know what the client is actually doing; it simply handles allocation and mapping. Anything that can provide a 64-bit address space can be used as a supplier, a file could be used, for example. 

It is an abstract interface, he said, that is not specific to any filesystem or block device. It could be ext4 as a client with XFS as a supplier, or vice versa if ext4 implements the supplier interface. Ted Ts'o said that he originally thought this was all simply targeting thin provisioning, but having filesystems as the supplier "becomes interesting"; "that's neat". Chinner said his actual motivation was for XFS subvolumes, not thin provisioning. 

The problem has turned out to be fairly simple to solve. It is about 1700 lines of code right now and he thinks it will grow to 3000 or so once he gets it cleaned up and ready for posting. He does think it will be interesting for other filesystems. Kara said that it resembled some things that Btrfs does; Chinner agreed, he is not really doing anything new, but is simply "repackaging and reimagining" ideas that are already out there. 

One of the reasons he likes this approach is that it reuses the infrastructure already available in the filesystem layer. It can turn snapshots into regular files, for example. Chris Mason said that he uses loopback devices for some containers, but that this mechanism would be better. Chinner acknowledged that and noted that he has some "wild plans" for page-cache sharing that will make it even better. There are lots of use cases, he said, so will get his act together and post patches soon.
