---
title: The initial kGraft submission
url: https://lwn.net/Articles/596854/
date: "April 30, 2014"
category: Live patching
author: "By Jonathan Corbet April 30, 2014"
---

By **Jonathan Corbet**  
April 30, 2014

Rebooting the system to install a kernel patch is a pain. In many cases, it's worse than that: the system(s) involved may have uptime constraints that rule out reboots at arbitrary times. Users faced with such issues would often welcome a way to get a high-priority patch into the kernel without taking the system down. In response to this need, the [Ksplice patch set](<https://lwn.net/Articles/280058/>) was first posted in 2008. Ksplice never made it into the mainline, though, and, after its acquisition by Oracle, it has mostly disappeared from the Linux environment. Recently, a few alternative patches offering similar functionality have been announced; now one of them, [kGraft](<https://lwn.net/Articles/596776/>), has been posted for consideration. 

KGraft is the work of Jiří Kosina and Jiří Slaby, both working at SUSE. The approach they have taken is simpler than Ksplice, and lacks some of the capabilities that Ksplice offers (adding shadow members to structures, for example). On the other hand, the basic kGraft code is only a 600-line patch, and the process of applying a patch is quite a bit more lightweight, with less impact on the system. 

KGraft works by replacing entire functions in the kernel. Using the tools supplied with the patch set, a developer can turn a patch into a list of changed functions; the new versions of those functions are then compiled into a separate kernel module. When that module is loaded into the kernel, kGraft takes care of the task of replacing the existing, buggy functions with the new, fixed versions. 

That replacement is a bit of a tricky task; performing live surgery on the kernel while it is running is fraught with potential perils. The good news for the kGraft developers is that this problem has already been mostly solved for them. The function tracer (ftrace) subsystem needs to do just this type of live patching, so the ftrace developers have written the code to perform this function, debugged all the weird errors, and [taken the hits](<https://lwn.net/Articles/304105/>) when things didn't go quite as expected. All the kGraft developers have to do is to use the ftrace machinery — which can already intercept function calls in a running kernel — and use it to replace calls to the buggy functions with calls to the new code. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

Of course, there are still some hazards out there. Chief among them is this: what happens if a process is running in kernel space when the patch is applied? This process might call the old version of a function once, then call the new version shortly thereafter; depending on what has changed in the meantime, that could lead to confusion and just the sort of downtime that this whole exercise was meant to prevent. Looking at this problem, the kGraft developers concluded that problems could be avoided if no process sees two versions of the same function during a single trip into and out of kernel space. 

So the patch adds a marker to the `thread_info` structure in each process to track whether that process has left or returned to user space since the patch application process started. When a call to the old function is intercepted, the "slow stub" checks that flag for the running process; if that process has entered or left kernel space, it is deemed to be running in the "old universe," and it gets the old version of the function. Otherwise, control goes to the new version. Once every process in the system has made this transition to the new universe, the slow stub can be removed and the new function can be called unconditionally. 

What about processes that don't make this transition in a reasonable period of time? For example, a process stuck waiting for I/O on a network socket could remain in the kernel for a long time, preventing the cleanup of the graft operation. When Vojtěch Pavlik [talked about this work](<http://collaborationsummit2014.sched.org/event/0d798ed17bfaa0361d0aec63f2331c8d>) at the Linux Foundation Collaboration Summit in March, he mentioned a mechanism that would send a signal to slow processes to force this transition. That mechanism is not present in the posted patch set, though. What there is, instead, is a flag under `/proc` allowing the system administrator to identify processes that are gumming up the works. 

Next question: what about kernel threads, which have no user space to return to? Most kernel threads will, when they reach a suitable completion point, make a call to `kthread_should_stop()` to see whether they should exit. The kGraft code modifies `kthread_should_stop()` to reset the "old universe" flag. For threads that do not make such calls, the kGraft developers have inserted calls to `kgr_task_safe()`, which marks the new-universe transition, in suitable locations. 

Finally, what about interrupts? The kGraft code can block interrupts on the local CPU while it is making its changes, but it cannot block them globally. To make sure that no surprises come about while an interrupt handler is running, kGraft adds a per-CPU array to track whether each processor has run in process (non-interrupt) context. That flag is initially set to false, and `schedule_on_each_cpu()` is called to run a kGraft function, in process context, on each processor. That function, which cannot run until any pending interrupts on a given CPU have been serviced, will set the per-CPU flag. The function-replacement stub, meanwhile, will force interrupt code to run in the old universe on any CPU that has not yet set its per-CPU "new universe" flag. 

This patch set had just been posted as of this writing, so there is relatively little in the way of comments to report. Certainly there have been no objections to the overall approach expressed so far. There is obvious value to this functionality, so one would expect to see something merged at some point. The interesting variable here is the competing patches that are waiting in the wings. There's no place in the kernel for two or more runtime patching mechanisms, so either these patches will need to be unified in some fashion, or somebody will have to make a decision.
