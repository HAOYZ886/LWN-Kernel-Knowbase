---
title: Char devices for network interfaces
url: https://lwn.net/Articles/356899/
date: "October 14, 2009"
category: "Device drivers-Network drivers; Networking"
author: "By Jonathan Corbet October 14, 2009"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
October 14, 2009

One of the longstanding quirks of BSD-inspired networking is that network interfaces are a strange sort of device. They live in their own namespace, do not appear in `/dev`, and generally fail to live up to the "everything is a file" idea that drives much of the POSIX API. That said, the Unix way of networking has functioned well for nearly 30 years. It is likely that few people were expecting a serious patch which tries to change things. 

[This patch](<http://lwn.net/Articles/356541/>) from "Narendra K" at Dell does exactly that, though, and in surprising ways. With this patch in place, every network interface gets an associated char device. It is a singularly useless device: any attempt to open it just returns `ENOSYS`. The only real reason for this device's existence, it turns out, is to generate events for udev which, in turn, can generate alternative names for the interface. 

Why this change? System vendors and administrators are getting tired of their network interfaces changing name at each boot. Non-deterministic interface naming is the result of a few factors, including weird BIOS behavior and the way current kernels enumerate devices via a parallel hot-plug approach. When interfaces change names, configuration scripts can get confused; the end result is rarely a working network. Some of this confusion can be avoided by carefully configuring interfaces based on their MAC address, but that, too, can fail in the face of "swap out the entire server" approach to fast failure recovery. 

Vendors have tried to work around some of these difficulties at the hardware level. Dell machines are designed to enumerate network interfaces in the same order as often as possible. HP blade servers can configure interface MAC addresses at power-on time. But there are limits to how many hardware hacks the vendors are willing to add to deal with this problem. [This message from Matt Domsch](<https://lwn.net/Articles/356900/>) is recommended reading for anybody wanting a full understanding of the difficulties here. 

Thus the patch, which allows udev to create pseudo-names for each interface based on criteria like the PCI slot number, chassis label, or anything else that seems to make sense. The patch is tied to the [libnetdevname library](<http://linux.dell.com/wiki/index.php/Oss/libnetdevname>), which maps these pseudo-names into the real interface name, which can then be used with the socket system calls. 

The patch has gotten a bit of a rough reception; it looks to some like a hack for problems which can be solved in other ways. The discussion has made it clear that a real problem exists, though, so some sort of solution will likely be applied in the end. Udev seems like the place for this solution to happen - that is how naming has been handled for every other device in the system, after all. So expect something to get in eventually, though the current may evolve somewhat before it is considered ready for merging.
