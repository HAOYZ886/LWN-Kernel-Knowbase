---
title: Relief for insomniac tracepoints
url: https://lwn.net/Articles/835426/
date: "October 29, 2020"
category: "BPF-Tracing; Tracing-with BPF"
author: "By Jonathan Corbet October 29, 2020"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
October 29, 2020

The kernel's tracing infrastructure is designed to be fast and to interfere as little as possible with the normal operation of the system. One consequence of this requirement is that the code that runs when a tracepoint is hit cannot sleep; otherwise execution of the tracepoint could add an arbitrary delay to the execution of the real work the kernel should be doing. There are times, though, that the ability to sleep within a tracepoint would be handy, delays notwithstanding. The [sleepable tracepoints patch set](<https://lwn.net/ml/linux-kernel/20201023195352.26269-1-mjeanson@efficios.com/>) from Michael Jeanson sets the stage to make it possible for (some) tracepoint handlers to take a nap while performing their tasks — but stops short of completing the job for now. 

Within the kernel, the tracing machinery has no need to sleep; its task is normally to package up the data associated with a given tracepoint and place the result into a ring buffer for transport to user space. This work can be accomplished without the need to wait for any outside events. The use cases driving the push for sleepable tracepoints thus must come from elsewhere — from BPF programs attached to tracepoints by user space, in particular. These programs are currently limited to accessing data in kernel space, which can always be done without the need to sleep. There would be value, though, in the ability to look at user-space data in a tracepoint handler as well. This data is not guaranteed to be resident in RAM when the handler tries to access it; should it not be present, a page fault will result. Handling page faults can take an arbitrary amount of time, during which the faulting process must be put to sleep. 

In current kernels, this possibility prevents access to user-space data from tracepoint handlers. Specifically, it means that tracers cannot dereference pointers passed from user space. Thus, for example, a tracepoint running on entry to the [`openat2()`](<https://man7.org/linux/man-pages/man2/openat2.2.html>) system call can see the pointer to the `open_how` structure passed by user space, but is unable to examine the contents of the structure itself. 

There is nothing about tracepoints that inherently makes sleeping impossible — at least, for those tracepoints that are executed when the kernel is not running in atomic context. But the BPF subsystem has long had its own rule that BPF programs could not sleep. That will change in the 5.10 kernel, though, thanks to the addition of [sleepable BPF programs](<https://lwn.net/Articles/825415/>), which no longer have this constraint. Only certain types of BPF programs are allowed to block; in 5.10, tracing programs are on that list. There will be no users of this ability in the 5.10 release, though. 

Jeanson's patch set lays the groundwork for the addition of such a user, establishing the infrastructure to support the attachment of sleepable BPF programs to specific tracepoints. This ability must be supported with care since, as noted above, the kernel is often running in a context where sleeping is a bad idea. Specifically, a sleepable BPF program can only be attached to a tracepoint located in a region of code where sleeping is allowed in general. 

There is no way to know automatically whether a given tracepoint can safely sleep or not, so existing tracepoints will not allow the attachment of sleepable BPF programs without explicit modification to that effect. Tracepoints are added to kernel code with the `TRACE_EVENT()` macro, along with a few variants; the brave of heart can see the horrifying macro-magic details in [include/linux/tracepoint.h](<https://elixir.bootlin.com/linux/latest/source/include/linux/tracepoint.h>). Jeanson's patch set adds a new macro called `TRACE_EVENT_FN_MAYSLEEP()` as a variant of `TRACE_EVENT_FN()`, which defines a tracepoint that has associated registration and unregistration functions. Switching an existing tracepoint to the new macro indicates that it is safe to attach sleepable programs there. 

The most significant change within those macros is that, if a tracepoint is marked as accepting sleepable programs, the tracers called when that tracepoint is hit will be run with preemption enabled. That is a necessary precondition to being able to handle page faults, but it also changes the expectations under which all of those tracers were written. The tracers themselves will need modification to run safely with preemption enabled — work that has not yet been posted. The patch set handles that situation, for now, by modifying the ftrace, perf, and BPF tracers to explicitly disable preemption internally, thus avoiding any unfortunate surprises. 

As noted above, the use case that is driving this work is following pointers passed to system calls from user space. So it is not surprising that the first user of this capability will be system-call tracing. Jeanson's patch set changes the system-call entry and exit tracepoints to use `TRACE_EVENT_FN_MAYSLEEP()`, thus setting the stage for the attachment of sleepable programs that could rummage around in user-space memory in response to system calls. 

There is only one piece that is missing at this point: actually fixing up the tracers and using the new infrastructure to attach and run sleepable BPF programs. As the cover letter to the patch set notes: 

This series only implements the tracepoint infrastructure required to allow tracers to handle page faults. Modifying each tracer to handle those page faults would be a next step after we all agree on this piece of instrumentation infrastructure. 

This may seem like a strange place to stop, just before making everything actually work, but changes at this point could have significant effects on the subsequent patches. 

Based on the discussion so far, it doesn't appear that there is any need for big changes at this level of the code; most of the comments relate to details around the edges. If that situation holds, we should expect to see patches in the near future that finish the job and enable the attachment of sleepable tracepoint programs. That may well lead to another increase in the capability of the tracing infrastructure for Linux.
