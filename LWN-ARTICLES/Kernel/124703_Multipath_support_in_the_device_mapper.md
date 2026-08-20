---
title: Multipath support in the device mapper
url: https://lwn.net/Articles/124703/
date: "February 23, 2005"
category: Device mapper; Multipath IO
author: 
---

Multipath connectivity is a feature of high-end storage systems. A storage box packed with disks will be connected to multiple transport paths, any one of which can be used to submit I/O requests. A computer will be connected to more than one of these transport interconnects, and can choose among them when it has an I/O request for the storage server. This sort of arrangement is expensive, but it provides for higher reliability (things continue to work if an interconnect fails) and better performance. 

Support for multipath in Linux has traditionally been spotty, at best. Some low-level block drivers have included support for their specific devices, but support at that level leads to duplicated functionality and difficulties for administrators. Some thought has gone into how multipath is best supported: does that logic belong at the driver layer, the SCSI mid-layer, the block layer, or somewhere else? The conclusion that was reached at last year's Kernel Summit was that the device mapper was the best place for multipath support. 

That support has now been coded up and [posted](<https://lwn.net/Articles/123132/>) for review; it was added to the 2.6.11-rc4-mm1 kernel. When used with the user-space [multipath tools](<http://christophe.varoqui.free.fr/>) distribution, the device mapper can now provide proper multipath support - for some hardware, at least. 

Internally, the multipath code creates a data structure, attached to a device mapper target, which looks like this: 

![\[Cheezy multipath diagram\]](https://static.lwn.net/images/ns/kernel/multipath.png)

When time comes to transfer blocks to or from a device mapper target representing a multipath device, the code goes to the first priority group in the list. Each group represents a set of paths to the device, each of which is considered equal to the others; the preferred paths (being the fastest and/or most reliable) should be contained in the first group in the list. Priority groups include a path selector - a function which determines which path should be used for each I/O request. The current patches include a [round-robin selector](<https://lwn.net/Articles/123133/>) which simply rotates through the paths to balance the load across them. Should situations arise which require more complicated policies, it should not be tremendously difficult to create an appropriate path selector. 

If a given path starts to generate errors, it is marked as failed and the path selector will pass over it. Should all paths in a priority group fail, the next group in the list (if it exists) will be used. The multipath tools include a management daemon which is informed of failed paths; its job is to scream for help and retry the failed paths. If a path starts to work again, the daemon will inform the device mapper, which will resume using that path. 

There may be times when no paths are available; this can happen, for example, when a new priority group has been selected and is in the process of initializing itself. In this situation, the multipath target will maintain a queue of pending BIO structures. Once a path becomes available, a special worker thread works through the pending I/O list and sees to it that all requests are executed. 

At the lower level, the multipath code includes [a set of hardware hooks](<https://lwn.net/Articles/123134/>) for dealing with hardware-specific events. These hooks include a status function, an initialization function, and an error handler. The patch set includes [a hardware handler](<https://lwn.net/Articles/123135/>) for EMC CLARiiON devices. 

Comments on the patches have been relatively few, and have dealt mostly with trivial issues. The multipath patches are unintrusive; they add new functionality, but do not make significant changes to existing code. So chances are good that they could find their way into the 2.6.12 kernel.
