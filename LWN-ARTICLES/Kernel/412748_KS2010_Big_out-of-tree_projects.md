---
title: "KS2010: Big out-of-tree projects"
url: https://lwn.net/Articles/412748/
date: "November 2, 2010"
category: Development model
author: "By Jonathan Corbet November 2, 2010 2010 Kernel Summit"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
November 2, 2010

* * *

[2010 Kernel Summit](<https://lwn.net/Articles/KernelSummit2010/>)

For almost as long as the kernel has existed, there have been large kernel projects which live outside of the mainline. At the 2010 Kernel Summit, Roland Dreier and Chris Mason led a session to discuss these projects and what, if anything, should be done about them. 

Roland started off by talking about the OpenFabrics Enterprise Distribution (OFED), which is [described](<http://www.openfabrics.org/>) as: 

...a downloadable, networking open-source software package that implements Remote Direct Memory Access (RDMA) and kernel bypass mechanisms to deliver high-efficiency computing, wire-speed messaging, ultra-low microsecond latencies and fast I/O for servers, block storage and file systems. 

In reality, it's a large chunk of out-of-tree code distributed by the OpenFabrics alliance. There are a number of enhancements to the InfiniBand infrastructure, backports to older kernels, and so on. Hardware companies have been telling their customers to use the OFED stack instead of mainline; the enterprise distributions have been shipping it. 

So why does OFED exist? Roland says it's partly his fault; he has not been merging stuff into the mainline quickly enough. He has had trouble getting people to help review all of that code. But, he says, there is also an incentive for OFED: it gives the OpenFabrics Alliance an ongoing reason to exist. Perhaps OFED should go into the staging tree? The OFED tree consists mainly of patches to code which is already in the mainline; it's not a set of standalone drivers which can sensibly be put into the staging tree. 

It was suggested (nobody was able to confirm it) that Red Hat will not be shipping updated OFED stacks in the future. Chris said that the distributors need to push back more against this kind of out-of-tree code. Oracle, he noted, is doing that. 

Ted Ts'o asked about the scope of the problem - what other large, out-of-tree projects exist? The list started with Xen's Dom0 code. Jeremy Fitzhardinge took a moment to update the crowd on the status of Dom0; most of the trickiest bits have been merged for 2.6.37, so it's actually possible to boot a Dom0 kernel. There are no backend drivers, though, so that kernel will have no devices available; Jeremy allowed that this limitation reduces the usefulness of that kernel somewhat. Some of those drivers will be merged in forthcoming development cycles; others are so ugly that they may never get in. 

Other significant out-of-tree projects include the Lustre filesystem, the realtime preemption tree, OpenVZ, the LTTng trace kit, the grsecurity patches, and the always controversial SCSI target code. In general, it was agreed, we are doing much better than before. 

The process needs to continue, though. According to Chris, Oracle has found every vendor driver it has ever encountered to be horribly broken and unsupportable. So Oracle has taken a strong position that only upstream drivers can be supported. Ted said that, for enterprise systems, the problem has mostly been solved; the same is, alas, not true in the embedded area. So we will have to go through the same education process with embedded vendors. 

It may be the same problem, but out-of-tree code may be harder to eradicate in the embedded world. Vendors of embedded systems are incredibly price-sensitive; upstream drivers fall by the wayside when compared to a savings of a few cents on a component. What's needed is to make "upstream drivers" a stronger part of the procurement process; it must be a technical requirement for any candidate components. Tim Bird said that this problem had been discussed at a recently concluded embedded Linux summit; the result is a new effort to encourage a "preference" for open-source drivers. Many vendors apparently claim that they have never even heard that free drivers are preferable; the first step is to cure them of that impression. 

The session concluded with a statement that, while out-of-tree projects still exist, things have improved and are continuing to do so. 

[Next: Checkpoint/restart](<https://lwn.net/Articles/412749/>).
