---
title: "LSFMM: Copy offload"
url: https://lwn.net/Articles/548347/
date: "April 24, 2013"
category: Block layer
author: "By Jake Edge April 24, 2013 LSFMM Summit 2013"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jake Edge**  
April 24, 2013

* * *

[LSFMM Summit 2013](<https://lwn.net/Articles/LSFMM2013/>)

Copy offload is a feature that allows filesystems or storage devices to be instructed to copy files without requiring involvement of the local CPU. In an unfortunately post-lunch session at the 2013 LSFMM Summit, Zach Brown, Martin Petersen, and Roland Dreier led a discussion of the feature. The unfortunate part was that I nearly succumbed to a food coma, so my notes are rather weak--apologies to readers and participants. 

There are three kinds of users for copy offload, Brown said, local filesystems like Btrfs, the NFS filesystem (so copies can be done on the server without involving the network), and SCSI-attached storage arrays, which could do a copy on the array itself. Trond Myklebust mentioned that he had an intern implement the functionality for NFS, which resulted in some "nice performance improvements" because the data was not copied down to the client. A big win for this feature is "copying giant files" like virtual machine images, as Ric Wheeler pointed out. 

Brown said that they want the "cleanest possible interface" for copy offload. It would be relatively straightforward to add the feature into the block layer stack, but "it wants to be asynchronous". That means adding a new system call that would return a cookie, which applications could use to poll with or block on awaiting completion. 

Dreier said that there are two operating system vendors who already ship support for the feature. VMWare uses the "EXTENDED COPY" SCSI command, while Windows 2012 uses a different set of SCSI commands in its ODX (Offloaded Data Transfer) feature. 

There are some atomicity questions that need to be answered as well, Brown said. For example, if a user creates a new file with the name of the destination of a ongoing copy offload, it is unclear what the right semantics should be. Joel Becker noted that getting an `EEXIST` a day after issuing a copy offload would be rather painful. 

Brown concluded by noting that patches would be forthcoming and that further discussion could be done on the mailing lists.
