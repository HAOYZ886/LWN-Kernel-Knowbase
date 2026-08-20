---
title: Distributed storage
url: https://lwn.net/Articles/246272/
date: "August 21, 2007"
category: Block layer; Device mapper
author: "By Jonathan Corbet August 21, 2007"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
August 21, 2007

Evgeniy Polyakov is not an easily discouraged developer. He has been the source of a great deal of interesting kernel code - including a network channels implementation, an asynchronous crypto framework, the kevent subsystem, the "network tree" memory management layer, and the netlink connector code. Of all of those patches, only the netlink connector has made it into the mainline kernel - and that was back in 2005. Undeterred, Evgeniy has come forward with another significant patch set for consideration. His ambitions are no lower this time around: he would like to replace much of the functionality offered by the device mapper, iSCSI, and network block device (NBD) layers. He calls the new subsystem [distributed storage](<http://lwn.net/Articles/245566/>), or DST for short. The goal is to allow the creation of high-performance storage networks in a reliable and easy manner. 

At the lowest level, the DST code implements a simple network protocol which allows block devices to be exported across a network. The number of operations supported is small: block read and write operations and a "how big is your disk?" information request is about it. But it is intended to be fast, non-blocking, and able to function without copying the data on the way through. The zero-copy nature of the code allows it to perform I/O operations with no memory allocations at all - though the underlying network subsystem might do some allocations of its own. 

There is no data integrity checking built into the DST networking layer; it relies on the networking code to handle that aspect of things. There is also no real security support at all. If a block device is exported for use in DST, it is exported to anybody who can reach the host. The addition of explicit export lists could certainly be done in the future, but, for now, hosts exporting drives via DST are probably best not exposed to anything beyond an immediate local network. 

The upper layer of the DST code enables the creation of local disks. A simple `ioctl()` call would create a local disk from a remote drive, essentially reproducing the functionality offered by NBD. Evgeniy claims better performance than NBD, though, with non-blocking processing, no user-space threads, and a lack of busy-wait loops. There is also a simple failure recovery mechanism which will reconnect to remote hosts which go away temporarily. 

Beyond that, though, the DST code can be used to join multiple devices - both local and remote - into larger arrays. There are currently two algorithms available: linear and mirrored. In a linear array, each device is added to the end of what looks like a much larger block device. The mirroring algorithm replicates data across each device to provide redundancy and generally faster read performance. There is infrastructure in place for tracking which blocks must be updated on each component of a mirrored array, so if one device drops out for a while it can be quickly brought up to date on its return. Interestingly, that information is not stored on each component; this is presented as a feature, in that one part of a mirrored array can be removed and mounted independently as a sort of snapshot. Block information also does not appear, in this iteration, to be stored persistently anywhere, meaning that a crash of the DST server could make recovery of an inconsistent mirrored array difficult or impossible. 

Storage arrays created with DST can, in turn, be exported for use in other arrays. So a series of drives located on a fast local network can be combined in a sort of tree structure into one large, redundant array of disks. There is no support for the creation of higher-level RAID arrays at this time. Support for more algorithms is on the "to do" list, though Evgeniy has said that the Reed-Solomon codes used for traditional RAID are not fast enough for distributed arrays. He suggests that [WEAVER codes](<http://www.usenix.org/events/fast05/tech/full_papers/hafner_weaver/hafner_weaver_html/weaver_fast05.html>) might be used instead. 

At this level, DST looks much like the device mapper and MD layers already supported by Linux. Evgeniy claims that the DST code is better in that it does all processing in a non-blocking manner, works with more network protocols, has simple automatic configuration, does not copy data, and can perform operations with no memory allocations. The zero-allocation feature is important in situations where deadlocks are a worry - and they are often a worry when remote storage is in use. Making the entire DST stack safe against memory-allocation deadlocks would require some support in the network layer as well - but, predictably, Evgeniy [has some ideas](<https://lwn.net/Articles/246283/>) for how that can be done. 

This patch set is clearly in a very early state; quite a bit of work would be required before it would be ready for production use with data that somebody actually cares about. Like all of Evgeniy's patches, DST contains a number of interesting ideas. If the remaining little details can be taken care of, the DST code could eventually reach a point where it is seen as a useful addition to the Linux storage subsystem.
