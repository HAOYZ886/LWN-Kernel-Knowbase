---
title: Netgpu and the hazards of proprietary kernel modules
url: https://lwn.net/Articles/827596/
date: "July 31, 2020"
category: "Copyright issues; Modules-Exported symbols"
author: "By Jonathan Corbet July 31, 2020"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
July 31, 2020

On its face, the [netgpu patch set](<https://lwn.net/ml/netdev/20200727224444.2987641-1-jonathan.lemon@gmail.com/>) appears to add a useful feature: the ability to copy network data directly between a network adapter and a GPU without moving it through the host CPU. This patch set has quickly become an example of how not to get work into the kernel, though; it has no chance of being merged in anything like its current form and has created a backlash designed to keep modules like it from ever working in mainline kernels. It all comes down to one fundamental mistake: basing kernel work on a proprietary kernel module. 

The use case for netgpu appears to be machine-learning applications that consume large amounts of data. The processing of this data is offloaded to a GPU for performance reasons. That GPU must be fed a stream of data, though, that comes from elsewhere on the network; this data follows the usual path of first being read into main memory, then written out to the GPU. The extra copy hurts, as does the memory-bus traffic and the CPU time needed to manage this data movement. 

This overhead could be significantly reduced if the network adapter were to write the data directly into the GPU's memory, which is accessible via the PCI bus. A suitably capable network adapter could place packet data in GPU memory while writing packet headers to normal host memory; that allows the kernel's network stack to do the protocol processing as usual. The netgpu patch exists to support this mode of operation, seemingly yielding improved performance at the cost of losing some functionality; anything that requires looking at the packet payload is going to be hard to support if that data is routed directly to GPU memory. 

A lot of work has been done in recent years to enable just this kind of zero-copy, device-to-device data transfer, so one might expect this functionality to be well received. And, indeed, the code was reviewed normally until [the last patch](<https://lwn.net/ml/netdev/20200727224444.2987641-22-jonathan.lemon@gmail.com/>) in this 21-part series, where things ran into a snag. This is the patch that interfaces between the netgpu module and the proprietary NVIDIA GPU driver; it can't even be built without the NVIDIA driver's files on disk. On seeing this, Greg Kroah-Hartman [stopped and complained](<https://lwn.net/ml/netdev/20200728163100.GD4181352@kroah.com/>): 

Ok, now you are just trolling us. 

Nice job, I shouldn't have read the previous patches. 

Please, go get a lawyer to sign-off on this patch, with their corporate email address on it. That's the only way we could possibly consider something like this. 

What followed was an occasionally harsh and acrimonious discussion on whether the patches should have ever been posted in the first place. Jonathan Lemon, the author of the patches, [insisted](<https://lwn.net/ml/netdev/20200728014812.izihmnon3khzyr32@bsd-mbp.dhcp.thefacebook.com/>) that they were not about providing functionality to the proprietary NVIDIA module in particular: 

This is not in support of a proprietary driver. As the cover letter notes, this is for data transfers between the NIC/GPU, while still utilizing the kernel protocol stack and leaving the application in control. 

While the current GPU utilized is nvidia, there's nothing in the rest of the patches specific to Nvidia - an Intel or AMD GPU interface could be equally workable. 

Others disagreed, though, stating that the code was clearly designed around the NVIDIA module from the beginning. Christoph Hellwig [argued](<https://lwn.net/ml/netdev/20200728172703.GA5667@infradead.org/>) that any upstream-oriented driver should be based on the existing [P2PDMA](<https://lwn.net/Articles/767281/>) framework, which exists just to support device-to-device data transfers. Jason Gunthorpe [agreed](<https://lwn.net/ml/netdev/20200728181904.GA138520@nvidia.com/>), and [argued](<https://lwn.net/ml/netdev/20200728233806.GC16789@nvidia.com/>) that the design of the module as a whole was driven by NVIDIA's choices: 

The design copied the nv_p2p api design directly into struct netgpu_functions and then aligned the rest of the parts to use it too. Yes, other GPU drivers could also be squeezed into this API, but if you'd never looked at the NVIDIA driver you'd never pick such a design. It is inherently disconnected from the [memory-management subsystem]. 

By the time the discussion wound down, it was clear that this patch set wasn't going anywhere in its current form. Large amounts of work will have to be done to build it on top of the existing kernel mechanisms for cross-device data movement — P2PDMA, the [DMA-buf subsystem](<https://www.kernel.org/doc/html/v5.6/driver-api/dma-buf.html>), etc. This work may have achieved its initial goal, but it clearly went far down the wrong path when it comes to being merged into the mainline kernel. 

The sad part is that, by all appearances, the goal of this work was not to add functionality for NVIDIA GPUs in particular. Lemon does not seem to be an NVIDIA employee; the patches included a Facebook email address. But NVIDIA, with its proprietary module, was what was at hand, so that is the device that the patch set was designed to work with. Designing the module to work with free GPU drivers from the outset would have driven a number of decisions in different directions and avoided much of the trouble that has ensued. 

Meanwhile, in an attempt to make this sort of mistake harder to make (and, surely only by coincidence, make life a bit harder for proprietary modules in general), Hellwig has posted [a patch set](<https://lwn.net/ml/linux-kernel/20200730061027.29472-1-hch@lst.de/>) changing the way GPL-only symbols are handled. Symbols exported as GPL-only by the kernel are made unavailable to proprietary modules, but there has always been a bit of a loophole in how this is enforced. A proprietary module can be broken into two parts, one of which is a minimal shim layer that interfaces between the kernel and the proprietary code. If the shim module is GPL-licensed, it can access GPL-only symbols, which it can then make available to the proprietary module. 

Hellwig's patch does not entirely close this loophole, but it makes exploiting it a bit harder. With this patch applied, any module that imports symbols from a proprietary module is itself marked as being proprietary, denying it access to GPL-only symbols. If the shim module has already accessed GPL-only symbols by the time it gets around to importing symbols from the proprietary module, that import will not be allowed. This new restriction would keep the netgpu-NVIDIA interface module from loading, impeding the development of such modules in the future. It still does not cover the case, though, of a shim module that exports its own symbols to a proprietary module without importing anything from that module. 

For as long as the kernel has had a module loader, there have been debates over proprietary modules. In 2006, the development community [seriously discussed](<https://lwn.net/Articles/215075/>) banning them entirely. Two years later, a long list of kernel developers signed [a position statement](<https://lwn.net/Articles/287056/>) stating that proprietary modules are ""detrimental to Linux users, businesses, and the greater Linux ecosystem"". These modules continue to exist, though, and do not appear to be going away anytime soon. This episode, where a proprietary module helped send a significant development project in the wrong direction and makes it nearly impossible to implement this functionality in a way that works with all GPUs, demonstrates one of the reasons why the development community sees those modules as being harmful.
