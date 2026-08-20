---
title: "Little-endian PowerPC"
url: https://lwn.net/Articles/408845/
date: "October 6, 2010"
category: Architectures
author: "By Jonathan Corbet October 6, 2010"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
October 6, 2010

The PowerPC architecture is normally thought of as a big-endian domain - the most significant byte of multi-byte values comes first. Big-endian is consistent with a number of other architectures, but the fact that one obscure architecture - x86 - is little-endian means that the world as a whole tends toward the little-endian persuasion. As it happens, at least some PowerPC processors can optionally be run in a little-endian mode. Ian Munsie has posted [a patch set](<http://lwn.net/Articles/408051/>) which enables Linux to take advantage of that feature and run little-endian on suitably-equipped PowerPC processors. 

The first question that came to the mind of a few reviewers was: "why?" PowerPC runs fine as a big-endian architecture, and there has been little clamor for little-endian support. Besides, endianness seems to be one of those things that users can feel strongly about; to at least some PowerPC users, little-endian apparently feels cheap, wrong, and PCish. 

The [answer](<https://lwn.net/Articles/408848/>), as expressed by Ben Herrenschmidt, appears to be graphics hardware. A number of GPUs, especially those aimed at embedded applications, only work in the little-endian mode. Carefully-written device drivers can work around that sort of limitation without too much trouble, but user-space code - which often ends up talking to graphics hardware - is another story. Fixing all of that code is not a task that anybody wants to take on. As a result, PowerPC processors will not be considered for situations where little-endian support is needed. Running the processor in little-endian mode will nicely overcome that obstacle. 

That said, it will take a little while before this support is generally available. The kernel patches apparently look good, but there are toolchain changes required which are not, yet, generally available. Until that little issue is resolved, PowerPC will remain a club for big-endian users only.
