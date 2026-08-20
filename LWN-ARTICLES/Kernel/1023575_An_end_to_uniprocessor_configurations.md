---
title: An end to uniprocessor configurations
url: https://lwn.net/Articles/1023575/
date: "June 10, 2025"
category: Scheduler
author: "By Jonathan Corbet June 10, 2025"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
June 10, 2025

The Linux kernel famously scales from the smallest of systems to massive servers with thousands of CPUs. It was not always that way, though; the initial version of the kernel could only manage a single processor. That limitation was lifted, obviously, but single-processor machines have always been treated specially in the scheduler. That longstanding situation may soon come to an end, though, if [this patch series](<https://lwn.net/ml/all/20250528080924.2273858-1-mingo@kernel.org>) from Ingo Molnar makes it upstream. 

Initially, Linus Torvalds's goal with Linux was simply to get something working; he did not have much time to spare for hardware that he did not personally have. And he had no multiprocessor machine back then — almost nobody did. So, not only did the initial version of the kernel go out with no SMP support, the kernel lacked that support for some years. The 1.0 and 1.2 releases of the kernel, which came out in 1994 and 1995, respectively, only supported uniprocessor machines. 

The beginnings of SMP support found their way into the 1.3.31 development release in late 1995; the associated [documentation file](<https://elixir.bootlin.com/linux/1.3.31/source/Documentation/SMP.txt>) included the warning: ""This is experimental. Back up your disks first. Build only with gcc2.5.8"". It took some time for the SMP work to stabilize properly; the dreaded [big kernel lock](<https://lwn.net/Articles/281938/>), which ensured that only one CPU was running within the kernel at any time, wasn't even introduced until 1.3.54. But, by the time 2.0 was released in June 1996, Linux worked reasonably well on two-CPU systems, for some workloads, at least. 

At that time, though, SMP systems were still relatively rare; most people running Linux did not have one. The majority of Linux users running on uniprocessor systems had little patience for the idea that their systems might be made to run slower in order to support those expensive SMP machines that almost nobody had. The tension between support for users of "big iron" and everybody else ran strong in those days, and a two-CPU system was definitely considered to be big iron. 

As a result, the addition of SMP support was done under the condition that it not regress performance on uniprocessor systems. This is a theme that has been seen many times over the history of Linux kernel development. Perhaps most famously, the realtime preemption code was not allowed to slow down non-realtime systems; in the end, realtime preemption brought a lot of improvements for non-realtime systems as well. In the case of SMP, this rule was implemented with a lot of macro magic, `#ifdef` blocks, and similar techniques. 

It is now nearly 30 years after the initial introduction of SMP support into the Linux kernel, and all of that structure that enables the building of special kernels for uniprocessor systems remains, despite the fact that one would have to look hard to _find_ a uniprocessor machine. Machines with a single CPU are now the outlier case; in 2025, we all are big-iron users. Many of the uniprocessor systems that are in use (low-end virtual servers, for example) are likely to be running SMP kernels anyway. Maintaining a separate uniprocessor kernel is usually more trouble than it is worth, and few distributors package them anymore. 

As Molnar pointed out in his patch series, there are currently 175 separate `#ifdef` blocks in the scheduler code that depend on `CONFIG_SMP`. They add complexity to the scheduler, and the uniprocessor code often breaks because few developers test it. As he put it: ""It's rare to see a larger scheduler patch series that doesn't have some sort of build complication on !SMP"". It is not at all clear that these costs are justified at this point, given how little use there is of the uniprocessor configuration. 

So Molnar proposes that uniprocessor support be removed. The 43-part patch series starts with a set of cleanups designed to make the subsequent surgery easier, then proceeds to remove the uniprocessor versions of the code. Once it is complete, the SMP scheduler is used on all systems, though parts of it (such as load balancing) will never be executed on a machine with a single CPU. Once the work is done, nearly 1,000 lines of legacy code have been removed, and the scheduler is far less of a `#ifdef` maze than before. 

Switching to the SMP kernel will not be free on uniprocessor systems; all that care that was taken with the uniprocessor scheduler did have an effect on its performance. A scheduler benchmark run using the SMP-only kernel on a uniprocessor system showed a roughly 5% performance regression. There is also a 0.3% growth in the size of the kernel text (built with the `defconfig` x86 configuration) when uniprocessor support is removed. This is a cost that, once upon a time, would have been unacceptable but, in 2025, Molnar said, things have changed: 

> But at this point I think the burden of proof and the burden of work needs to be reversed: and anyone who cares about UP performance or size should present sensible patches to improve performance/size. 

He described the series as ""lightly tested"", which is not quite the standard one normally wants to see for an invasive scheduler patch; filling out that testing will surely be required before this change can be accepted. But, so far, there have been no objections to the change; there are no uniprocessor users showing up to advocate for keeping their special configuration — yet. Times truly have changed, to the point that it would be surprising if this reversal of priorities didn't make it into the kernel in the relatively near future.
