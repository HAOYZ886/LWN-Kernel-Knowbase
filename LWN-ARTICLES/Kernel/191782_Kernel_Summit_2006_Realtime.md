---
title: "Kernel Summit 2006: Realtime"
url: https://lwn.net/Articles/191782/
date: "July 19, 2006"
category: Realtime
author: 
---

[2006 Kernel Summit coverage](<https://lwn.net/Articles/KernelSummit2006/>) on LWN.net.   
---  
Ever since the current realtime patch set began to circulate, there have been questions about when (and if) it would be merged into the mainline. Ted Ts'o and Ingo Molnar's session on realtime support started with the note that, to a great extent, this merging has happened. Over the last year, features like robust futexes, priority inheritance, the generic IRQ layer, the core timekeeping rewrite, and the lock validator have all gone into the mainline. For the first time in history, somebody (Ingo) was able to propose the merging of priority inheritance without getting flamed to a crisp by Linus. 

As a result of all this merging, the out-of-tree realtime patch set has shrunk considerably in size. About 700KB of patches remain, however. The biggest chunks at this point are the high-resolution timers and sleeping spinlocks. 

A realtime kernel can now regularly achieve 20 µsec response times on a heavily loaded system. Anymore, when there are response time issues, the problem is likely to be hardware related. It is, in other words, possible to make a Linux kernel which can provide realtime response - something many people would not have believed even a short time ago. 

There is some cost to this, in terms of code complexity and system performance, but the performance hit is often less than people think. Stock kernels do not properly account for time spent handling interrupts, while realtime kernels (as a result of moving interrupt handlers into processes) do. So programs may look like they are using more CPU, but it is really just a matter of the kernel noticing the time lost to interrupts. 

With regard to merging, the hard work has been done. Some of the trickiest patches were the ones which cleaned up locking semantics, made the big kernel lock preemptible, etc. One challenging part which remains is identifying the few locks which must remain "raw" (non-preemptible) spinlocks in a realtime kernel. There is a relatively small number of those, but it is important to find them all and change them over to the proper lock type. 

One outstanding issue is recovery when a realtime task goes out of control. On a realtime kernel, a high-priority task can block out everything else, including interrupt handlers. Current techniques for handling this situation are limited to those involving the power button. It might be nice to add something like a soft lockup detector. Linus, however, thinks that this is not a problem which should concern people. If you truly want hard realtime, that is what you will get, and vulnerability to runaway high-priority processes is part of the deal. 

A nice feature for the future might be realtime disk and filesystem I/O. Essentially, a process which remains within a given bandwidth limit could perform I/O and still have realtime response behavior. 

The question was asked: why bother with sleeping locks? Making locks preemptible is seen by some as a way of papering over the real problem: long lock hold times. Why not simply fix those? The answer comes in a couple of parts: 

  * Extensive efforts have been expended toward fixing lock problems for many years, and those efforts will continue into the future. The use of sleeping locks is not being used as an excuse to avoid fixing code which holds locks for too long. 

  * Ensuring realtime response in the absence of preemptible locks requires auditing the entire body of kernel source - all eight million lines or so. That's a big job, and one which is hard to keep up with in an environment where nearly ten thousand lines of code are being changed every day. Sleeping locks reduce the audit requirements to a couple thousand lines - a much more tractable problem. For those who need realtime response, guaranteed, that is a good deal. 

Linus indicated that (unlike some others) he has no problem with merging the sleeping locks patch, as long as there is no danger that sleeping locks might be configured in by default. That seems unlikely, given that distributors do not even turn on the much milder kernel preemption feature which has been present throughout the 2.6 kernel series. Since the realtime features ruin performance on large SMP systems, they are not likely to be turned on in enterprise kernels either. Even so, there is said to be pressure on distributors to make realtime kernels available. That pressure becomes harder to resist once a feature makes it into the mainline. It appears that this work may well be merged, however, so distributors may have to show a little backbone. 

  * [Next: Embedded systems](<http://lwn.net/Articles/191822/>).
