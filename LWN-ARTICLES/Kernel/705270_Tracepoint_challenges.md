---
title: Tracepoint challenges
url: https://lwn.net/Articles/705270/
date: "November 2, 2016"
category: "Tracing-ABI issues"
author: "By Jonathan Corbet November 2, 2016 2016 Kernel Summit"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
November 2, 2016

* * *

[2016 Kernel Summit](<https://lwn.net/Articles/KernelSummit2016/>)

The final core-day session at the 2016 Kernel Summit, run by Steve "Batman" Rostedt and Shuah Khan, concerned the use of tracepoints in the kernel. It started with a discussion of tracepoint performance issues, but quickly came around to the perennial area of concern about tracepoints: whether they form part of the kernel's user-space ABI or not. 

Steve started by noting that he is seeing an "explosion" in the number of tracepoints being added. The problem is that, while the cost of tracepoints has been made as low as possible, they are still not free. Each tracepoint hurts performance slightly. So it may eventually become necessary to limit the addition of tracepoints into the kernel. 

David Howells noted that a number of maintainers have been seen to push back on the addition of `printk()` calls to the kernel, saying that tracepoints should be used instead. Steve responded that they should push back on tracepoints too. Each tracepoint should have its own rationale [![\[Steve
Rostedt and Shuah Khan\]](https://static.lwn.net/images/conf/2016/ks/SteveShuah-sm.jpg)](<https://lwn.net/Articles/705279/>) justifying its existence. Chris Mason suggested that the best way to cut down on tracepoints is to require developers to document them. 

Mel Gorman reminded the group that tracepoints can be inserted dynamically into a running kernel. Mark Brown said that dynamic tracepoints require more tooling; that may be fine for a server system, but is harder on a phone. But Steve said that no special tools are required to insert tracepoints; it can all be done with `echo` commands. 

Shuah brought things around to the ABI issue by saying that tracepoints can be highly effective for debugging problems on deployed systems. But, she asked, if we add tracepoints, do we have to maintain them forever? Ted Ts'o noted that the current work with eBPF makes tracepoints far easier to use, a change with both good and bad aspects. On the good side, the kernel now has dynamic tracing capabilities approaching those of DTrace. On the other hand, that means that people are starting to use these capabilities, and system administrators are starting to depend on them. So the ABI issue is no longer theoretical. 

Peter Zijlstra said that there are tracepoints in the scheduler now that he would like to remove, but fears he can't without breaking things. Linus, though, said that problematic tracepoints should simply be taken out, especially if they are hindering development. This should happen even if the removal would break the LatencyTOP tool. Greg Kroah-Hartman protested that, in the past, Linus had [blocked a tracepoint change](<https://lwn.net/Articles/442113/>) that broke the [PowerTOP](<https://01.org/powertop>) utility. Linus's answer is that the community was still figuring out how to work with tracepoints then, and that there was no actual need to break PowerTOP at that time. 

But, he said, tracepoints are still a view into the kernel's internals. They have to be able to change over time. If the removal of a particular tracepoint proves to be painful for user space, that removal will have to be reconsidered, but only then. That, he said, has always been the ABI rule: we can change things, but, if the result is broken user space, we'll change it back. Additionally, he said, LatencyTOP users tend to be people who compile their kernels anyway, while PowerTOP users are not. So LatencyTOP users can better adjust to a tracepoint change. 

And, in the end, Linus said, if a tracepoint becomes so useful that it becomes part of the ABI, there is probably a good reason for it and it likely should be kept. But the way to find out is to change things and see who screams. 

Ted suggested that now would be a good time to look at Brendan Gregg's [perf-tools](<https://github.com/brendangregg/perf-tools>) set to see which tracepoints it depends on. If those tracepoints need adjustment to be supportable in the long run, now is the time to make those changes before the usage of those tools increases further. 

Some maintainers may feel better now about allowing tracepoints in the code they are responsible for, but others have not changed their view. Al Viro made it clear that his policy would not be changing, and that he would not be allowing any tracepoints in the virtual filesystem layer. He is worried about how some developers may use those tracepoints, and does not want to see a day in the future where systems are unable to boot with newer kernels as the result of tracepoint changes. 

The session concluded with Linus saying that, in the history of kernel development, nobody has ever screamed about a change to a tracepoint. He allowed that this might happen as the use of tracepoints increases. But, he said, there is no point in making a big deal about that possibility before it proves to be a problem.
