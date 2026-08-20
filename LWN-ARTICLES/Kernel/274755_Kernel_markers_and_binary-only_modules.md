---
title: "Kernel markers and binary-only modules"
url: https://lwn.net/Articles/274755/
date: "March 24, 2008"
category: "Copyright issues; Kernel markers; Modules-Debugging"
author: "By Jonathan Corbet March 24, 2008"
---

By **Jonathan Corbet**  
March 24, 2008

[Kernel markers](<http://lwn.net/Articles/245671/>) are a mechanism which allows developers to put static tracepoints into the kernel. Once placed, these markers can be used by operations staff to trace well-known events in running systems without that staff having to know about kernel code. Solaris provides a long list of static tracepoints for use with Dtrace, but Linux, thus far, has none. That situation should eventually change - static markers were only merged into the mainline in 2.6.24. But, as the developers start to look more seriously at markers, some interesting issues are coming up. 

One of those emerged as a result of [this patch](<https://lwn.net/Articles/274756/>) from Mathieu Desnoyers which allows proprietary modules to contain markers. The fact that current kernels do not recognize markers in binary-only modules is mostly an accident: markers are disabled in modules with any sort of taint flag set as a way to prevent kernel crashes - a kernel oops being a rather heavier-weight marker than most people wish to encounter. Matthieu tightened that test in a way that allows markers in proprietary modules, saying ""let's see how people react"". Needless to say, he saw. 

One might well wonder why the kernel developers, not known for their sympathy toward proprietary modules in general, would want to consider letting those modules include static tracepoints. The core argument here is that static markers allow proprietary modules to export a bit more internal information to the kernel, and to their users. It is seen as a sort of (very) small opening up on the part of the proprietary module writer. Mathieu [says](<https://lwn.net/Articles/274762/>): 

I think it's only useful for the end user to let proprietary modules open up a bit, considering that proprietary module writers can use the markers as they want in-house, but would have to leave them disabled on shipped kernels. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

The idea is that, by placing these tracepoints, module authors can help others learn more about what's going on inside the module and help people track down problems. The result should be a more stable kernel which - whether proprietary modules have been loaded or not - is generally considered to be a good thing. 

On the other hand, there's no shortage of developers who are opposed to extending any sort of helping hand to binary module authors. Giving those modules more access to Linux kernel internals, it is argued, only leads to trouble. Ingo Molnar [put it this way](<https://lwn.net/Articles/274760/>): 

Why are we even arguing about this? Binary modules should be as isolated as possible - it's a totally untrusted entity and history has shown it again and again that the less infrastructure coupling we have to them, the better. 

Ingo also worries that allowing binary modules to use markers will serve to make the marker API that much harder to change in the future. Since that API is quite young, chances are good that changes will happen. As much as the kernel developers profess not to care about binary-only modules, the fact of the matter is that there are good reasons to avoid breaking those modules. The testing community certainly gets smaller when testers cannot load the modules they need to make their systems work in the manner to which they have become accustomed. So it is possible that allowing proprietary modules to use markers could make the marker API harder to fix in future kernel releases. 

The grumbles have been loud enough that Matthieu's patch will probably not be merged for 2.6.25. The idea is likely to come back again, but not necessarily right away: the marker feature may have been merged in 2.6.24, but it would appear that 2.6.25 will be released with no actual markers defined in the source. It's not clear that binary-only module authors are pushing to add tracepoints when none of the other developers are doing so. Until somebody starts actually using static markers, debates on _where_ they can be used will continue to be of an academic nature.
