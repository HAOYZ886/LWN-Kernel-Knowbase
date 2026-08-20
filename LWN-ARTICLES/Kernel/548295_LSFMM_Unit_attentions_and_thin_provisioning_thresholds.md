---
title: "LSFMM: Unit attentions and thin provisioning thresholds"
url: https://lwn.net/Articles/548295/
date: "April 24, 2013"
category: SCSI
author: "By Jake Edge April 24, 2013 LSFMM Summit 2013"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jake Edge**  
April 24, 2013

* * *

[LSFMM Summit 2013](<https://lwn.net/Articles/LSFMM2013/>)

Handling "unit attention" conditions--essentially a class of SCSI device errors and alerts--specifically as they relate to the thresholds used for thin provisioning was the topic of a 2013 LSFMM Summit discussion led by Hannes Reinecke. He noted that Ewan Milne had [posted](<http://www.spinics.net/lists/linux-scsi/msg64230.html>) some unit attention (UA) handling patches recently that he thought were "mostly OK" and has been waiting to see them get merged. SCSI maintainer James Bottomley saw things a little differently. He said the "first few" patches in the set were fine, but there were a lot of other pieces entangled into the latter parts of patch set. Reinecke agreed that the UA handling could be untangled into a separate patch set. 

Some devices use UAs to indicate when they have hit their "soft threshold" for space consumption. They are meant to warn that the device has reached the limit set by an administrator so that more storage can be provisioned. But Linux currently doesn't do anything with those UAs, so Reinecke wanted to investigate what could or should be done with those alerts. 

One answer is that the SCSI layer should alert user space or an administrator about the condition, as Ric Wheeler suggested. Jeff Mahoney agreed, saying that some kind of "policy agent" in user space would make the decision about what to do. Reinecke said that Kay Sievers had done some work on turning UAs into udev events though there were concerns about swamping the system if there were many LUNs (logical unit numbers, which correspond to logical disks in SCSI array terms) all reporting the same condition. Someone noted that the flooding problem had been solved with support for only handling the attention from one LUN in an array. It only makes sense to tell user space if there is something it can do with the information, Reinecke said. 

There was a bit of digression into a discussion about how and where these thresholds get set. One possibility is to do it on the storage array at LUN creation time. It could be done array-wide from the host, but that is probably not wise, Reinecke said. It was suggested that the SCSI mode and log pages could be used to set the values, though that didn't seem very popular. As Martin Petersen pointed out, it needs to be done when the array is being set up. The array could be serving multiple hosts, with multiple operating systems, so setting it on the host side would be problematic. 

Reinecke said that the soft threshold is in some ways similar to the way Btrfs can "run out" of space when it actually hasn't. He wondered if the same user-space API should be used in both cases. Mahoney said that it made sense to handle both in one place. When using something like the "snapper" tool around a big set of changes such as a distribution upgrade will lead to twice the data storage, so old snapshots may need to be cleaned up, he said. 

James Bottomley asked what device would be used to signal the event to user space. Reinecke suggested the block device specified in the uevent, but it was noted that uevents are associated with kobjects, not block devices. Petersen suggested associating the event with the filesystem mount point, but Dave Chinner pointed out that there could be more than one of those. In the end, Bottomley's idea to "let udev sort it out" seemed popular. 

Ted Ts'o asked what a system administrator was supposed to do with the information. They need to free some space, but they will need to figure out which volume needs the space, so they will want to get an indication of which array has hit its limit. Petersen said that there already exists a protocol to handle this kind of notification: SNMP. There didn't seem to be a lot of support in the room for that particular option. 

In another semi-digression, Dave Chinner asked about handling the case where the storage has actually run out of space (e.g. hit its hard threshold), but might be able to free up space quickly. Right now, filesystems just see an `EIO` error for any writes "in flight", which means that they toss those writes out, even though they could potentially retry in a short time. What's needed is for the filesystem to get `ENOSPC` instead of `EIO`. 

While there is more context in the `struct bio`, Reinecke said, that is generally not known by filesystem developers, Chinner said. There is a bit of a disconnect in the communication channel between filesystem and storage developers, he said. Reinecke was thankful that they were all finally in the same room to get that cleared up, he said with a chuckle. 

But the idea of returning `ENOSPC` when the hard threshold is reached was popular, and in fact spawned another session later in the summit to see if there were more errors that could get similar treatment.
