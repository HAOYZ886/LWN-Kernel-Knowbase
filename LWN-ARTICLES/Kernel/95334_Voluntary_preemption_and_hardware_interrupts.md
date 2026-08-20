---
title: Voluntary preemption and hardware interrupts
url: https://lwn.net/Articles/95334/
date: "July 27, 2004"
category: Interrupts; Latency; Voluntary preemption; irqd
author: 
---

Ingo Molnar's voluntary preemption work, described here [two weeks ago](<https://lwn.net/Articles/92806/>), has continued to progress. Indeed, since Mr. Molnar did not attend the kernel summit or OLS, this could well have been the fastest-moving kernel project over the last week. The [2.6.8-rc1-L2 version of the patch](<https://lwn.net/Articles/95332/>), released on July 27, claims a maximum 100μs latency - almost good enough, says Ingo, for hard real-time work. One of the methods used may raise some eyebrows, however. 

The core of the voluntary preemption patch stays true to its original intent: it adds scheduling points in places where the kernel may hold onto the CPU for overly long periods. As kernel testers report problems, Ingo has been going in and breaking up the offending bits of code. Ingo has also added a new knob to control the maximum number of sectors the block I/O subsystem will try to transfer at once; if block operations get too big, the IDE completion routines can take too long to perform their cleanup. 

That change pointed at a larger problem, however: some interrupt handlers can, despite conventions to the contrary, occupy the processor for a long time. While an interrupt handler is running, high-priority processes cannot run. Ingo decided to address this problem head on: he has moved hardware interrupt handling into process context. 

To do this, he had to change the core kernel interrupt dispatcher. That code now checks to see if the interrupt comes from the timer; if so, it is handled immediately, in the traditional manner. Otherwise, the IRQ number is added to a per-CPU list of pending hardware interrupts, and control returns to the scheduler without having actually serviced the interrupt. Calling the real interrupt handler falls to the `ksoftirqd` process (which has been renamed `irqd`). Once it is scheduled, it simply iterates through the list of pending interrupts and calls all of the registered handlers for each. Due to the change in context, the `pt_regs` structure is no longer available to the handler, but, since almost no interrupt handlers ever use that argument, that will not be a problem. 

The `irqd` process runs at a high priority, but a high-priority, real-time process can still preempt it. While it is dispatching an interrupt to its handler(s), `irqd` goes into a simulated interrupt mode and cannot be preempted. It drops out of that mode between interrupts, however, making scheduling possible. It is also possible for an interrupt handler to enable preemption at a given point with a call to `cond_resched_hardirq()` (or one of its variants). Without such a call, hardware interrupt handlers will not be put to sleep. 

There are no such calls in drivers in Ingo's current patch - at least, not directly. Ingo has also added a new version of `end_that_request_first()` (the function used to indicate partial completion of a block I/O request) which allows preemption. The IDE completion handler calls the new version, which makes it preemptable - even though it is an interrupt handler. 

Ingo claims some very good results from this work. The software latencies are now all very small. It would be interesting to see whether the redirecting of hardware interrupts has any effect on interrupt response latency, however. It remains to be seen whether a change of this magnitude will be accepted - especially since (involuntary) kernel preemption is supposed to be the real solution to latency problems. Building trust in involuntary preemption is an ongoing process, while the voluntary approach appears to have solved the problem now. In the end, that is likely to count for something. 

(Coincidentally, Scott Wood has posted [a different patch](<https://lwn.net/Articles/95387/>) moving interrupt handlers into process context. His patch creates a separate thread for each interrupt, which allows the priority of each interrupt handler to be set independently. There is also an `SA_NOTHREAD` flag to `request_irq()` which allows a driver to request the old, IRQ-context mode. Ingo has said that he is likely to adopt the multi-thread approach for his patch as well).
