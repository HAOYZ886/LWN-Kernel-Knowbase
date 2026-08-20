---
title: The state of realtime Linux
url: https://lwn.net/Articles/392154/
date: "June 15, 2010"
category: "LinuxTag-2010; Realtime"
author: "By Jonathan Corbet June 15, 2010"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
June 15, 2010

Since 2005, the realtime preemption project has worked to provide deterministic response times in stock Linux kernels. Over that time, though, it has come to appear that there is no guaranteed latency with regard to when all of this code will actually be merged. At LinuxTag 2010, realtime hacker Thomas Gleixner talked about the state of this patch set, what's coming, and, yes, when it might actually be merged in its entirety. Don't hold your breath. 

In truth, the realtime preemption code has been going into the mainline, piece by piece, for years. Some recently-merged pieces include [threaded interrupt handlers](<http://lwn.net/Articles/302043/>) and the sleeping spinlock precursor patches. The threaded handlers make a [![](https://static.lwn.net/images/conf/2010/LinuxTag/ThomasGleixner-sm.jpg)](<https://lwn.net/Articles/392166/>) number of driver tasks simpler (regardless of any realtime needs) by eliminating much of the need for tasklets and workqueues. They have also proved to be useful in providing support for some strange i2c-attached interrupt controller hardware. The spinlock changes do not affect the generated code (in mainline kernels), but they are useful for annotating the type of each lock. 

Recent movements of code into the mainline notwithstanding, the realtime patchset isn't getting any smaller. It seems that the realtime developers have an interesting problem: the realtime kernel is a really good place to try out a wide variety of new features. So, despite the fact that code occasionally moves to the mainline, new stuff keeps getting added to the realtime tree. 

This tree's attractiveness for the testing of new code comes from the fact that it tends to reveal scalability problems much more quickly than mainline kernels do. The extra preemptibility offered by this kernel comes at a cost: the price for lock contention is much higher. So the realtime tree shows scalability issues at lower levels of contention than non-realtime kernels. The important point is that the scalability bottlenecks encountered by realtime kernels are not unique to realtime; they just come sooner than the same bottlenecks will show up with the mainline. So realtime kernels can be used to look forward to the problems that the mainline kernel will be experiencing next year. 

Thus, for example, realtime kernels exhibit scalability problems in the virtual filesystem layer that are otherwise only seen in big-iron torture-test labs. That makes them useful for testing features, and especially useful for testing scalability improvements. That is why code like the [VFS scalability patch set](<http://lwn.net/Articles/360199/>) currently makes its home in that tree. Eventually, most of these pieces will get merged into the mainline. Thomas says that it will all be in by the end of the year - but _which_ year is not something he is willing to commit to. 

The next patch set to move to the mainline might be Peter Zijlstra's [memory management preemptibility series](<http://lwn.net/Articles/391176/>), which solves some long latencies in the memory management code; the current plan is to push these patches for 2.6.36. Another bit of code which might make the move is an option to force all drivers to use threaded interrupt handlers regardless of whether they explicitly request them. This option would almost certainly not be turned on for most production kernels, but it makes the testing of drivers with involuntarily threaded handlers easier. 

The realtime tree also suffers from a few unsolved problems. One of them is latencies in the slab allocator, which runs with preemption disabled for long periods of time. The [SLQB allocator](<http://lwn.net/Articles/311502/>) had raised hopes for a while, but it appears that it will not be pushed for merging anytime soon. So the realtime hackers have to find a way to fix one of the existing allocators, or give up and write a slab allocator of their own. Thomas noted that there are still a few letters left in the SL?B namespace, so there might just be an SLRB in the future. That is all quite vague at this point, though; Thomas admitted that he has no idea how this problem will be resolved. 

Another ongoing problem is the increasing use of per-CPU data. In throughput-oriented environments, per-CPU data increases scalability by eliminating contention between processors. But use of per-CPU data necessarily requires that preemption be disabled while the data is being manipulated; to do otherwise is to risk that the process working with that data will be preempted or moved to another processor, making a mess of things. Disabling preemption is anathema in an environment where everything is always supposed to be preemptable, though. So the realtime patch set currently puts a lock around per-CPU data accesses, eliminating the preemption problem but wrecking scalability. Here, too, a real solution has not yet been found. 

Thomas finished with a bit of talk about testing of the realtime tree. Quite a bit of "enterprise-class" testing is done in the well-furnished labs at companies like IBM and Red Hat. At the embedded level, the [Open Source Automation Development Lab](<http://osadl.org/>) has a modest [testing lab](<http://www.osadl.org/Test-Rack.test-rack.0.html>) of its own. But there's another interesting source of testing: the Linux audio community has been enthusiastic in its use of the realtime kernel and has helped find a number of issues. There's also a growing set of tools maintained in the [rt-tests](<http://lwn.net/Articles/371478/>) collection. 

All told, the picture painted by Thomas was one of a healthy project, even if we still don't know when it will all get into the mainline. Even in the realtime world, there are things we simply have to wait for.
