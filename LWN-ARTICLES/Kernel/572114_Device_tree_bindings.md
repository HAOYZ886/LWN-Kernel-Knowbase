---
title: Device tree bindings
url: https://lwn.net/Articles/572114/
date: "October 29, 2013"
category: Device tree
author: "By Jonathan Corbet October 29, 2013 2013 Kernel Summit"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
October 29, 2013

* * *

[2013 Kernel Summit](<https://lwn.net/Articles/KernelSummit2013/>)

The device tree issue was omnipresent during the 2013 Kernel Summit, with dedicated minisummit sessions, hallway discussions, and an interminable mailing list thread all in the mix. Despite all the noise, though, some progress was seemingly made on the issue of how to evolve device tree bindings without breaking systems that depend on them. A group of developers presented those results to the plenary session. 

Grant Likely and David Woodhouse started by reiterating the problems that led to the adoption of device trees for the ARM architecture in the first place. It comes down to undiscoverable hardware — hardware that does not describe itself to the CPU, and which thus cannot be enumerated automatically. This hardware is [![\[David Woodhouse\]](https://static.lwn.net/images/conf/2013/lce-ks/DavidWoodhouse-sm.jpg)](<https://lwn.net/Articles/572138/>) not just a problem with embedded ARM systems, Grant said; it is showing up in desktop and server systems too. In many situations, we are seeing the need for a general hardware description mechanism. The problem is coming up with the best way of doing this description while supporting systems that were shipped with intermediate device tree versions. 

The solution comes down to a set of process changes, starting with a statement that device tree bindings are, indeed, considered to be stable by default. Once a binding has been included in a kernel release, developers should not break systems that are using that binding. That said, developers should not get hung up on creating perfect bindings now; we still do not know all of the common patterns and will need to make changes as we learn things. That means that bindings can, in fact, change after they have been released in a kernel; the key is to make those changes in the correct way. 

Another decision that has been made is that configuration data is allowed within device tree bindings. This has been a controversial area; many developers feel that device trees should describe the hardware and nothing else. Grant made the claim that much configuration data should be considered part of the hardware design; there may be a region of memory intended for use as graphics buffers, for example. 

There will be a staging-like mechanism for unstable bindings, but it expected that this mechanism will almost never be used. The device tree developers will be producing a document describing the recommended best practices and processes around device trees; there will also be a set of validation tools. Much of this work, it is hoped, will be completed within the next year. 

The current rule that device tree bindings must be documented will be reinforced. The documentation lives in `Documentation/devicetree/bindings` in the kernel tree. The device tree maintainers would prefer to see these documents posted as a separate patch within a series so they can find it quickly. Bindings should get an acknowledgment from the device tree maintainers, but there is already too much review work to be done in this area. So, if the device tree maintainers are slow in getting to a patch, subsystem maintainers are considered empowered to merge bindings without an ack. These changes should go through the usual subsystem tree. 

The compatibility rules say that new kernels must work with older device trees. If changes are required, they should be put into new properties; the old ones can then be deprecated but not removed. New properties should be optional, so that device trees lacking those properties continue to work. The device tree developers will provide a set of guidelines for the creation of future-proof bindings. 

If it becomes absolutely necessary to introduce an incompatible change, Grant said, the first step is that the developer must submit to the mandatory public flogging. After that, if need be, developers should come up with a new "compatible" string and start over, while, of course, still binding against the older string if that is all that is available. DTS files (which hold a complete device tree for a specific system) should contain either the new or the old compatible string, but never both. 

If all else fails, it is still permissible to add quirks in the code for specific hardware. If this is done with care, it should not reproduce the old board file problem; such quirks should be relatively rare. 

[![\[Ben Herrenschmidt\]](https://static.lwn.net/images/conf/2013/lce-ks/BenHerrenschmidt-sm.jpg)](<https://lwn.net/Articles/572139/>) Ben Herrenschmidt worried about the unstable binding mechanism; it is inevitable, he thought, that manufacturers would ship hardware using unstable bindings. David replied that bad manufacturer behavior is not limited to bindings; they ship a lot of strange code as well. But, he said, manufacturers have learned over time that things go a lot easier if they work with upstream-supported code. He didn't think that the unstable binding mechanism would ever be used; it is a "political compromise" that should never need to be employed. Arnd Bergmann added that, should this ever happen, it will not be the end of the world; the kernel community just has to make the consequences of shipping unstable bindings clear. In such cases, users will just have to update the device tree in their hardware before they can install a newer kernel. 

What about the reviewer bandwidth problem? The main change in this area, it seems, is that the device tree reviewers will _only_ look at the binding documentation; they will not look at the driver code itself. That is part of why they want the documentation in a separate patch. That means that subsystem maintainers will have to be a bit more involved in ensuring that the code matches the documentation — though there will be some tools that will help in that area as well. 

[Next: [Checkpoint/restart in user space](<https://lwn.net/Articles/572125/>)]
