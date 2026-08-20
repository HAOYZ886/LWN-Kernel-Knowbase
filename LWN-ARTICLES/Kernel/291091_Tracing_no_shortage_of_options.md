---
title: "Tracing: no shortage of options"
url: https://lwn.net/Articles/291091/
date: "July 22, 2008"
category: "Development tools-Kernel tracing; SystemTap; Tracing"
author: "By Jonathan Corbet July 22, 2008"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
July 22, 2008

Three weeks ago, LWN [looked at the renewed interest in dynamic tracing](<http://lwn.net/Articles/287906/>), with an emphasis on SystemTap. Tracing is a perennial presence on end-user wishlists; it remains a handy tool for companies like Sun Microsystems, which wish to show that their offerings (Solaris, for example) are superior to Linux. It is not surprising that there is a lot of interest in tracing implementations for Linux; the main surprise is that, after all this time, Linux still does not have a top-quality answer to DTrace - though, arguably, [Linux had a working tracing mechanism](<https://lwn.net/Articles/291092/>) long before DTrace made its appearance. 

Even a casual reader of the kernel mailing list will have noticed that there are a lot of tracing-related patches in circulation at the moment. There are so many, in fact, that it is hard to keep track of them all. So this article will take a quick look at the code which has been posted in an attempt to make the various options a bit clearer. 

### SystemTap

SystemTap remains the presumptive Linux tracing solution of choice. It is hampered by a few problems, though, including usability issues, a complete lack of static trace points in the mainline kernel, and no user-space tracing capability. On the usability side, we are seeing a few more kernel developers trying to put SystemTap to work and posting about the problems they are having. If one takes as a working hypothesis the notion that, if kernel hackers cannot make SystemTap work, many other users are likely to encounter difficulties as well, then one might conclude that addressing the reported problems would be a priority for the SystemTap developers. 

The SystemTap developers do seem to be interested in these reports, which is a good sign. There are other things happening in the SystemTap arena, including the [release of version 0.7](<http://lwn.net/Articles/290127/>) on July 15\. This release adds a number of new features and tapsets, and a substantial set of examples as well. Meanwhile, Anup Shan has posted [an interesting integration of SystemTap and the fault injection framework](<https://lwn.net/Articles/289932/>), allowing tapsets to control fault injection and trace the results. 

James Bottomley has been playing some with the SystemTap code; one result of that work is [changes to SystemTap's internal relocation code](<http://lwn.net/Articles/290159/>) in an attempt to make it more acceptable for mainline kernel inclusion. There can be no doubt that the out-of-tree nature of much of the SystemTap support code has made it harder for that code to progress, so any improvement which makes it more likely that some of this code will be merged is welcome. 

Also by James is [this patch](<http://lwn.net/Articles/289339/>) implementing a new way to put markers into the kernel. The addition of markers (or static tracepoints) has always been problematic in that many of these markers, by their nature, need to go into some of the hottest code paths in the kernel. To support dynamic tracing, these markers need to be available on production systems, so they must work without creating any significant performance regressions. Quite a bit of work has gone into the static marker code which is in the kernel (but mostly unused) now, but some developers are still uncomfortable with putting them into performance-critical paths. 

James's patch addresses these concerns by putting the tracepoints entirely outside of the code paths. Rather than add some sort of marker to the code, these markers just make a note of just where in the code the marker is supposed to be; this note is stored in a separate part of the kernel binary. That information is enough for a run-time tool to patch in an actual jump to a tracing function should somebody want to see the information from that tracepoint. An additional benefit is that these markers do not interfere with any optimizations done by the compiler. Other solutions can insert optimization barriers which, while they do make life easier for the tracing subsystem, also affect the speed of the code even when the trace points are not active. 

### Ftrace

The text above said that the kernel's static tracepoint code is "mostly unused." That would have been better expressed as "completely," except that the 2.6.27 kernel will include a user in the form of the ftrace framework. One of the things which makes ftrace truly unique is that its documentation was not only merged before the code itself, but well before: the 2.6.26 kernel includes the excellent [Documentation/ftrace.txt](<http://lwn.net/Articles/290277/>) file. 

The ftrace (which stands for "function tracer") framework is one of the many improvements to come out of the realtime effort. Unlike SystemTap, it does not attempt to be a comprehensive, scriptable facility; ftrace is much more oriented toward simplicity. There is a set of virtual files in a debugfs directory which can be used to enable specific tracers and see the results. The function tracer after which ftrace is named simply outputs each function called in the kernel as it happens. Other tracers look at wakeup latency, events enabling and disabling interrupts and preemption, task switches, etc. As one might expect, the available information is best suited for developers working on improving realtime response in Linux. The ftrace framework makes it easy to add new tracers, though, so chances are good that other types of events will be added as developers think of things they would like to look at. 

### Tracepoints

The [kernel markers](<http://lwn.net/Articles/245671/>) mechanism is meant to be the way that static tracepoints are inserted into the kernel. To that end, a great deal of effort went into making these markers fast; they are, for all practical purposes, a set of no-op instructions until somebody wants to turn one on, at which point the real tracing code is patched into the running kernel. Since they were merged, however, kernel markers have been the subject of a few grumbles. 

In particular, kernel markers use a somewhat awkward mechanism to ensure that any arguments passed to the tracing function are interpreted correctly there. Each marker has a `printk()`-style format string associated with it; that string describes the type of each "argument" (a variable or expression within the code being traced). When tracing code activates a marker, it will supply a function to be called when the marker is hit and a format string describing the arguments that the function expects. The marker code will ensure that both format strings match; otherwise the marker will not be enabled. The problem is that the format string requires extra work to write and is only approximate in its specification of the types involved. These strings can make it clear that a given argument is a pointer, for example, but they say nothing about what type is pointed to. 

In response to various efforts to get around this issue, Mathieu Desnoyers (the original author of the kernel marker work) has proposed a new mechanism called [tracepoints](<http://lwn.net/Articles/290203/>). They are another way of putting static trace points into the kernel, but with a simpler and more type-safe way of putting the pieces together. 

With tracepoints, every trace point must be declared in a header file with a mildly ugly set of macros: 

```
#include <linux/tracepoint.h>
    
        DEFINE_TRACE(tracepoint_name,
                     TPPROTO(trace_function_prototype),
    		 TPARGS(trace_function_args));
```

This definition will create a new tracepoint called `tracepoint_name`. Any function attached to that tracepoint must have a function prototype as provided in the `TPPROTO()` macro; the names of the associated arguments are provided with `TPARGS()`. 

Perhaps this is better understood with an example. The tracepoints patch set includes quite a few static points for use with the LTTng tracing toolkit. There is one called `sched_wakeup` which fires whenever the scheduler wakes up a process. It is defined with: 

```
DEFINE_TRACE(sched_wakeup,
    	         TPPROTO(struct rq *rq, struct task_struct *p),
    		 TPARGS(rq, p));
```

The actual insertion of the tracepoint is a line like this: 

```
trace_sched_wakeup(rq, p);
```

Note the `trace_` prefix added to the supplied name. At this point in the code, a tracing function can be called with `rq` (the run queue of interest) and `p` (the process which is waking up) as parameters. Until an actual function is connected to the tracepoint, though, this declaration is essentially a no-op. Connection of a trace function is done through a call to: 

```
void my_sched_wakeup_tracer(struct rq *rq, struct task_struct *p);
    
        register_trace_sched_wakeup(my_sched_wakeup_tracer);
```

The `register_trace_sched_wakeup()` function (created as part of the `DEFINE_TRACE()` definition) will connect the supplied trace function to the tracepoint. The fact that the function prototype for the trace function is supplied as part of the tracepoint definition means that the compiler can perform thorough type checking; if the prototypes do not match up, compilation will fail. And that, in turn, should put an end to those embarrassing situations where turning on tracing causes the system to go down in flames. 

Interestingly, tracepoints have dispensed with much of the mechanism developed to minimize the runtime impact of kernel markers; in particular, they do not use the "immediate values" code. Profiling has shown that the performance impact of tracepoints is so low that there is little value in the added complexity of runtime patching of kernel code. Still, there are signs that some kernel developers will object to the addition of tracepoints in their current form. Developers want tracing support - but not at the cost of slower performance, even if that cost is hard to measure. 

### Tracehook

Finally, Roland McGrath recently surfaced with the [tracehook](<http://lwn.net/Articles/290954/>) patch set. Tracehook has a rather different focus; it is, essentially, a cleanup of the way the kernel handles the `ptrace()` system call. The tracehook patches try to organize all of the process tracing code (much of which is architecture-dependent) into one place where it can be dealt with as a unit. 

Tracehook is meant to be a first step toward the merging of a new version of the [utrace](<http://lwn.net/Articles/224772/>) code. Utrace has long been planned as the successor to the current `ptrace()` implementation, which has few admirers. But utrace has encountered a number of difficulties, so its path into the kernel has been slow. It disappeared from the lists entirely for a while, but a new version of the patches is said to be coming soon; Roland notes that he expects "some vigorous feedback" when that happens. 

The real importance of the `ptrace()` rework is that it is the path toward integrated tracing of kernel- and user-space events. And that, of course, is one of the biggest features offered by DTrace which is not yet available in SystemTap. Getting user-space tracing into the kernel - especially if it could work with the tracepoints already being inserted into some applications for DTrace - would be a major step forward for Linux. A lot of people will be watching when this patch set comes around again. 

Meanwhile, Roland would like to see the tracehook code merged for 2.6.27. He is late to the party, though, and this code has not done any time in linux-next. So it is not yet clear whether tracehook will go in before the merge window closes, or whether, instead, it will have to wait for 2.6.28. 

### In summary...

As can be seen, there is a lot happening in the area of tracing support for Linux. Tracing, it seems, is an idea whose time has come, at last. If the pieces described here can be merged and integrated into a unified framework, and if it can all be made sufficiently easy to use, the time for "DTrace envy" will come to an end. Those "ifs" are not small ones, though. There is quite a bit of work to be done yet; hopefully the current level of energy will remain until the job is done.
