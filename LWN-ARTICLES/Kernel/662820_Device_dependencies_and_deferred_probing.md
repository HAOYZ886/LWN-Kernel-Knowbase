---
title: Device dependencies and deferred probing
url: https://lwn.net/Articles/662820/
date: "November 3, 2015"
category: "Device drivers-Asynchronous probing"
author: "By Jonathan Corbet November 3, 2015 2015 Kernel Summit"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
November 3, 2015

* * *

[2015 Kernel Summit](<https://lwn.net/Articles/KernelSummit2015/>)

A "device" as seen by a user of a modern system is, often as not, a set of multiple independent devices working together. This organization is flexible, but it also can lead to problems for device drivers. How does a driver instantiate a new device if the other devices it depends on are not yet available? One answer is deferred probing — delaying the setup of a device until its dependencies show up. Mark Brown led a somewhat inconclusive Kernel Summit session on deferred probing and possible alternatives. 

The core problem, Mark said, is that there is no way to tell the kernel's device model about dependencies between devices. Rafael Wysocki responded that the real issue is dependencies between _drivers_ , but Mark disagreed, saying that a dependency shows itself as one device waiting for [![\[Mark Brown\]](https://static.lwn.net/images/conf/2015/klf-ks/MarkBrown-sm.jpg)](<https://lwn.net/Articles/662822/>) another specific device to be instantiated. He noted that there is currently [a deferred probing patch set](<https://lwn.net/Articles/658690/>) under discussion, but it doesn't address the full problem. In particular, it doesn't address suspend/resume operations (when devices must be suspended and resumed in the proper order), and it does nothing to speed boot time. 

Grant Likely said that the problem is with the tree-oriented device model, which is not able to describe the dependencies in real-world systems. This point was generally agreed — it has been understood for a while — but solutions are still hard to come by. Greg Kroah-Hartman described deferred probing as a "hack workaround" for the problem, but didn't immediately offer a less hacky alternative. 

There was some talk about the nature of device dependencies. Some of them are described in the system's firmware (or device tree). Others only show up when drivers initialize themselves and can't be seen at boot time. Dependencies can even change over the lifetime of the system as devices are configured in different ways. Tim Bird suggested using the [phandles](<http://wiki.laptop.org/go/Device_Tree_Hacking#Phandle_Properties>) (inter-device references) found in the system's device tree; there is a lot of dependency information to be found there. Most dependencies are "stupid clocks and regulators"; creating a graph of such dependencies prior to device probing would solve a lot of problems, he said. 

Greg, despite his earlier comment on deferred probing, now asked whether a more complex dependency graph was needed. Nobody has demonstrated real problems resulting from excessive deferral of device probing. Tim said that readily available dependency information could at least be used to perform hinting; Rafael agreed that pulling together as much information as possible would help the kernel to get the probing order roughly right. 

As a whole, the group seemed to agree that the problem is real, and that it should somehow be solved in the driver core. An important first step would be to come up with a way to register dependencies from the information that is already available. Rafael promised to post a proposal with some ideas; he [followed through](<https://lwn.net/Articles/662205/>) later that day.
