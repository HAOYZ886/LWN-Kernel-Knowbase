---
title: Conditional tracepoints
url: https://lwn.net/Articles/417772/
date: "November 30, 2010"
category: "Development tools-Kernel tracing; Tracing"
author: "By Jonathan Corbet November 30, 2010"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
November 30, 2010

Tracepoints are small hooks placed into kernel code; when they are enabled, they can generate event information which can be consumed through the ftrace or perf interfaces. These tracepoints are defined via the decidedly gnarly `TRACE_EVENT()` macro which Steven Rostedt nicely [described in detail](<https://lwn.net/Articles/379903/>) for LWN earlier this year. As kernel developers add more tracepoints to the kernel, they are occasionally finding things which can be improved. One of those seems relatively simple: what if a tracepoint should only fire some of the time? 

Arjan van de Ven recently posted [a patch adding a tracepoint](<https://lwn.net/Articles/417774/>) to `__mark_inode_dirty()`, a function called deep within the virtual filesystem layer to, surprisingly, mark an inode as being dirty. Arjan's purpose is to figure out which processes are causing files to have dirty contents; that will allow tools like PowerTop to tell laptop users which process is causing their disk to spin up. The only problem is that some calls to `__mark_inode_dirty()` are essentially noise from this point of view; they happen, for example, when an inode is first created or is being freed. Tracing those calls could create a stream of useless events which would have to be filtered out by PowerTop, causing PowerTop itself to require more power. So it is preferable to avoid creating those events in the first place if possible. 

For that reason, Arjan made the call to the tracepoint be conditional: 

```
if (flags & (I_DIRTY_SYNC | I_DIRTY_DATASYNC | I_DIRTY_PAGES))
    	trace_writeback_inode_dirty(inode, flags);
```

This code works in that it causes the tracepoint to be "hit" only when an application has actually done something to dirty an inode. 

The VFS developers seem to have no objection to this tracepoint being added; the resulting information can be useful. But they didn't like the conditional nature of it. Part of the problem is that tracepoints are supposed to keep a low profile; developers want to be able to ignore them most of the time. Expanding a tracepoint to two lines and an `if` statement rather defeats that goal. But tracepoints are also supposed to not affect execution time. They have been carefully coded to impose almost no overhead when they are not enabled (which is most of the time); with techniques like [jump label](<https://lwn.net/Articles/412072/>), that overhead can be reduced even further. But that `if` statement, being outside of the tracepoint altogether, will always be executed regardless of whether the tracepoint is currently enabled or not. Multiply that test-and-jump across millions of calls to `__mark_inode_dirty()` on each of millions of machines, and the extra CPU cycles start to add up. 

So it was asked: could this test be moved into the tracepoint itself? One approach might be to put the test into the `TP_fast_assign()` portion of the tracepoint, which copies the tracepoint data into the tracing ring buffer. The problem with that idea is that, by that time, the tracepoint has already fired, space has been allocated in the ring buffer, etc. There is currently no mechanism to cancel a tracepoint hit partway through. There has, in the past, been talk of adding some sort of "never mind" operation which could be invoked within `TP_fast_assign()`, but that idea seems less than entirely elegant. 

What might happen, instead, is the creation of a variant of `TRACE_EVENT()` with a name like `TRACE_EVENT_CONDITION()`. It would take an extra parameter which would be, of course, another tricky macro. For Arjan's tracepoint, the condition would look something like: 

```
TP_CONDITION(
    	    if (flags & (I_DIRTY_SYNC | I_DIRTY_DATASYNC | I_DIRTY_PAGES))
    	    	return 1;
    	    else
    	    	return 0;
        ),
```

The tracepoint code would then test the condition before doing any other work associated with the tracepoint - but only if the tracepoint itself has been enabled. 

This solution should help to keep the impact of tracepoints to a minimum once again, especially when those tracepoints are not enabled. There is one potential problem in that the condition is now hidden deeply within the definition of the tracepoint; that definition is usually found in a special header file far from the code where the tracepoint is actually inserted. At the tracepoint itself, the condition which might cause it not to fire is not visible in any way. So, if somebody other than the initial developer wants to use the tracepoint, they could misinterpret a lack of output as a sign that the surrounding code is not being executed at all. That little problem could presumably be worked around with clever tracepoint naming, better documentation, or simply expecting users to understand what tracepoints are actually telling them.
