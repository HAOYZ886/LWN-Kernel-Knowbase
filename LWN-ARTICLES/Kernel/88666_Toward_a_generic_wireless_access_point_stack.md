---
title: Toward a generic wireless access point stack
url: https://lwn.net/Articles/88666/
date: "June 9, 2004"
category: "HostAP; Networking-Wireless"
author: 
---

The Linux kernel has long had support for wireless networking. What the kernel does not have, however, is support for operation as a wireless access point. A standard Linux system has many of the required pieces (network bridging, DHCP service, etc.), but there are necessary functions that only the kernel can provide. These include WEP encryption (or some other protocol), access control, Wireless Distribution System support, etc. 

The mainline kernel may not support these capabilities, but that doesn't mean they don't exist. A few different implementations of the software necessary to create wireless access points are out there; each has been developed independently, and each tends to support only one family of wireless network cards. Anybody wanting to set up an access point needs to find the implementation best suited to the hardware at hand, patch the kernel, and put all of the pieces together. 

In an attempt to encourage the creation of a single access point support implementation in the kernel, Jeff Garzik has [announced](<https://lwn.net/Articles/87952/>) the creation of a new wireless patch set. He is starting with [HostAP](<http://hostap.epitest.fi/>), a widely-used software stack developed for Prism-based cards. It is, he says, the implementation which is best suited to being evolved into a generic wireless stack for the kernel. 

A number of the other access point implementations have taken chunks of code from HostAP, so it does seem like a good choice for a starting point. A fair amount of work may be required, however, to move it from being a driver for a specific set of cards to being a more generic implementation. Jeff hopes that this work can be done without a lot of core kernel changes; he would like to see the result merged into the 2.6 kernel. Now is the time for interested hackers to dive in and move the code in that direction.
