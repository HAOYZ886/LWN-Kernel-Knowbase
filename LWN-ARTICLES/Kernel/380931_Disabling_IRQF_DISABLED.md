---
title: Disabling IRQF_DISABLED
url: https://lwn.net/Articles/380931/
date: "March 30, 2010"
category: Interrupts
author: "By Jonathan Corbet March 30, 2010"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
March 30, 2010

Interrupt handlers run asynchronously in response to signals from the hardware. Since they pull the CPU away from whatever it was doing before, handlers are supposed to be very quick; they should, in most cases, tell the hardware to be quiet, arrange for any followup work to be done, and get out of the way. Historically, the situation has not been so simple, though, leading to a distinction between "fast" and "slow" handlers in the earliest days of Linux. It now seems, though, that this distinction could disappear as early as 2.6.35. 

The core distinction between fast and slow handlers is this: fast handlers are run with further interrupts disabled, while slow handlers run with interrupts enabled. A slow handler, thus, can be interrupted by another handler, while a fast handler cannot. In an ideal world, slow handlers would not exist; they would all get their work done quickly and not monopolize the CPU, so there would be no point in interrupting them. In the real world, which includes problematic hardware, slow processors, and developers of varying ability, slow handlers have been a fact of life. The nature of some hardware (old IDE controllers, for example) makes it hard to avoid doing a lot of work in the interrupt handler. Meanwhile, other types of devices must have exceedingly fast interrupt response to avoid loss of data; a classic example here is a number of serial ports which are able to buffer exactly one character in the UART. The slow IDE work could not be allowed to delay serial processing; thus, the IDE interrupt handler had to be a slow one. 

Over time, though, the situation has changed. Hardware has gotten smarter and better able to handle interrupt response latency. CPUs have gotten faster, so even a relatively slow handler can get a lot of work done quickly. The needs of the realtime tree (and other latency-sensitive workloads) have motivated the reworking of the worst interrupt-time offenders, and improvements in the kernel's deferred work mechanisms have made it easier to move work out of handlers. So the need for the distinction between the two types of interrupt handlers has been fading. 

Simultaneously, problems associated with the fast/slow dichotomy have been growing. There is no way to run handlers for interrupts on shared lines (found on any system with a PCI bus) with interrupts disabled, because any other handler for a device on the same line can enable interrupts. Allowing interrupt handlers to interrupt each other leads to worse cache behavior and unpredictable completion times. What set off the recent discussion, though, was [this patch](<https://lwn.net/Articles/380937/>) from Andi Kleen which was aimed at addressing another problem: deeply nested interrupt handlers can overflow the processor's interrupt stack - a situation from which good things cannot be expected to ensue. 

Andi's solution is to monitor the depth of the interrupt stack within the core kernel's interrupt-handling code. Should the stack become more than half full, the core code will no longer enable interrupts before calling slow handlers. In effect, it treats slow handlers as if they were fast handlers for the duration of the stack-space squeeze. This patch solved the problem that was being observed, but it ran into some trouble; in particular, Thomas Gleixner did not hesitate to make his dislike for the patch known. Your editor will try to rephrase the argument in slightly more polite terms; according to Thomas, the patch implemented a solution which was unreliable at best, was liable to create significant latencies in the system, and which ignored the real problem. 

Said real problem, according to Thomas, is the fact that slow handlers exist at all. He would like to see a world where all interrupt handlers are run with interrupts disabled, and where all of those handlers get their work done quickly. Any extended interrupt processing should be moved to threaded handlers. [In summary](<https://lwn.net/Articles/380940/>): 

So what's the point of running a well written (short) interrupt handler with interrupts enabled ? Nothing at all. It just makes us deal with crap like stacks overflowing for no good reason. 

Linus initially [squashed the idea](<https://lwn.net/Articles/380942/>), saying that a world where we only have fast handlers is not really possible: 

So Thomas, you're wrong. We can't fix all irq handlers to be really quick, and we MUST NOT run them with all other irq's disabled if they are not - because that obviates the whole point. 

It is interesting to note, though, that this position shifted over time. Linus (and others) expressed a number of concerns about running all handlers with interrupts disabled: 

  * The handlers for some devices simply have to do a lot of work, and that cannot be easily changed. Embedded systems, in particular, can have fussy hardware and slow processors. 

  * Some handlers will not work properly if interrupts are not enabled. In the past, some drivers have done things like waiting for a certain amount of time to pass (as reflected in changes to the `jiffies` variable). This dubious practice fails outright if interrupts are disabled: the timer interrupt will be blocked, and `jiffies` will not advance. 

  * Some hardware simply has strict latency requirements which cannot wait for another interrupt handler to finish its job. 

Looking at all these worries, one might well wonder if a system which disabled interrupts for all handlers would function well at all. So it is interesting to note one thing: any system which has the lockdep locking checker enabled has been running all handlers that way for some years now. Many developers and testers run lockdep-enabled kernels, and they are available for some of the more adventurous distributions (Rawhide, for example) as well. So we have quite a bit of test coverage for this mode of operation already. 

Another thing that happened over the last few years was the integration of the dynamic tick code, which disables the clock tick when the system is idle. Clock ticks are not turned back on for interrupt handlers. So any handler which expects `jiffies` to change while it is running will, sooner or later, go into a rather undignified infinite loop. Users tend to notice that kind of behavior, so most drivers which behave this way have long since been fixed. 

Finally, the realtime tree developers have spent a great deal of time tracking down sources of latency; excessive time spent in interrupt handlers is one of the worst of those. So drivers which control hardware of interest have generally been fixed. The addition of threaded interrupt handlers has made it easier to fix drivers; most of the code can simply be pushed into the threaded handler with no other change at all. 

Given all of this, Ingo Molnar [felt confident](<https://lwn.net/Articles/380945/>) in saying: 

I'm fairly certain, based on having played with those aspects from many angles that disabling irqs in all drivers should work just fine today already. 

After hearing this from a few core developers, and after doing some research of his own, Linus eventually [stopped opposing the idea](<https://lwn.net/Articles/380946/>) and started talking about how it should be implemented. Thomas then posted [a patch](<http://lwn.net/Articles/380536/>) implementing the change. With this patch, the `IRQF_DISABLED` flag (used to indicate a fast handler) becomes a no-op; it is expected to be removed altogether in 2.6.36. 

There are still some [concerns](<https://lwn.net/Articles/380948/>) about the change, especially with regard to slow hardware on embedded systems. In some of these cases, the problem can be solved with threaded interrupt handlers. Some developers worry, though, that threaded handlers impose too much latency on interrupt response. Improving on that situation is a task for the future; in the mean time, some interrupt handlers may just have to enable interrupts internally to get the required behavior. The preferred function for this purpose is `local_irq_enable_in_hardirq()`; its use can already be found in the IDE layer. 

Since all of the technical obstacles have seemingly been overcome, chances are good that this patch will find its way into the kernel in the 2.6.35 merge window.
