---
title: Plenary day lightning talks
url: https://lwn.net/Articles/572129/
date: "October 29, 2013"
category: "Security-Random number generation"
author: "By Jonathan Corbet October 29, 2013 2013 Kernel Summit"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
October 29, 2013

* * *

[2013 Kernel Summit](<https://lwn.net/Articles/KernelSummit2013/>)

The 2013 Kernel Summit closed out with a set of lightning talks covering a wide range of topics including random numbers, configuration options, and ARM board testing. 

Ted Ts'o started things off with a discussion of the kernel's random number generators. There has been, he noted dryly, a significant increase in interest in the quality of the kernel's random numbers recently. His biggest concern in this area remains embedded devices that create cryptographic keys shortly after they boot; there may be little or no useful entropy in the system at that point. Some fixes have been added recently, including adding more entropy sources and mixing in system-specific information like MAC addresses, but that still may not be enough entropy to do a whole lot of good. That can be especially true on systems where the in-kernel `get_cycles()` function returns no useful information, a problem which was [covered here](<https://lwn.net/Articles/567055/>) in September. 

MIPS was one of the architectures that had just that problem. Since MIPS chips are used in devices like home routers, this is a real concern. In that case, the developers were able to find a fine-grained counter that, while useless for timekeeping, can be used to add a bit of entropy to the random pool. A new interface has been added to allow architecture code to provide access to such counters. But the best solution, he said, was for vendors to put hardware random number generators on their chips. 

Josh Triplett presented a proposal to get rid of the various "defconfig" files found in the kernel tree. These files are supposed to contain a complete, bootable configuration for a given architecture. He would like to move that information into the Kconfig that define the configuration options themselves. There would be a new syntax saying whether a given option should be enabled by default whenever the default system config was requested by the user. 

Linus didn't like the idea, though, saying that it would clutter the Kconfig files and still not solve the problem. He also noted that most defconfig files are nearly useless; the x86 one, he said, is essentially a random configuration used by Andi Kleen several years ago. A lot of the relevant configuration settings are architecture-dependent, so it would be necessary to add architecture-specific selectors and such. 

The plan at this point is to move further discussion to the mailing list, but, without some changes, this idea probably will not get too far. 

Peter Senna talked briefly about the Coccinelle semantic analysis tool which, he said, is finding a few bugs in each kernel development cycle. He would like to add more test cases to the system; interested developers are directed toward [coccinellery.org](<http://coccinellery.org/>) for examples of how to use this tool. (One could also see [this LWN article](<https://lwn.net/Articles/315686/>) for an introduction to Coccinelle). Dan Carpenter talked briefly about his smatch tool, which is also improving over time. His biggest goal at this point is to provide more user-friendly output; the warnings that come out of smatch now can be rather difficult to interpret. 

The final talk was presented by Paul Walmsley; it covered automatic testing of ARM kernels. He is running a testing lab that builds and boots a number of trees, generating reports when things go wrong. Olof Johansson run an elaborate testing setup; among other things, it performs fuzz testing with Trinity. There is also a 20-board testing array run by Kevin Hilman; he is doing power consumption tests as well. 

These testing rigs, Paul said, are catching a lot of bugs, often before the relevant patches get very far. There are also a lot of work to keep going, though. Part of that problem may be related to the fact that the bisection of problems must be done manually; work is being done to automate that process as soon as possible. 

After that there was a brief discussion of the Kernel Summit itself; some developers complained about communications, saying that they didn't always know about everything that was going on. There was also some discussion of the Linux Foundation Technical Advisory Board election held the night before, which was a somewhat noisy and chaotic affair. Thereafter, the group picture was taken and the developers headed out in search of dinner and beer.
