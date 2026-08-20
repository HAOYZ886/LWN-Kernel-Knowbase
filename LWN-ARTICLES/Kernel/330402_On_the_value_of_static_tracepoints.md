---
title: On the value of static tracepoints
url: https://lwn.net/Articles/330402/
date: "April 28, 2009"
category: "Development tools-Kernel tracing; Ftrace; Tracing"
author: "By Jonathan Corbet April 28, 2009"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
April 28, 2009

As has been well publicized by now, the Linux kernel lacks the sort of tracing features which can be found in certain other Unix-like kernels. That gap is not the result of a want of trying. In the past, developers trying to put tracing infrastructure into the kernel have often run into a number of challenges, including opposition from their colleagues who do not see the value of that infrastructure and resent its perceived overhead. More recently, it would seem that the increased interest in tracing has helped developers to overcome some of those objections; an ongoing discussion shows, though, that concerns about tracing are still alive and have the potential to derail the addition of tracing facilities to the kernel. 

Sun's DTrace is famously a dynamic tracing facility, meaning that it can be used to insert tracepoints at (almost) any location in the kernel. But the Solaris kernel also comes with [an extensive and well-documented set of static tracepoints](<http://docs.sun.com/app/docs/doc/817-6223>) which can be activated by name. These tracepoints have been placed at carefully-considered locations which facilitate investigations into what the kernel is actually doing. Many real-world DTrace scripts need only the static tracepoints and do no dynamic tracepoint insertion at all. 

There is clear value in these static tracepoints. They represent the wisdom of the developers who (presumably) are the most familiar with each kernel subsystem. System administrators can use them to extract a great deal of useful information without having to know the code in question. Properly-placed static tracepoints bring a significant amount of transparency to the kernel. As tracing capabilities in Linux improve, developers naturally want to provide a similar set of static tracepoints. The fact that static tracing is reasonably well supported (via FTrace) in mainline kernels - with more extensive support available via SystemTap and LTTng - also encourages the creation of static tracepoints. As a result, there have been recent patches adding tracepoints to [workqueues](<http://lwn.net/Articles/330004/>) and [some core memory management functions](<http://lwn.net/Articles/329577/>), among others. 

### Digression: static tracepoints

As an aside, it's worth looking at the form these tracepoints take; the design of Linux tracepoints gives a perspective on the problems they were intended to solve. As an example, consider the following tracepoints for the memory management code which reports on page allocations. The declaration of the tracepoint looks like this: 

```
#include <linux/tracepoint.h>
      
        TRACE_EVENT(mm_page_allocation,
    
    	TP_PROTO(unsigned long pfn, unsigned long free),
    
    	TP_ARGS(pfn, free),
    
    	TP_STRUCT__entry(
    		__field(unsigned long, pfn)
    		__field(unsigned long, free)
    	),
    
    	TP_fast_assign(
    		__entry->pfn = pfn;
    		__entry->free = free;
    	),
    
    	TP_printk("pfn=%lx zone_free=%ld", __entry->pfn, __entry->free)
    	);
```

That seems like a lot of boilerplate for what is, in a sense, a switchable `printk()` call. But, naturally, there is a reason for each piece. The `TRACE_EVENT()` macro declares a tracepoint - this one is called `mm_page_allocation` \- but does not yet instantiate it in the code. The tracepoint has arguments which are passed to at its actual instantiation (which we'll get to below); they are declared fully in the `TP_PROTO()` macro and named in the `TP_ARGS()` macro. Essentially, `TP_PROTO()` provides a function prototype for the tracepoint, while `TP_ARGS()` looks like a call to that tracepoint. 

These values are enough to let the programmer place a tracepoint in the code with a line like: 

```
trace_mm_page_allocation(page_to_pfn(page),
    			     zone_page_state(zone, NR_FREE_PAGES));
```

This tracepoint is really just a known point in the code which can have, at run time, one or more function pointers stored into it by in-kernel tracing utilities like SystemTap or Ftrace. When the tracepoint is enabled, any functions stored there will be called with the given arguments. In this case, enabling the tracepoint will result in calls whenever a page is allocated; those calls will receive the page frame number of the allocated page and the number of free pages remaining as parameters. 

As can be seen in the declaration above, there's more to the tracepoint than those arguments; the rest of the information in the tracepoint declaration is used by the Ftrace subsystem. Ftrace has a couple of seemingly conflicting goals; it wants to be able to quickly enable human-readable output from a tracepoint with no external tools, but the Ftrace developers also want to be able to export trace data from the kernel quickly, without the overhead of encoding it first. And that's where the remaining arguments to `TRACE_EVENT()` come in. 

When properly defined (the magic exists in a bunch of header files under `kernel/trace`), `TP_STRUCT__entry()` adds extra fields to the structure which represent the tracepoint; those fields should be capable of holding the binary parameters associated with the tracepoint. The `TP_fast_assign()` macro provides the code needed to copy the relevant data into that structure. That data can, with some changes merged for 2.6.30, be exported directly to user space in binary format. But, if the user just wants to see formatted information, the `TP_printk()` macro gives the format string and arguments needed to make that happen. 

The end result is that defining a tracepoint takes a small amount of work, but using it thereafter is relatively easy. With Ftrace, it's a simple matter of accessing a couple of debugfs files. But other tools, including LTTng and SystemTap, are also able to make use of these tracepoints. 

### The disagreement

Given all the talk about tracing in recent years, there is clearly demand for this sort of facility in the kernel. So one might think that adding tracepoints would be uncontroversial. But, naturally enough, it's not that simple. 

The first objection that usually arises has to do with the performance impact of tracepoints, which are often placed in the most performance-critical code paths in the kernel. That is, after all, where the real action happens. So adding an unconditional function call to implement a tracepoint is out of the question; even putting an `if` test around it is problematic. After literally years of work, the developers came up with a scheme ~~involving run-time code patching~~ that reduces the performance cost of an inactive tracepoint to, for all practical purposes, zero. Even the most performance-conscious developers have stopped fretting about this particular issue. But, of course, there are others. 

A tracepoint exists to make specific kernel information available to user space. So, in some real sense, it becomes part of the kernel ABI. As an ABI feature, a tracepoint becomes set in stone once it's shipped in a stable kernel. There is not a universal agreement on the immutability of kernel tracepoints, but the simple fact is that, once these tracepoints become established and prove their usefulness, changing them will cause user-space tracing tools to break. That means that, even if tracepoints are not seen as a stable ABI the way system calls are, there will still be considerable resistance to changing them. 

Keeping tracepoints stable when the code around them changes will be a challenge. A substantial subset of the developer community will probably never use those tracepoints, so they will tend to be unaware of them and will not notice when they break. But even a developer who is trying to keep tracepoints stable is going to run into trouble when the code evolves to the point that the original tracepoint no longer makes sense. One can imagine all kinds of cruft being added so that a set of tracepoints gives the illusion of a very different set of decisions than is being made in a future kernel; one can also imagine the hostile reception any such code will find. 

The maintenance burden associated with tracepoints is the reason behind Andrew Morton's opposition to their addition. With regard to the workqueue tracepoints, Andrew [said](<https://lwn.net/Articles/330456/>): 

If someone wants to get down and optimise our use of workqueues then good for them, but that exercise doesn't require the permanent addition of large amounts of code to the kernel. The same amount of additional code and additional churn could be added to probably tens of core kernel subsystems, but what _point_ is there to all this? Who is using it, what problems are they solving? 

We keep on adding all these fancy debug gizmos to the core kernel which look like they will be used by one person, once. If that! 

Needless to say, the tracing developers see the code as being more widely useful than that. Frederic Weisbecker gave [a detailed description](<https://lwn.net/Articles/330457/>) of the sort of debugging which can be done with the workqueue tracepoints. Ingo Molnar's [response](<https://lwn.net/Articles/330568/>) appears to be an attempt to hold up the addition of other types of kernel instrumentation until the tracepoint issue is resolved. Andrew remains unconvinced, though; it seems he would rather see much of this work done with dynamic tracing tools instead. 

As of this writing, that's where things stand. If these tracepoints do not get into the mainline, it is hard to see developers going out and creating others in the future. So Linux could end up without a set of well-defined static tracepoints for a long time yet - though it would not be surprising to see the enterprise Linux vendors adding some to their own kernels. Perhaps that is the outcome that the development community as a whole wants, but it's not clear that this feeling is universal at this time. If, instead, Linux is going to end up with a reasonable set of tracepoints, the development community will need to come to some sort of consensus on which kinds of tracing instrumentation is acceptable.
