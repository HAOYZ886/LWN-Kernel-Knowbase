---
title: IMA memory hog
url: https://lwn.net/Articles/410895/
date: "October 20, 2010"
category: Integrity measurement architecture
author: "By Jonathan Corbet October 20, 2010"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
October 20, 2010

Dave Chinner recently [noticed a problem](<https://lwn.net/Articles/410896/>) on one of the kernel.org systems: the slab cache was using well over 2GB of memory, mainly on radix tree nodes. Intrigued, he looked further into the problem. It came down to the integrity measurement architecture (IMA) security code, which uses the hardware TPM to help ensure that files on the system have not been tampered with. IMA, it seems, was using a [radix tree](<http://lwn.net/Articles/175432/>) to store integrity information, indexed by inode address. Radix trees perform poorly with sparse, unclustered keys, so IMA's usage was causing the creation of a separate node for each inode in the system. That added up to a lot of memory. 

A number of questions came after this revelation, including: 

  1. Why is IMA using such an inappropriate data structure? 
  2. Why is it keeping all this information around even though it was disabled on the system in question? 
  3. Why was IMA configured into the kernel in the first place? 

The answer to the first question seems to be that the IMA developers, as part of the process of getting the code into the mainline, were not allowed to expand the inode structure at all. So they created a separate tree for per-inode information; it just happens that they chose the wrong type of tree and never noticed how poorly it performed. 

Question #2 is answered like this: the IMA code needs to keep track of which files are opened for write access at any time. There is no point in measuring the integrity of files (checksumming them, essentially) when they can be changed at any time. Without tracking the state of all files all the time, IMA can never know which files are opened for write access when it first starts up. The only way to be sure, it seems, is to track all files starting at boot time just in case somebody tries to turn IMA on at some point. 

As for #3: kernel.org was running a Fedora kernel, and the Fedora folks turned on the feature because it looked like it might be useful to some people. Nobody expected that it would have such an impact on systems where it was not turned on. Some participants in the discussion have given the Fedora kernel maintainers some grief for not having audited the code before enabling it, but auditing everything in the kernel to that level is a bit larger task than Fedora can really be expected to take on. 

Eric Paris has started work on slimming IMA down; his [patches](<https://lwn.net/Articles/410902/>) work by moving the "open for write" counts into the inode structure itself, eliminating the need to allocate the separate IMA structures most of the time. IMA is also shifted over to a red-black tree when it does need to track those structures. This work eliminates almost all of the memory waste, but at the cost of growing the inode structure slightly. That does not sit well with everybody, especially, it seems, those developers who feel that IMA should not exist in the first place. But it's a clear step in the right direction, so one should expect something along these lines to be merged for 2.6.37.
