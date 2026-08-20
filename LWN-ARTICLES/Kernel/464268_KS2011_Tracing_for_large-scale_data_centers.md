---
title: "KS2011: Tracing for large-scale data centers"
url: https://lwn.net/Articles/464268/
date: "October 24, 2011"
category: Tracing
author: "By Jonathan Corbet October 24, 2011 2011 Kernel Summit coverage"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
October 24, 2011

* * *

[2011 Kernel Summit](<https://lwn.net/Articles/KernelSummit2011/>) coverage 

Google's David Sharp made an appearance at the 2011 Kernel Summit to talk about his employer's tracing needs. Linux generally works well for Google, but there will always be cases where things go wrong or take too long. As a part of its effort to track down the sources of the "last 0.1%" of performance problems, Google makes heavy use of the kernel's tracing features. 

Traced kernel events go through the kernel's ring buffer on their way to user space. For Google's purposes, the more data that can be fit into that buffer, the better. There is only so much memory that can be dedicated to that task; it needs to be able to hold data for as long a period as [![\[David Sharp\]](https://static.lwn.net/images/conf/2011/lc-europe/DavidSharp-sm.jpg)](<https://lwn.net/Articles/464311/>) possible. That memory cannot be increased, and the frequency of traced events cannot be decreased; that means that the only way to increase the capacity of the ring buffer is to reduce the size of the recorded events. 

David put up a plot showing the results of the work they have done. With their configured buffer size, a mainline kernel can hold about 10s worth of data. Taking some unnecessary fields out of the event header - things like the preemption count and lock depth - increases it to 12s. A much bigger gain can be had by removing the arguments to system calls, leaving only the lowest 16 bits of the first argument; that increases the duration to 32s. Finally, if the process ID is also removed, the buffer can hold 45s worth of data. That makes it far more useful for Google's purposes. 

It would be nice to be able to make such changes without hacking the kernel, but messing with the event format will clearly cause problems with the applications that consume those events. There are other things that Google would like; one of those is the ability to dynamically size the ring buffer on each CPU. In normal operation, some CPUs will fill their buffers more quickly than others; moving memory from the slower CPUs to the faster ones would, again, increase the duration for the whole system. Some sort of always-on "flight recorder" mode of tracing would also be useful; the data could then be grabbed when something goes wrong. 

Google does not limit itself to tracing kernel events; there is infrastructure to trace user space and remote procedure calls as well. That leads to the thorny problem of stitching the separate trace streams together, especially when they involve multiple machines. Those machines will never have 100% synchronized clocks, so lining up the trace streams can be a challenge. 

There are patches for some of the changes that have been made. But Linus worried that most heavy tracing users have specialized tools. The kernel sometimes gets the tracing changes, but the tools don't show up; in the absence of those tools, Linus is a lot less interested in the patches. So, he asked of tracing users in general: please make your tools available to others. David said that Google is open to the idea, but actually getting the tools pushed out into an open source release is hard. 

[![\[Ingo Molnar\]](https://static.lwn.net/images/conf/2011/lc-europe/IngoMolnar-sm.jpg)](<https://lwn.net/Articles/464320/>) Ingo Molnar talked about how tracing events tend to expose kernel internals; that can lead to ABI constraints as tools come to depend on those internals. The solution, according to Ingo, is to move the tools closer to the source of the events; in this case, that means putting the tools into the kernel source tree. The tooling, he said, is even more important than the kernel side of things. Peter Zijlstra joined in to say that it is a shame that we allow tools outside of the kernel at all. That leads to ABI preservation needs, with the result that tracepoints are telling lies to user space in response. There is no value in the "lock depth" field contained in trace events; they measure the behavior of the big kernel lock, which was removed for 2.6.39. But that is how it goes, responded Ingo; if the tool is useful and the code is open we will not break it. 

Steve Rostedt said that these problems are the result of bad ABI design. We need, he said, a better ABI that is easier to support; "we need a platform." Ingo responded that it is still early in the game for tracing infrastructure; we will need another ten years to really get it right. We should, he said, figure out the tools first, only then will we know what the kernel needs to provide. We need tools that will increase tracing use by an order of magnitude; messing with the ABI now will not accomplish that. 

Linus added that it's not the ABI that matters, it's backward compatibility. Anybody who does not understand that, he said, does not belong in technology. People blow it all the time, breaking things that used to work; desktop environments were cited as an example. He said that there is no real point in talking about getting the ABI right because we will not achieve it. What we will do is avoid breaking the tools; in the end, what the tools actually use is the ABI. 

Ted Ts'o raised the issue of trying to make a distinction between tracepoints that are intended to be a part of the ABI and those that should be treated more like `printk()`. But Linus said that it was not going to work, that the effort needs to go elsewhere. He complained that people are still talking about [how PowerTop broke](<https://lwn.net/Articles/442113/>) in response to a tracepoint change; it would have been better, he said, to put the effort into fixing PowerTop. If that had been done, then the change could be applied in five years or so. Ingo added that we should not legislate an ABI for as-yet nonexistent tracing tools; we should, instead, let things develop together. Phasing out ABIs is relatively easy in the tracing arena because users naturally want current tools. 

Few hard conclusions were reached in this session; with luck, though, we will at least see some new tools out of Google before too long. 

Next: [Structured error logging](<https://lwn.net/Articles/464276/>)
