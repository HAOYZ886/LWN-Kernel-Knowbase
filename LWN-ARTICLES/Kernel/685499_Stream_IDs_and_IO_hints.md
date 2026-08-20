---
title: Stream IDs and I/O hints
url: https://lwn.net/Articles/685499/
date: "May 4, 2016"
category: "Device drivers-Block drivers"
author: "By Jake Edge May 4, 2016 LSFMM 2016"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jake Edge**  
May 4, 2016

* * *

[LSFMM 2016](<https://lwn.net/Articles/lsfmm2016/>)

I/O hints are a way to try to give storage devices information that will allow them to make better decisions about how to store the data. One of the more recent hints is to have multiple "streams" of data that is associated in some way, which was mentioned in a [storage standards update](<https://lwn.net/Articles/684264/>) session the previous day. Changho Choi and Martin Petersen led a session at the 2016 Linux Storage, Filesystem, and Memory-Management Summit to flesh out more about streams, stream IDs, and I/O hints in general. 

[ ![\[Changho Choi\]](https://static.lwn.net/images/2016/lsf-choi-sm.jpg) ](<https://lwn.net/Articles/685986/>)

Choi said that he is leading the multi-stream specification and software-development work at Samsung. There is no mechanism for storage devices to expose their internal organization to the host, which can lead to inefficient placement of data and inefficient background operations (e.g. garbage collection). Streams are an attempt to provide better collaboration between the host and the device. The host gives hints to the device, which will then place the data in the most efficient way. That leads to better endurance as well as improved and consistent performance and latency, he said. 

A stream ID would be associated with operations for data that is expected to have the same lifetime. For example, temporary data, metadata, and user data could be separated into their own streams. The ID would be passed down to the device using the multi-stream interface and the device could put the data in the same erase blocks to avoid copying during garbage collection. 

For efficiency, proper mapping of data to streams is essential, Choi said. Keith Packard noted that filesystems try to put writes in logical block address (LBA) order for rotating media and wondered if that was enough of a hint. Choi said that more information was needed. James Bottomley suggested that knowing the size and organization of erase blocks on the device could allow the kernel to lay out the data properly. 

But there are already devices shipping with the multi-stream feature, from Samsung and others, Choi said. It is also part of the T10 (SCSI) standard and will be going into T13 (ATA) and NVM Express (NVMe) specifications. 

[ ![\[Martin Petersen\]](https://static.lwn.net/images/2016/lsf-petersen-sm.jpg) ](<https://lwn.net/Articles/685987/>)

Choi suggested exposing an interface for user space that would allow applications to set the stream IDs for writes. But Bottomley asked if there was really a need for a user-space interface. In the past, hints exposed to application developers went largely unused. It would be easier if the stream IDs were all handled by the kernel itself. He was also concerned that there would not be enough stream IDs available, so the kernel would end up using them all; none would be available to offer to user space. 

Martin Petersen said that he was not against a user-space interface if one was needed, but suggested that it would be implemented with `posix_fadvise()` or something like that rather than directly exposing the IDs to user space. Choi thought that applications might have a better idea of the lifetime of their data than the kernel would, however. 

At that point, Petersen took over to describe some research he had done on hints: how they are used and which are effective. There are several conduits for hints in the kernel, including `posix_fadvise()`, [ioprio](<https://www.kernel.org/doc/Documentation/block/ioprio.txt>) (available using [`ioprio_set()`](<http://man7.org/linux/man-pages/man2/ioprio_set.2.html>)), the `REQ_META` flag for metadata, NFS v4.2, SCSI I/O advice hints, and so on. There are tons of different hints available; vendors implement different subsets of them. 

So he wanted to try to figure out which hints actually make a difference. He asked internally (at Oracle) and externally about available hints, which resulted in a long list. From that, he pared the list back to hints that actually work. That resulted in half a dozen hints that characterize the data: 

  * Transaction - filesystem or database journals 
  * Metadata - filesystem metadata 
  * Paging - swap 
  * Realtime - audio/video streaming 
  * Data - normal application I/O 
  * Background - backup, data migration, RAID resync, scrubbing 
Of those, the background class made the most difference, as it ensures that a backup program doesn't steal I/O operations from real work, he said. 

The original streams proposal requires that the block layer request a stream ID from a device by opening a stream. Eventually those streams would need to be closed as well. For NVMe, streams are closely tied to the hardware write channels, which are a scarce resource. The explicit stream open/close is not popular and is difficult to do in some configurations (e.g. multipath). 

So Petersen is proposing a combination of hints and streams. Device hints would be set based on knowledge the kernel has about the I/O. The I/O priority would be used to set the background I/O class hint (though it might move to a `REQ_BG` request flag), other request flags (`REQ_META`, `REQ_JOURNAL`, and `REQ_SWAP`) would set those hints, and `posix_fadvise()` flags would also set the appropriate hints. 

Stream IDs would be based on files, which would allow sending the file to different devices and getting roughly the same behavior, he said. The proposal would remove the requirement to open and close streams and would provide a common model for all device types, so flash controllers, storage arrays, and shingled magnetic recording (SMR) devices could all make better decisions about data placement. This solution is being proposed to the standards groups as a way to resolve the problems with the existing hints and multi-stream specifications.
