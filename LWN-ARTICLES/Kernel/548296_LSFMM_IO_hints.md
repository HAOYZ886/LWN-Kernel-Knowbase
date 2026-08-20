---
title: "LSFMM: I/O hints"
url: https://lwn.net/Articles/548296/
date: "April 24, 2013"
category: Block layer; SCSI
author: "By Jake Edge April 24, 2013 LSFMM Summit 2013"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jake Edge**  
April 24, 2013

* * *

[LSFMM Summit 2013](<https://lwn.net/Articles/LSFMM2013/>)

At the 2013 LSFMM Summit, Martin Petersen led a discussion of the proposed "hints"--indications of how the storage is being used--for the T10 SCSI block command (SBC) standard. These hints "keep coming up" when he talks to storage and flash vendors, but the vote on them in the T10 committee was postponed until a later ballot. Petersen said he was looking for feedback from filesystem developers on which of the hints--describing access patterns, caching, data placement, and other attributes--would be useful. He is trying to identify which hints could be usefully passed into the block I/O stack via `struct bio`. 

Petersen put up a list of the hints implemented or used by Linux, NFS, and Windows, along with those proposed for T10 SBC. Some of the hints themselves were questioned, including "SEQUENTIAL_BACKWARDS" for NFS, which Ric Wheeler wondered about: are there really applications that need to do that? It turns out that some unnamed database does actually have that access pattern. 

But beyond that, there are questions of interpreting the hints. As Ted Ts'o asked: how sequential is "sequential" and how frequent is "frequent". He also asked about the "READ/WRITE RANDOMNESS" hints proposed for T10. That, at least, has an answer: it is a two-byte value that indicates how likely a given logical block address (LBA) will be read or written randomly within an LBA range, Petersen said. 

Dave Chinner said that the question comes down to what user space will find useful because filesystems just get hints from `fadvise()`. The hints that user space provides via `fadvise()` are what the filesystem can pass down to the storage. Petersen wanted to know if there are hints that could be added, and Wheeler noted that filesystems are really an application to the storage subsystems. But Boaz Harrosh thought that kind of thinking was a "pyramid standing on its head"; the "smarts" reside at the upper layers, never at the lower, he said, so the hints should just be ignored as a "layering violation". 

Ts'o noted that the hints tend to tie filesystem developers in knots because the meaning is undefined at the storage layer, which makes it hard to give it any meaning above that. The T10 stack is so abstract that filesystems and application developers have no idea what the storage will do with the hints, he said. 

But the hints are also fairly specific to "spinning rust", Roland Dreier said, so adding more hints won't really help. Petersen countered that tagging data consistently will allow the storage vendors to eventually figure things out. For example, he said, giving hints on metadata and nothing else might lead to better performance. 

But hinting will just lead to application problems, Harrosh said. Each vendor will treat the hints differently, based on a single application that is important to them. That will lead to a feedback loop so that applications are tuned for specific storage vendors. Dreier said that with his "array vendor hat on", he would ignore the hints entirely. That's fine, Petersen said, as other devices will at least have the opportunity to act on the hints. 

One use case that Petersen described involved a "well-known database from a well-known company" that does a lot of random I/O. It would like to be able to back up the data sequentially, but without having that data get cached, so that it wouldn't impact performance of the normal database processing. Another is for Btrfs, which can do deduplication and compression, so it would make sense for it to tell that to the storage and avoid wasted effort at that level.
