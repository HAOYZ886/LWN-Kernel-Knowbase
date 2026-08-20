---
title: "Accelerating netfilter with hardware offload, part 1"
url: https://lwn.net/Articles/809333/
date: "January 14, 2020"
category: "Device drivers-Network drivers; Networking-Packet filtering; Packet filtering"
author: "January 14, 2020 This article was contributed by Marta Rybczyńska"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

January 14, 2020

This article was contributed by Marta Rybczyńska

Supporting network protocols at high speeds in pure software is getting increasingly difficult, with 25-100Gb/s interfaces available now and 200-400Gb/s starting to show up. Packet processing at 100Gb/s [must happen in 200 cycles or less](<https://lwn.net/Articles/629155/>), which does not leave much room for processing at the operating-system level. Fortunately some operations can be performed by hardware, including checksum verification and offloading parts of the packet send and receive paths. 

As modern hardware adds more functionality, new options are becoming available. The 5.3 kernel includes a patch set from Pablo Neira Ayuso that [added support](<https://lwn.net/ml/netdev/20190709205550.3160-1-pablo%40netfilter.org/>) for offloading some packet filtering with [netfilter](<https://www.netfilter.org/>). This patch set not only adds the offload support, but also performs a refactoring of the existing offload paths in the generic code and the network card drivers. More work came in the following kernel releases. This seems like a good moment to review the recent advancements in offloading in the network stack.

#### Offloads in network cards

Let us start with a refresh on the functionality provided by network cards. A network packet passes through a number of hardware blocks before it is handled by the kernel's network stack. It is first received by the physical layer (PHY) processor that deals with the low-level aspects, including the medium (copper or fiber for Ethernet), frequencies, modulation, and so on. Then it is passed to the medium access control (MAC) block, which copies the packet to system memory, writes the packet descriptor into the receive queue, and possibly raises an interrupt. This allows the device driver to start the processing in the network stack.

MAC controllers, however, often include other logic, including specific processors or FPGAs, that can perform tasks far beyond launching DMA transfers. First, the MAC may be able to handle multiple receive queues that allow separating packet processing onto different CPUs in the system. It can also sort packets with the same source and destination addresses and ports, called "flows" in this context; different flows can be redirected to specific receive queues. This has performance benefits, including better cache usage. More than that, the MAC blocks can perform actions on flows, such as redirecting them to another network interface (when there are multiple interfaces in the same MAC), dropping packets in response to a denial-of-service attack, and so on.

The hardware behind that functionality includes two blocks that are important for netfilter offload: a parser and a classifier. The parser extracts fields from packets at line speed; it understands a number of network protocols, so that it can handle the packet at multiple layers. It usually extracts both well-known fields (like addresses and port numbers) and software-specified ones. In the second step the classifier uses the information from the parser to perform actions on the packet.

The hardware implementation of those blocks uses a structure called [ternary content-addressable memory](<https://en.wikipedia.org/wiki/Content-addressable_memory>) (TCAM), a special type of memory that uses three values (0, 1 and X) instead of the typical two (0 and 1). The additional X value means "don't care" and, in a comparison operation, it matches both 0 and 1\. A typical parser provides a number of TCAM entries, with each entry associated with another region of memory containing actions to perform. That implementation allows the creation of something like regular expressions for packets; each packet is compared in hardware with the available TCAM entries, yielding the index for any matching entries with the actions to perform.

The number of TCAM entries is limited. For example, controllers in Marvell SoCs like Armada 70xx and 80xx have a TCAM with 256 entries (covered in a [slide set [PDF]](<https://static.sched.com/hosted_files/osseu19/3d/chevallier-network-classification-offload.pdf>) from Maxime Chevallier's talk about adding support for classification offload to a network driver at the 2019 Embedded Linux Conference Europe). In comparison, netfilter configurations often include thousands of rules. Clearly, one of the challenges of configuring a controller like this is to limit the number of rules stored in TCAM. It is also up to the driver to configure the device-specific actions and different types of classifiers that might be available. The hardware available is usually complex and the drivers usually support only a subset of what is available.

Offload capabilities in MAC controllers can be more sophisticated than that. They include implementations of offloading for the complete TCP stack, called TCP offload engines. Those are currently not supported by Linux, as the code needed to handle them [raised many objections years ago](<https://lwn.net/Articles/148697/>) from the network stack maintainers. Instead of supporting TCP offloading, the Linux kernel provides support for specific, mostly stateless offloads.

Interested readers can find the history of the offload development in a [paper [PDF]](<http://vger.kernel.org/lpc_net2018_talks/Brandeburg_networking_hardware_offloads.pdf>) from Jesse Brandeburg and Anjali Singhai Jain, presented at the 2018 Linux Plumbers Conference.

#### Kernel subsystems with filtering offloads

The core networking subsystem supports a long list of offloads to network devices, including checksumming, scatter/gather processing, segmentation, and more. Readers can view the lists of available and active offload functionality on their machine with:

```
ethtool --show-offload <interface>
```

The lists will be different from one interface to another, depending on the features of the hardware and the associated driver. `ethtool` also allows configuring those offloads; [the manual page](<http://man7.org/linux/man-pages/man8/ethtool.8.html>) describes of some of the available features.

The other subsystem making use of hardware offloads is traffic control (tc with the configuration tool of the same name); the [`tc` manual page](<http://man7.org/linux/man-pages/man8/tc.8.html>) offers an overview of the available features, in particular the flower classifier, which allows administrators to set up scheduling of network packets. Practical examples of `tc` use include bandwidth limiting per service or adding priorities to some traffic. Interested readers can find more about tc flower offloads in an [article [PDF]](<https://netdevconf.info/2.2/papers/horman-tcflower-talk.pdf>) by Simon Horman presented at NetDev 2.2 in November 2017.

Up to this point, filtering offloads were possible with both `tc` and `ethtool`; these two features were implemented separately in the kernel. This duplication also required duplication of work by authors of network card drivers, as each offload implementation used different driver callbacks. With the advent of a third system adding offload functionality, the developers started working on common paths; this required refactoring some of the common code and changes in the callbacks to be implemented by the drivers. 

#### Summary

Network packet processing with high speed interfaces is not an easy task -- the number of CPU cycles available to do so is small. Fortunately, the hardware is offering offload capabilities that the kernel can use to ease the task. In this article we have provided an overview of how a network card works and some offload basics. This is to lay the foundations for the second part, where we're going to look into the details of the changes brought by the netfilter offloading functionality, both in the common code, and how it affects driver authors — and how to use the netfilter offloads, of course.
