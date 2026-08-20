---
title: "KS2009: The future of perf events"
url: https://lwn.net/Articles/357481/
date: "October 19, 2009"
category: Performance monitoring
author: "By Jonathan Corbet October 19, 2009 LWN's 2009 Kernel Summit coverage"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
October 19, 2009

* * *

LWN's [2009 Kernel Summit coverage](<https://lwn.net/Articles/KernelSummit2009/>)

At last year's kernel summit, the perf events subsystem did not yet exist, even in out-of-tree form. Now, this subsystem (initially called "perf counters") is in the mainline and rapidly gaining new capabilities. Paul Mackerras led a discussion on recent and future developments for this code. 

The perf counters code brought with it the `perf_counter_open()` system call. Unlike previous work in this area, perf counters deals with single counters at a time, rather than presenting a view of the entire performance measurement unit (PMU) in the processor. Since being merged, this code has gained the ability to treat tracepoints as software events, collect user stack chains, perform filtering with C-like expressions, and work with kprobes. It also has been rebranded "perf events" to highlight its expanding scope. 

What does the future hold? One is better scheduling of counters. A typical hardware PMU can only measure a subset of all the possible events at any given time. If an application asks for more events than the PMU can handle, the kernel will simulate a larger PMU by having it count different events at different times. This scheduling of the PMU works, but it would be nice to have better control. User space should be able to attach priorities to counters, indicating which ones it really cares about. It would also be nice to be able to modify the scheduling interval used. 

Internally, there needs to be a well-defined API for users of the perf events subsystem. Support for more architectures is in the works; currently recent x86, server PowerPC, and SPARC64 processors are supported. Also desired is the ability to combine multiple counters to create abstract events. 

Currently the kernel has a number of ways to report events to user space. It seems that at least some of the developers working in this area would like perf events to subsume the others and become the one true event counting and reporting infrastructure. If this happens, ftrace would lose its ring buffer implementation and report its data through the (different) ring buffer used by perf events. There was no real opposition to this idea; nobody was willing to take the position that having duplicated ring buffer implementations in the kernel was a good idea. The fact that the author of the ftrace ring buffer was not present may also have contributed to the silence on this issue. 

There was some talk of working with user-space tracepoints. How should data from these tracepoints be integrated with kernel tracing data? One idea was to make some sort of `inject_tracepoint()` system call, but that was dismissed as being too slow. If user-space tracing data needs to go through the kernel, some sort of inbound ring buffer needs to be implemented. 

An alternative would be to record user-space trace data separately, then integrate it during postprocessing. The problem here is that it is surprisingly difficult to timestamp this data in a way that allows it to be reliably merged with kernel trace data. In the end, it may be necessary to just record trace information with CPU numbers and time stamp counter contents. 

There are also issues with the collection of backtrace information from user space. That is apparently hard to do if user space has not been built with frame pointers. But, on some architectures, using frame pointers has a serious performance cost, so most distributors are unwilling to build their systems that way. There was some debate over whether it's possible to reliably generate backtraces without frame pointer information, but no clear conclusion. 

The following session covered the related issue of tracepoints and user-space ABI issues. Tracepoint documentation was covered; all of those tracepoints are not as useful as they could be if they are not properly documented. There is an effort underway to document tracepoints in the kerneldoc system, but that was criticized as being the wrong approach. What is really needed, people said, was self-documenting tracepoints. The documentation could be put into a special kernel section, in the same way as module author and parameter information is stored now. The documentation could be extracted by tools when needed. That would require keeping an uncompressed kernel image around, but that is often necessary anyway. 

What about tracepoint stability? As developers discover tracepoints, they will create tools that rely on them; changing the tracepoints will then break the tools. Arjan van de Ven made the point that, without some sort of tracepoint stability, these tools would quickly become useless. So, at some level, tracepoints need to be seen as part of the kernel ABI. 

There is little appetite for casting every kernel tracepoint in cement, though; many (or most) of them are really just debugging aids for developers. So what is needed is a way to mark some tracepoints as being part of the stable ABI. It was asserted that this marking must be done in the code itself; that way developers will see the ABI status of a given tracepoint and avoid changing it. Presumably something like Arjan's [TRACE_EVENT_ABI](<http://lwn.net/Articles/354865/>) patch would be used. 

These sessions also included a quick demonstration of the `ftrace`, `perf`, and timechart tools. The demonstrations were interesting but not really amenable to a writeup here. The best thing for interested developers to do is to read the documentation and LWN articles about these tools and play with them for a while. 

[Next: LKML volume and related issues](<https://lwn.net/Articles/357483/>)
