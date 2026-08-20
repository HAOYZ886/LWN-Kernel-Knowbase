---
title: "Supporting multi-actuator drives"
url: https://lwn.net/Articles/753652/
date: "May 15, 2018"
category: Block layer
author: "By Jake Edge May 15, 2018 LSFMM"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jake Edge**  
May 15, 2018

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2018/>)

In a combined filesystem and storage session at the 2018 Linux Storage, Filesystem, and Memory-Management Summit (LSFMM), Tim Walker asked for help in designing the interface to some new storage hardware. He wanted some feedback on how a [multi-actuator drive](<https://blog.seagate.com/craftsman-ship/multi-actuator-technology-a-new-performance-breakthrough/>) should present itself to the system. These drives have two (or, eventually, more) sets of read/write heads and other hardware that can all operate in parallel. 

He noted that his employer, Seagate, had invested in a few different technologies, including host-aware [shingled magnetic recording](<https://en.wikipedia.org/wiki/Shingled_magnetic_recording>) (SMR) devices, that did not pan out. Instead of repeating those missteps, Seagate wants to get early feedback before the interfaces are set in stone. He was not necessarily looking for immediate feedback in the session (though he got plenty), but wanted to introduce the topic before discussing it on the mailing lists. Basically, Seagate would like to ensure that what it does with these devices works well for its customers, who mostly use Linux. 

[ ![\[Tim Walker\]](https://static.lwn.net/images/2018/lsf-walker-sm.jpg) ](<https://lwn.net/Articles/753665/>)

The device is a single-port serial-attached SCSI (SAS) drive, with the I/O going to two separate actuators that share a cache, he said. Both actuators can operate at full speed and seek independently; each is usable on a subset of the platters in the device. This is all based on technology that has already been mastered; it is meant to bring parallelism to a single drive. The device would present itself as two separate [logical unit numbers](<https://en.wikipedia.org/wiki/Logical_unit_number>) (LUNs) and each of the two actuator channels would map to its own LUN. Potential customers have discouraged Seagate from making the device be a single LUN and opaquely splitting the data between the actuators behind the scenes, Walker said.

One problem Walker foresees is that management commands, in particular those that affect the LUN as a whole, such as start and stop commands, could come addressed to either LUN but would affect the entire drive, thus the other LUN. Hannes Reinecke said that it would be better to have a separate LUN that was only for management commands rather than accepting management commands on the data LUNs. If not, though, making the stop commands do what is expected (park the heads if it is just for one LUN or spin down the drive if it is for both) would be an alternative. 

Fred Knight said that storage arrays have been handling this situation for years. They have hundreds of LUNs and have just figured it out and made it all work. He noted that, even though it may not be what customers expect, most storage arrays will simply ignore stop commands. The kernel does not really distinguish between drives and arrays, Martin Petersen said; there really is no condition where the kernel would want to stop one LUN and not the other. Knight said that other operating systems will spin down a LUN for power-management reasons, but that the standards provide ways to identify LUNs that are tied together, so there should not be a real problem here. 

Ted Ts'o said that a gathering like LSFMM (or the mailing lists) will not provide the full picture. Customers may have their own ideas about how to use this technology; the enterprise kernel developers may be able to guess what their customers might want to do, but that is only a guess. For the cloud, there is an advisory group that will give some input, he said, but it may be harder to get that for enterprises. Ric Wheeler said that he works for an enterprise vendor (Red Hat), which has internal users of disk drives (Ceph and others) that have opinions and thoughts that the company would be willing to share. 

From the perspective of a filesystem developer, all of what is being discussed is immaterial; the filesystem developers "don't care about any of this", Dave Chinner said. The storage folks will figure out how and when drives spin up and down (and other things of that nature), but the filesystems will just treat the device as if it were two entirely separate devices. Knight pointed out that there are some different failure modes that could impact filesystems; if the spindle motor goes, both drives are lost, while a head loss will lead to inaccessible data, but that may just be handled with RAID-5, for example. 

Ts'o noted that previously there had been "dumb drives and smart arrays", but that now we are seeing things that are between the two. Multi-actuator drives as envisioned by Seagate are just the first entrant; others will undoubtedly come along. It would be nice to standardize some way to discover the topology (spindles, heads, etc.) for these. Wheeler added that information about the cache would also be useful. 

This device has a shared cache, but devices with split caches might be good, Reinecke said. Kent Overstreet worried that there could be starvation problems if there are different I/O schedulers interacting in the same cache. As time wound down, Walker said that the session provided him with exactly the kind of feedback he was looking for.
