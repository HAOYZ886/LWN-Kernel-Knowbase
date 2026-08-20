---
title: Error handling for I/O memory management units
url: https://lwn.net/Articles/608914/
date: "August 20, 2014"
category: IOMMU
author: "By Jonathan Corbet August 20, 2014 Kernel Summit 2014"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
August 20, 2014

* * *

[Kernel Summit 2014](<https://lwn.net/Articles/KernelSummit2014/>)

Traditionally the Kernel Summit focuses more on process-oriented issues than technical problems; the latter are usually deemed to be better addressed by smaller groups on the mailing lists. The 2014 Summit, however, started off with a purely technical topic: how to handle errors signaled by I/O memory management units (IOMMUs)? 

An IOMMU performs translations between memory addresses as seen by devices and those seen by the CPU. It can be used to present a simplified address space to peripherals, to make a physically scattered buffer look contiguous, or to restrict device access to a limited range of memory. Not all systems have IOMMUs in them, but there is a slow trend toward including them in a wider range of systems. 

David Woodhouse started off by saying that, in the IOMMU context, there is no generalized way of reporting errors. So drivers cannot easily be notified if something goes wrong with an IOMMU. What _does_ exist is a certain amount of subsystem-specific infrastructure. The PowerPC architecture has its "extended error handling" (EEH) structure that "only [![\[David Woodhouse\]](https://static.lwn.net/images/conf/2014/ks/DavidWoodhouse-sm.jpg)](<https://lwn.net/Articles/608916/>) Ben Herrenschmidt understands," and the PCI subsystem has an error-handling mechanism as well. But the kernel needs a coherent way to report errors from an IOMMU to drivers regardless of how they are connected to the system. There also needs to be a generalized mechanism by which an offending device can be shut down to avoid crippling the system with interrupt storms. 

David presented a possible way forward, which was based on extending and generalizing the PCI error-handling infrastructure. It would need to be moved beyond PCI and given extra capabilities, such as the ability to provide information on exactly what went wrong and what the offending address was. He asked: does anybody have any strong opinions on the subject? This is the Kernel Summit, so, of course, opinions were on offer. 

Ben started by saying that it is not always easy to get specific information about a fault to report. The response to errors — often wired into the hardware — is to isolate the entire group of devices behind a faulting IOMMU; at that point, there is no access to any information to pass on. Drivers do have the ability to ask for fault notification, and they can attempt to straighten out a confused device. In the absence of driver support, he said, the default response is to simulate unplug and replug events on the device. 

David pointed out that with some devices, graphics adapters in particular, users do not want the device to stop even in the presence of an error. One command stream may fault and be stopped, but others running in parallel should be able to continue. So a more subtle response is often necessary. 

Josh Triplett asked about what the response should be, in general, when something goes wrong. Should recovery paths do anything other than give up and reset the device fully? For most devices, a full reset seems like an entirely adequate response, though, as mentioned, graphics adapters are different. Networking devices, too, might need a gentler hand if error recovery is to be minimally disruptive. But David insisted that, almost all the time, full isolation and resetting of the device is good enough. The device, he said, "has misbehaved and is sitting on the naughty step." 

Andi Kleen asked how this kind of error-handling code could be tested. In the absence of comprehensive testing, the code will almost certainly be broken. David responded that it's usually easy to make a device attempt DMA to a bad address, and faults can be injected as well. But Ben added that, even with these tools, EEH error handling breaks frequently anyway. 

David asked: what about ARM? Will Deacon responded that there are no real standards outside of PCI; he has not seen anything in the ARM space that responds sanely to errors. He also pointed out that hypervisors can complicate the situation. An IOMMU can be used to provide limited DMA access to a guest, but that means exposing the guest to potential IOMMU errors. A guest may end up isolating an offending device, confusing the host which may not be expecting that. 

Arnd Bergmann asked that any error-handling solution not be specific to PCI devices since, in the ARM world, there often is not a PCI bus involved. David suggested that the existing PCI error-handling infrastructure could be a good place to start if it could be made more generic. There are some PCI-specific concepts there that would have to be preserved (for PCI devices) somehow, but much of it could be moved up into `struct device` and generalized. After hearing no objections to that approach, David said he would go off and implement it. He will not, he noted dryly, be interested in hearing objections later on. 

**Next** : [Stable tree maintenance](<https://lwn.net/Articles/608917/>).
