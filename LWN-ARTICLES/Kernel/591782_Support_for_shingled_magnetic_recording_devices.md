---
title: Support for shingled magnetic recording devices
url: https://lwn.net/Articles/591782/
date: "March 26, 2014"
category: Shingled magnetic recording
author: "By Jake Edge March 26, 2014 2014 LSFMM Summit"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jake Edge**  
March 26, 2014

* * *

[2014 LSFMM Summit](<https://lwn.net/Articles/LSFMM2014/>)

One of the plenary sessions on the first day of the Linux Storage, Filesystem, and Memory Management (LSFMM) Summit concerned Linux support for shingled magnetic recording (SMR) devices. These next-generation hard disks have a number of interesting characteristics that will be challenging to fully support. Martin Furuhjelm led a discussion among a few drive vendor representatives and the assembled kernel developers about the latest developments in SMR-land. 

There are three types of SMR drives: device managed, host aware, and host managed. Device-managed drives will essentially act just like regular disk drives, though the translation layer in the drive may cause unexpected performance degradation at times (much like flash devices today). Existing drivers don't need to change for device-managed disks. The discussion concentrated mostly on host-aware drives (where the host should _try_ to follow the requirements for shingled regions) and host-managed devices (where the requirements _must_ be followed). 

SMR drives will be made up of multiple zones, some that are "normal" and allow random reads and writes throughout the zone, and some that can only be written sequentially. For the sequential zones, there is a write pointer maintained for each zone that corresponds to where the next write must go. Depending on the mode, writing elsewhere in the zone will either be an error (in host-managed devices) or will lead to some kind of remapping of the write (for host-aware devices). That remapping may lead to latency spikes due to garbage collection at some later time. 

Two new SCSI commands have been added, one to query what zones exist on the drive and another to reset the write pointer to the beginning of a particular zone. To get the best performance, an SMR-aware driver will need to only write sequentially to the sequential zones (that will likely make up most of the disk), but if it fails to do so, it will be a fatal error only on host-managed drives. For that reason, most of the kernel developers seemed to think the first SMR drives are likely to be host-aware since those will work (though perhaps poorly at times) with today's software. 

The [T10 technical committee](<http://www.t10.org/>) (for SCSI interface standards) is currently working on finishing the standards for SMR, so it is important that Linux developers make any concerns they have with the drafts known soon. Ted Ts'o noted that the drafts are available from the T10 site (Furuhjelm recommended looking for "ZBC"). In addition, more information on SMR and Linux can be found in a [writeup](<https://lwn.net/Articles/548116/>) from last year's LSFMM. 

There were some questions about the zone reporting functionality, but much of that is still up in the air at this point. Currently, all zones are expected to be the same size, though there is a belief that will change before the draft is finalized. There has also been talk of adding a filtering capability on the query, so that only zones fitting a particular category (active, full, sequential-only, etc.) would be returned. 

The overall sense was that kernel developers are waiting for hardware before trying to determine how best to support SMR in Linux. No major complaints about the draft interface were heard, but until hardware hits, it will be difficult for anyone to determine where the problems lie. 

[ Thanks to the Linux Foundation for travel support to attend LSFMM. ]
