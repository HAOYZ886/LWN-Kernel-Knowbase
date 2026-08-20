---
title: Partial drive depopulation
url: https://lwn.net/Articles/684829/
date: "April 27, 2016"
category: Block layer
author: "By Jake Edge April 27, 2016 LSFMM 2016"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jake Edge**  
April 27, 2016

* * *

[LSFMM 2016](<https://lwn.net/Articles/lsfmm2016/>)

With today's large storage devices there are times when a component of the drive will fail (e.g. a head in a disk or a die in an SSD), which reduces the capacity of the device without rendering it completely unusable. But the arrangement of logical block addresses (LBAs) on the devices is such that the non-functioning LBAs are scattered across the device's address space. There is a need to "depopulate" (or "depop") those LBAs so that the rest of the device can continue to be used. Hannes Reinecke and Damien Le Moal led a combined storage and filesystem session at the 2016 Linux Storage, Filesystem, and Memory-Management Summit to discuss depop and how it should be handled by the kernel. 

[ ![\[Hannes Reinecke & Damien Le Moal\]](https://static.lwn.net/images/2016/lsf-rein-lamoal-sm.jpg) ](<https://lwn.net/Articles/685014/>)

Le Moal began by outlining the problem, noting that there are several types of components (head, surface, die, channel) that can go bad in a device without taking the entire device with them. The device will report the problem with a "unit attention" condition. One way to handle that is with offline logical depop, where the drive is simply reformatted to the new, smaller capacity. Reinecke said that would "not require a lot of work" to handle. 

The question of recovering data from the good portion of the device prior to reformatting came up. Ted Ts'o asked if there would be a list of bad sectors delivered to the kernel. Le Moal said there was a way for the host to get that list, but James Bottomley thought that sounded like an "awful lot of data to store in the kernel". For offline depop, though, the data would not need to be stored, Le Moal said. 

It is a large list, Fred Knight said, as the bad sectors are likely to be spread across the LBA range. Christoph Hellwig called the list "useless" to the kernel, but Knight said that if it was just needed for recovering the good data, the block list need not be stored. The problem is that disks are not uniform in the number of sectors per track across the drive and bad-block remapping can also complicate things. 

The discussion then turned to online logical depop, where the idea is to try to avoid reformatting the drive. The healthy LBAs would be kept intact, which would leave holes in the LBA space. The holes could be "amputated", removing them from the LBA range and never using them again. Or the blocks could be "regenerated" by allocating other blocks and remapping them into the holes. 

All of that seemed "overly complicated" to Ric Wheeler. He suggested that users would simply regenerate the filesystem from backups rather than fix the holes. They would truncate the size of the device and reformat it to get it back into production. The data still on the platters would just be ignored. 

Chris Mason agreed that users are likely to take the drive out of production, truncate and reformat it, then put it back. "Healing" drives is not an online process, he said. Wheeler said that he thought any work on online depop was likely a waste of time. 

But Knight said that a failure that only affected 10% of the drive would only take 10% of the time to rebuild, which might be attractive in some cases. Mason, though, felt that most would want some kind of verification step before bringing a partially failing drive back online. It may be true that it is simply one component that has failed, but that isn't truly known until the drive is examined and tested. Failing to do that, could result in a "bunch of borderline stuff" running in production, he said. 

Bottomley and Martin Petersen both said that a large discontiguous LBA range was not really usable. Wheeler summed up the feeling in the room by saying that offline depop is something that can be supported, but that unless the LBA regions were large or computable, they were not something that the kernel developers would use; "scatter-gather lists of LBAs" are not helpful.
