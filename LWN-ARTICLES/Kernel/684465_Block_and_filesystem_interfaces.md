---
title: Block and filesystem interfaces
url: https://lwn.net/Articles/684465/
date: "April 26, 2016"
category: Block layer
author: "By Jake Edge April 26, 2016 LSFMM 2016"
---

By **Jake Edge**  
April 26, 2016

* * *

[LSFMM 2016](<https://lwn.net/Articles/lsfmm2016/>)

Steven Whitehouse led a discussion about the interfaces between the block subsystem and filesystems in a combined storage and filesystem session at the 2016 Linux Storage, Filesystem, and Memory-Management Summit (LSFMM). In some ways, the discussion was a catch-all for topics that were not slated for their own session during the two-day summit. 

For a long time, the block interface was not that complicated, Whitehouse said. But, over the last few years, there have been more and more device types (both real and virtual), more filesystem choices, as well as features like encryption, compression, copy on write, copy offload, and so on. All of these have an impact on the block and filesystem interfaces. 

[ ![\[Steven Whitehouse\]](https://static.lwn.net/images/2016/lsf-whitehouse-sm.jpg) ](<https://lwn.net/Articles/684884/>)

He believes that getting the interfaces right should be the primary consideration that will ensure maximum interoperability. It allows modularization, which is more than simply a good engineering principle; things that don't work can simply be replaced. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

One of the changes that has come about is "dynamic devices", which are devices that can change their attributes after they have been mounted. [Thin provisioning](<https://lwn.net/Articles/592645/>) is a good example of that, he said. For example, the device mapper thin-provisioning (dm-thin) module needs a way for filesystems to communicate their requirements for space that has been allocated, but not written to, so that dm-thin can arrange to have that storage present (which might require operator intervention). Otherwise, operations that should always succeed (because of an earlier `fallocate()`, say) might return `ENOSPC`. 

Jan Kara said that there had been talk of an interface to provide dm-thin with the information it needs, but it might be better to use notifiers. Al Viro agreed and said that notifiers would also provide a more natural way to inform the other layers that the device topology had changed in some way. 

Mike Snitzer said that XFS is now working with dm-thin using a new reservation interface to ask that some space be set aside for the filesystem to use. It is nice to have that software interface, but he wondered if there was an equivalent way to reserve space on the hardware. Martin Petersen said that there is an "anchor" facility that can guarantee that certain logical block addresses (LBAs) are available for writes in the future. 

That led to a quick discussion of what is really needed. Snitzer said that specific LBA numbers are not important, as he is just looking for a reservation of a certain amount of space. Kara said that the LBAs are not known when writes go into the page cache, but the amount of space required is known. But reserving blocks on the device can lead to other problems. Fred Knight asked, when does the storage device know that it can release those blocks if the OS or application crashes? There was a thought that perhaps this could be combined with streams (as was discussed in the [standards update](<https://lwn.net/Articles/684264/>) earlier in the day) and a timeout, which Knight said had been talked about in the standards bodies along the way. 

Whitehouse then moved to another topic: error reporting. In particular, what changes might be needed to support thin provisioning and other new types of devices. There is a need to inform other layers of changes in the topology of the storage, but also to report when certain kinds of operations are not supported, such as shrinking a filesystem while it is mounted. There was general agreement that work was needed in this area, but no concrete plans emerged. 

[Shingled magnetic recording](<https://lwn.net/Articles/637035/>) (SMR) was up next. Whitehouse noted that it was the only new device type that didn't have its own session at LSFMM. Hannes Reinecke said that he didn't want to bore the attendees with yet another SMR session. He has posted [patches](<https://lwn.net/Articles/682361/>) that add a [red-black tree](<https://lwn.net/Articles/184495/>) to the request queue to track the write pointer in the various zones of the device. There have been no comments, "so it must be OK", he said. 

Reinecke mentioned that those patches also map the "reset write pointer" command to the existing discard functionality, since they are closely related. Whitehouse wondered about that because discard is more of a hint than a command. Petersen noted that currently the `ioctl()` commands are implemented directly as calls to the block library functions, but that should change. More specific building blocks should be used to implement those `ioctl()` commands. 

Ted Ts'o brought up a problem he had run into on mobile phones with inline cryptographic processors. Those processors keep the actual key information internally; the kernel just gets a key ID that it uses to decrypt a block device. Device mapper does not currently support that, but he would like to implement the functionality for the mainline kernel in a cleaner way than was done for the phone. 

Snitzer wondered why he had never heard of the problem, since he is a device mapper maintainer. Ts'o said that it was something that was done quickly internally, but he hadn't gotten around to filing bugs and the like. He suggested that perhaps the data integrity (DIF/DIX) support in the device mapper might be used instead, but was concerned that the device mapper support might be lacking. 

Darrick Wong said that device mapper does work with DIF/DIX, so it should in theory be possible to use it. He did not know how much "rigorous testing" it had seen, however. Snitzer said that he thought device mapper had "a pretty good story" with respect to using DIF/DIX and stacking protection profiles. 

Ts'o did mention another problem, though. Getting a key ID from the inline processor would involve an [ARM TrustZone](<http://www.arm.com/products/processors/technologies/trustzone/>) operation that could be slow, since it might require getting a PIN from a user. That part shouldn't go into device mapper, he said. 

As time for the session expired, Whitehouse asked about the status of [copy offload](<https://lwn.net/Articles/637436/>). Petersen said that the support for token-based copies is ready. It is awaiting some [block layer cleanups](<https://lwn.net/Articles/683750/>) from Mike Christie, but is in "pretty good shape" overall. He said the [`copy_file_range()`](<https://lwn.net/Articles/659523/>) interface will be available in the 4.6 kernel.
