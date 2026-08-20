---
title: Filesystem support for SMR devices
url: https://lwn.net/Articles/637035/
date: "March 18, 2015"
category: Shingled magnetic recording
author: "By Jake Edge March 18, 2015 LSFMM 2015"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jake Edge**  
March 18, 2015

* * *

[LSFMM 2015](<https://lwn.net/Articles/lsfmm2015/>)

Two back-to-back sessions at the 2015 Linux Storage, Filesystem, and Memory Management Summit looked at different attempts to support Linux filesystems on shingled magnetic recording (SMR) devices. In the first, Hannes Reinecke gave a status report on some prototyping he has done to support SMR in Linux. The second was led by Adrian Palmer of Seagate about a project to port the ext4 filesystem to host-managed SMR devices. 

[ ![\[Hannes Reinecke\]](https://static.lwn.net/images/2015/lsf-reinecke-sm.jpg) ](<https://lwn.net/Articles/637081/>)

Reinecke described some prototyping he has done in the block layer to support SMR. Those devices have a number of interesting attributes that [require code in the kernel](<https://lwn.net/Articles/592091/>) to support. For example, SMR devices have multiple zones, some of which are normal random-access disk zones, while others must be written to sequentially. He has been looking specifically at supporting host-managed SMR devices, which require that the host never violate the sequential-write restriction in those types of zones. 

SMR drives disallow I/O that spans zones, Reinecke said, which means that I/O operations need to be split at those boundaries. The zone layout could have a different size for each of the different zones, though none of the drives currently does that. To support that possibility, though, he used a [red-black tree](<https://lwn.net/Articles/184495/>) to track all of the zones. The current SMR specification allows for deferred lookup of some of the zone information, so the tree could just be partially filled for devices with lots of irregular zones. 

Ted Ts'o suggested that supporting "insane drives" that have a variety of zone sizes might use a different data structure. That way, the majority of drives that have a straightforward layout could have all of that information available in kernel memory. He was concerned that there might be I/O performance degradation when issuing the "report zones" command once the device has been mounted. 

There is also a question about "open zones" and the maximum number of open zones. Reinecke said that it is a topic that is still under discussion among the drive makers. From the LSFMM discussion it seems clear that there is no agreement on what an open zone is. Some believe that any partially filled zone qualifies, while to others it means zones that are simultaneously available to write to. In addition, the maximum may range from the four to eight that Martin Petersen has heard to the 128 that the drive makers have proposed. 

In fact, someone from one of the storage vendors asked what the kernel developers would like the maximum to be. The reply was, not surprisingly, "all of them". Reinecke said that he is lobbying that "zone control" (maximum number of open zones) be optional and that any I/O that violates the maximum open zones should be allowed, possibly with a performance penalty. Ts'o agreed with that, saying that writing to one more zone than is allowed must not cause an I/O error, though adding some extra latency would be acceptable. Reinecke said that he had hoped to avoid the whole topic of open zones "because it is horrible". 

Reinecke then moved back to his prototype work. He noted that sequential writes must be guaranteed. Each sequential zone has its own write pointer, which is where the next write for that zone must be. That "sort of works" using the NOP I/O scheduler, since it just merges adjacent writes. If out-of-order writes from multiple tasks are encountered, they can be requeued at the tail of the queue. The queue size must be monitored, he said, since if it never gets smaller, the I/O is making no progress, which should cause an I/O error. 

But Dave Chinner said that once a filesystem has allocated blocks to different tasks, it must then guarantee an ordering of those writes "all the way down". The only way to do that is to serialize the I/O to the zone once the allocation has been done. Reinecke said that requeueing at the tail can solve that problem, but Chinner said that in a preemptible kernel that won't work. "Sequential I/O is basically synchronous I/O", he said. 

There is a philosophical question about whether it makes sense to try to put a regular filesystem on SMR devices, Ts'o said. Chinner said that SMR is really a firmware problem. Actually solving the problems of SMR at the filesystem level is not really possible, he said. 

Reinecke wondered if the host-managed SMR drives would actually sell. Petersen piled on, noting that the flash-device makers had made lots of requests for extra code to support their devices, but that eventually all of those requests disappeared when those types of devices didn't sell. Reinecke's conclusion was that it may not make a lot sense to try to make an existing filesystem work for host-managed SMR drives. 

#### Ext4 on host-managed SMR

On the other hand, though, Palmer is quite interested in doing just that. He works on host-managed drives and is trying to get ext4 working on them. 

He started by looking at block groups as a way to track the zones, but ran into a problem with that idea. Zones are 256MB in length, but a 4KB block only has enough bits to address 128MB worth of blocks, so he would need to use 8KB blocks, which is a sizable change. He also noted that `O_DIRECT` I/O was going to be a problem for host-managed SMR, without really going into any details. 

As Reinecke said earlier, the order of writes to the disk is critical for host-managed drives. Out-of-order writes may not be written at all. Palmer looked at putting the code to keep write operations sequential into either the I/O scheduler or the block device. For now, the block device seems to be the right place. 

Ts'o said that he is mentoring a student who is working on making the ext4 journal writes more SMR-friendly. But Chinner is worried about `fsck`. A corrupt block in the middle of a sequential zone may need to be rewritten, but it can't be overwritten in place. Ts'o suggested a 256MB read-modify-write with a chuckle. 

One attendee noted that the drive makers want to start with host-aware drives (which will perform better with mostly sequential writes to those zones, but will not fail out-of-order writes) to get them working. That will allow the companies to learn from the market how much conventional space (zones without the sequential requirement) and overprovisioning is required. 

Chinner suggested that some of that conventional space might be used for metadata sections. Another attendee cautioned that SSD makers are also looking at zone block devices, so it may be more than just SMR drives that need this kind of support. But Chinner said that the kernel developers had "more than enough" on their hands rewriting filesystems for use on SMR. 

Another way to approach the problem, Chinner said, might be to have a new kind of write command for disks (perhaps "write allocate") that would return the logical block address (LBA) where the data was written, rather than getting the LBA from the filesystem or block layers with the write. That way, the drive would decide where to place the data and return that to the operating system. One attendee said that the driver vendors would probably welcome a discussion about what the API to these drives would look like. 

There was some discussion on how to proceed with a new command, which would (eventually) need to be handled by the [T10 committee](<http://www.t10.org/>) (for SCSI interface standards). Petersen (who represents Linux on T10) noted that it is difficult to change the standard. An attendee from one of the drive makers thought it might be possible to prototype the idea to try it out completely separate from the standards process. 

That is where the conversation trailed off, but the "write allocate" idea seemed to generate some interest. Whether that translates into action (or standards) remains to be seen. 

After the summit, on March 16, Dave Chinner [posted](<https://lwn.net/Articles/637148/>) a pointer to a design document on supporting XFS on host-aware SMR drives. 

[I would like to thank the Linux Foundation for travel support to Boston for the summit.]
