---
title: Stream ID status update
url: https://lwn.net/Articles/717755/
date: "March 29, 2017"
category: Block layer
author: "By Jake Edge March 29, 2017 LSFMM 2017"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jake Edge**  
March 29, 2017

* * *

[LSFMM 2017](<https://lwn.net/Articles/lsfmm2017/>)

Stream IDs as a way for the host to give storage devices hints about what kind of data is being written have been [discussed before](<https://lwn.net/Articles/685499/>) at LSFMM. This year, Andreas Dilger and Martin Petersen led a combined storage and filesystem session to update the status of the feature. 

[ ![\[Andreas Dilger\]](https://static.lwn.net/images/2017/lsfmm-dilger-sm.jpg) ](<https://lwn.net/Articles/717739/>)

Dilger began by noting that the feature looked like it was moving forward and would make its way into the kernel, but hasn't. There are multiple use cases for it, including making it easier for SSDs to decide where to store data to reduce the amount of copying needed when garbage collecting. It would also help developers using [`blktrace`](<http://man7.org/linux/man-pages/man8/blktrace.8.html>) to do analysis at the block layer and could help [bcachefs](<http://bcachefs.org/>) make better decisions about what to put in flash or on disk. 

Embedding a stream ID in block I/O requests would help with those cases and more, he said. It would allow all kinds of storage to make better allocation and scheduling decisions. But development on it seems to have gone quiet, so he was hoping to get an update from Petersen (and the others in the room) on the status of stream IDs. 

Petersen said that he ran some benchmarks using stream IDs and "all the results were great". But the storage vendors seem to have lost interest. They are off pursuing deterministic writes, he said. Deterministic writes are a way to avoid the performance hiccups caused by background tasks (like wear leveling and garbage collection) by writing in the "proper" way. 

But Jens Axboe thought that that stream IDs should still be worked on. He would like to see a small set of stream IDs (two, perhaps) that simply gave an advisory hint of whether the data is likely to be short-lived or long-lived. That would mean there don't need to be a bunch of different flags that would need to be agreed upon and defined. He prefers to simply separate data with different deletion characteristics. 

[ ![\[Martin Petersen\]](https://static.lwn.net/images/2017/lsfmm-petersen-sm.jpg) ](<https://lwn.net/Articles/717753/>)

Dilger said that filesystems could provide more information that might help the storage devices make even better decisions on data placement. Some fairly simple information on writes of metadata versus user data would help. Axboe wondered if an API should be exposed so that applications could tell the kernel what kind of data they were writing, but Dilger thought that the kernel is able to provide a lot of useful information on its own. 

Ted Ts'o asked if it would be helpful to add a 32-bit stream ID to `struct bio` that `blktrace` would display. Petersen said he had been using 16-bit IDs because that's what the devices use, but more bits would be useful for tracing purposes. Dilger said that he didn't want the kernel implementation to be constrained by the hardware; there will need to be some kind of mapping of the IDs in any case. The only semantic that would apply is that writes with the same ID are related to each other in some fashion. 

The hint that really matters is short-lived versus not short-lived, Axboe believes. So it makes sense to just have a simple two-stream solution. That will result in 99% of the benefit, he said. But an attendee said that only helps for flash devices, not shingled magnetic recording (SMR) devices and others. In addition, Ts'o thought that indicating filesystem journal writes was helpful. Petersen agreed that it made a big difference for SMR devices. 

Axboe said that he had a patch set from about a year ago that he will dust off and post to the list soon. The discussion whether an API is needed and, if so, what it should look like, can happen on the mailing list. Once the kernel starts setting stream IDs, though, there may be performance implications that will need to be worked out. In some devices, the stream IDs are closely associated with I/O channels on the device, so that may need to be taken into account.
