---
title: The VMI virtualization interface
url: https://lwn.net/Articles/175706/
date: "March 15, 2006"
category: Virtualization
author: 
---

Nobody could ever claim that there is a shortage of Linux virtualization technologies to choose from. There are numerous approaches, from lightweight "container" techniques which simply create walls between parts of the system, to full virtualization approaches which implement a complete virtual hardware platform capable of running a number of (unmodified) operating systems. Between the two are "paravirtualization" approaches which require a certain amount of awareness in the guest kernel. To many, paravirtualization seems like the best approach, in that it promises to combine a relatively high level of performance with strong isolation of guest systems. Xen is currently the highest-profile paravirtualization system out there, but there are others. 

Each paravirtualization approach places its own demands on the guest system. Before a particular system can run under a given hypervisor, it must be modified to work with that hypervisor's interface. This requirement can add to the work of creating a virtual system in the first place, and it increases the maintenance burden going forward, especially if both the hypervisor and the guest kernel are under heavy development. 

In an attempt to make life easier for virtualization hackers, Zachary Amsden (of VMware) has put forward [a complex proposal](<http://lwn.net/Articles/175457/>) for a common virtual machine interface (VMI) layer with some interesting properties. The VMI layer defines a set of calls for performing machine-specific functions - the sorts of things that generally require hypervisor intervention. These calls are very low-level - operations like changing page protections, enabling interrupts, writing model-specific registers, changing specific control registers, dealing with timer events, etc. As a result, the VMI interface currently only works with i386-architecture systems, though an x86-64 port is in the works. 

When a virtualized kernel boots, one of the first things it does is search for a "VMI ROM" provided by the hypervisor. That ROM provides the information needed for the low-level VMI calls to interact with the hypervisor. Using information found in the ROM, the just-booted kernel modifies its own code to use the hypervisor's functions without table lookups or indirect function calls. As a result, hypervisor operations are fast. 

There are a couple of interesting implications of this approach. One is that a VMI-equipped kernel can run under any VMI hypervisor without modification - or even recompilation. It simply grabs the ROM provided by whatever hypervisor is present and gets on with life. Just as interesting is the fact that such a kernel can run on the bare hardware with no hypervisor at all, as the host kernel. The VMI developers state that the performance impact of running with the VMI calls is essentially zero. That leads to this claim: 

VMI Linux has negligible overheads on native machines, so much so, that we are confident that VMI Linux can, in the long run, be the default Linux for i386. 

The actual code is packaged as a 24-part patch. It involves significant amounts of low-level tweaking and assembly language trickery. That may have something to do with why there have been few comments on the code itself. The discussion which has been seen seems somewhat favorable, if reserved. Among other things, there will need to be an open source hypervisor which uses this interface before it will be seriously considered for merging. In the mean time, anybody interested in the details can learn more from [the documentation file](<https://lwn.net/Articles/175715/>).
