---
title: "Threadable NAPI polling, softirqs, and proper fixes"
url: https://lwn.net/Articles/687617/
date: "May 18, 2016"
category: "Interrupts-Software; NAPI"
author: "By Jonathan Corbet May 18, 2016"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
May 18, 2016

Sometimes, when trying to make the kernel work better for specific workloads, one runs into problems originating in genuinely old code. Delving into such code can be a daunting task — it has worked for many years, and tweaking it could have surprising consequences in distant parts of the kernel. So it is not surprising that developers can be tempted to work around a problem in other ways. Such a situation recently came up with regard to high-rate networking on small systems; a look at the problem and its resolution provides a good excuse to dig into an ancient kernel mechanism: software interrupts or "softirqs". 

#### Software interrupts

Processing of hardware interrupts is considered to be one of the most urgent tasks for the kernel; when an interrupt is signaled, the system drops what it is doing and calls the interrupt handler. Often, an interrupt will tell the system that there is more work — such as processing a completed I/O operation — to be done, but that work should not be done in the interrupt handler itself. To handle such cases, the kernel provides a number of mechanisms for deferred execution; these include workqueues, tasklets, timers, and software interrupts. 

Of these mechanisms, software interrupts must be about the oldest; `kernel/softirq.c` starts with a copyright statement from Linus dated 1992 (though, in truth, that comment may be the only thing that survives from the [`kernel/softirq.c`](<https://lwn.net/Articles/687726/>) that appeared in the 1.2 release). It is also a mechanism that many developers will be relatively unfamiliar with; most will never interact directly with software interrupts. But they play an important role in how many urgent tasks in the kernel are handled. 

Essentially, a software interrupt is meant to look like a hardware interrupt, except that it runs at a lower priority. Software interrupts can execute asynchronously and are considered more important than most other things the kernel can do. They are, in short, a way to say "do some work as soon as possible after we finish handling hardware interrupts." 

All known softirqs are hard-coded in the kernel source; there is no mechanism to add them dynamically. There are currently ten softirqs defined, though one of them is not used. The ones that _are_ used are for network processing, block request processing, block interrupt polling, read-copy-update processing, scheduler housekeeping, and tasklet processing — the tasklet mechanism is a sort of wrapper around the softirq layer. Each subsystem that uses softirqs registers a handler to be called when an interrupt is pending; signaling of software interrupts is done with a call to `raise_softirq()`. 

There are a few places where software interrupt handlers actually get run, including immediately after a hardware interrupt handler completes. So softirqs signaled from interrupt handlers will often be handled almost immediately. Softirq processing can be disabled by kernel code (taking a spinlock will do it, for example); softirq handlers can be run once they are enabled again (once the spinlock is released, for example). That means that, in practice, there are many points throughout the kernel where softirqs can execute. 

One problem with the softirq mechanism is that it can create arbitrary latencies at almost any point in the kernel where softirq handlers can be run; there is often a _lot_ of work that is done in those handlers. Thus, developers working on the realtime patches have grumbled about softirqs for years and have [changed how they are processed](<https://lwn.net/Articles/520076/>) in realtime kernels. In the mainline kernel, this problem has been partially addressed by trying to limit the amount of time spent processing softirqs. If any call to `__do_softirq()` (the function that actually calls softirq handlers) finds itself running for more than 2ms, processing will stop and the `ksoftirqd` kernel thread will be woken up to finish the job. So, if the softirq load gets to high, it gets pushed into a thread where it has to compete with the rest of the work the system is doing. 

For the most part, this mechanism works well enough that it has not been removed, even though a number of developers would like to see it go. On the other hand, any attempt to add more softirq sources would almost certainly encounter strong pushback, which is why new ones are almost never added. Anybody thinking about doing so will be directed toward workqueues, or, if nothing else will suffice, tasklets. 

#### Softirqs and NAPI polling

The networking subsystem uses two software interrupts, one each for transmit and receive processing. The receive softirq is where NAPI processing (the polling of interfaces for new packets) is done. If there are a lot of packets to handle, this processing can take a long time. As with the softirq subsystem itself, the networking stack imposes a limit on how much time it will spend accepting packets in the softirq handler. That limit, though, is set to two jiffies — as much as 20ms, depending on the system's clock speed. In other words, the networking code increases the maximum time spent continuously handling softirqs by as much as a factor of ten. 

Of course, one wants the networking stack to have the CPU time it needs to deal with the flow of packets into the system. But it is also necessary to leave sufficient time for user space to actually do something with those packets. As Paolo Abeni reported in the introduction to his [threadable NAPI poll loop patch set](<https://lwn.net/Articles/686985/>), that doesn't always happen: 

Under high network load, the softirq loop can take nearly 100% of a given CPU, leaving very little time for use space processing. On single core hosts, this means that the user space can nearly starve. 

His solution was to (at the system administrator's option) move NAPI processing out of softirqs entirely, and into its own dedicated kernel thread. Even if no other configuration is done, this change allows the scheduler to balance NAPI processing against the needs of the other runnable threads on the system, giving user space a chance to run. The results, Paolo said, are good: 

With the default scheduling policy, the starvation issue observed on single vCPU guest under UDP flood is solved and the throughput measured under heavy overload is quite stable around the peak performance. 

Paolo's solution looks impressive, but it failed to impress the networking developers. Eric Dumazet [rejected the patch](<https://lwn.net/Articles/687629/>), saying: 

We already have ksoftirqd to normally cope with the case you are describing. If it is not working as intended, please identify the bugs and fix them, instead of adding yet another tests in fast path and extra complexity in the stack. 

Paolo was not pleased with this response, but he was able to come back with a bit of important information: why `ksoftirqd`, as it is implemented in current kernels, is not a sufficient solution to the problem. 

It seems that `ksoftirqd` will process packets for the two jiffies allowed by the networking code, then yield so that user space can run; Paolo estimated that, on his system, about 640 packets would be processed during this time. Once user space gets a chance to run, it will make a system call to retrieve a packet; the code implementing the system call will, at some point, find itself taking and releasing locks. But the point where a lock is released is an opportunity for the kernel to do softirq processing, and that is exactly what happens: softirq processing is done in the calling process's context. In Paolo's case, that in-context softirq processing would handle another 640 packets before returning a single packet to user space — the kernel, in other words, was processing nearly 1,300 packets before letting user space have even one of them. That is truly a system that is out of balance. 

To Paolo, this behavior was a good reason to move NAPI processing into a more controllable context, but Eric's [response](<https://lwn.net/Articles/687630/>) was, simply: ""Looks you found the bug then. Have you tried to fix it?"". After a bit, Eric came up with [a simple fix](<https://lwn.net/Articles/687631/>) of his own. With the resulting patch, waking up the `ksoftirqd` thread will set a per-CPU flag. Any code that would fire off softirq processing in process context first checks that flag; if it is set, the processing is skipped. This has the effect of forcing all softirq processing over to `ksoftirqd` once it has been invoked. 

Paolo [did some testing](<https://lwn.net/Articles/687633/>) and reported performance figures close to what were obtained with his patch. Even so, he preferred his patch, but indicated acceptance that it was probably not going to be merged. Eric's patch, after some further tweaking, looks likely to land in 4.7. 

One of Paolo's reasons for taking the approach he did was to avoid any possible effects on other softirq users. As Eric [put it](<https://lwn.net/Articles/687634/>), this was a natural tendency: ""Right, we are networking guys, and we feel that messing with such core infra is not for us."" But the long-term maintainability of the kernel depends on fixing problems where they are found rather than adding new mechanisms to work around those problems. In this case, a small problem would appear to have been correctly fixed. The rather larger problem (as seen by some developers) represented by the simple existence of the softirq code remains untouched, though.
