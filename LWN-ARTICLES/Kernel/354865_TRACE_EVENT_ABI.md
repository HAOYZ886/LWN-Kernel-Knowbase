---
title: TRACE_EVENT_ABI
url: https://lwn.net/Articles/354865/
date: "September 30, 2009"
category: "Development tools-Kernel tracing; Tracing-ABI issues"
author: "By Jonathan Corbet September 30, 2009"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
September 30, 2009

Tracepoints are proving to be increasingly useful as system development and diagnostic tools. There is one question about tracepoints, though, which has not yet gotten a real answer: do tracepoints constitute a user-space ABI? If so, some serious constraints come into play. An ABI, once exposed, cannot be changed in a way which might break applications. Tracepoints, being tightly tied to the kernel code they instrument, are inherently hard to keep stable. If a tracepoint cannot be modified or removed, it will make modifications to the surrounding code harder. In the worst case, ABI-preservation requirements could block the incorporation of important kernel changes - an outcome which could quickly sour developers on the tracepoint idea as a whole. 

Arjan van de Ven's [`TRACE_EVENT_ABI` patch](<http://lwn.net/Articles/353880/>) is an attempt to bring some clarity to the situation. For now, it just defines a tracepoint in exactly the same way as `TRACE_EVENT`; the difference is that it is meant to create a tracepoint which can be relied upon as part of the kernel ABI. Such tracepoints should continue to exist in future kernel releases, and the format of the associated trace information will not change in application-breaking ways. What that means in practice is that no fields would be deleted, and any new fields would be added at the end. 

Whether this approach will work remains to be seen. The word from Linus in the past has been that kernel ABIs are created by applications which rely on an interface, rather than any specific marking on the interface itself. So if people start using applications which expect to be able to use a specific tracepoint, that tracepoint may be set in cement regardless of whether it was defined with TRACE_EVENT_ABI. This macro would thus be a good guide to the kernel developers' intent, but it can make no guarantee that only specially-marked tracepoints will be subject to ABI stability requirements.
