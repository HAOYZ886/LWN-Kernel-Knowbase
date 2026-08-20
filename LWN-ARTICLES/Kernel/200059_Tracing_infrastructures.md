---
title: Tracing infrastructures
url: https://lwn.net/Articles/200059/
date: "September 19, 2006"
category: "Development tools-Kernel tracing; KProbes"
author: 
---

Sometimes, things just do not go according to plan. Mathieu Desnoyers is the current maintainer of the [Linux Trace Toolkit](<http://ltt.polymtl.ca>), a kernel event tracing package which has, despite a significant user base, remained outside of the mainline for many years. He recently posted [a new LTT release](<https://lwn.net/Articles/200060/>) with the following introduction: 

Following an advice Christoph gave me this summer, submitting a smaller, easier to review patch should make everybody happier. 

What resulted was [a thread of hundreds of messages](<http://thread.gmane.org/gmane.linux.kernel.tracing/1651>), many of which could be considered to be impolite even by linux-kernel standards. Clearly, LTT has hit a nerve - especially surprising given that the points of real disagreement are minimal. 

At times, people have questioned whether the kernel needs any sort of tracing facility at all. That particular question would appear to have been resolved (affirmatively); the disagreement now would appear to be whether that tracing should be static or dynamic. Static tracing works by putting explicit tracepoints into the source code (they look like function calls); the tracing framework can then enable or disable those tracepoints at run time as desired. In a dynamic system, instead, tracepoints are injected into a running system, usually in the form of a breakpoint instruction. 

The kernel already has dynamic tracing in the form of [KProbes](<http://lwn.net/Articles/132196/>); LTT, instead, uses (primarily) a static model. So the biggest question, at least on the surface, has been over whether Linux needs a static tracing package in addition to the dynamic mechanism it has now. This debate revolves around a few points: 

  * Overhead, part 1: when tracing is not being used (the normal situation on most systems), dynamic tracepoints clearly have lower overhead: they do not exist at all. For all the work that is done to make static tracepoints be fast when they are not in use, they still exist, and will still have a (small) runtime cost. 

  * Overhead, part 2: when tracing _is_ being used, static tracepoints will tend to be faster. The breakpoint mechanism used by KProbes can (in the current implementation) take about ten times as many CPU cycles as a static tracepoint. There are projects in the works (djprobes, in particular) which can reduce this overhead considerably; Ingo Molnar also, as part of the discussion, posted a series of patches which cut the KProbes overhead roughly in half. 

One might wonder why overhead concerns people in this case. Tracing is often used to track frequent events, so a higher tracepoint overhead can slow things down in a noticeable manner. More to the point, though, heavyweight tracepoints can change the timing of events, leading to the dreaded "heisenbugs" which vanish when the developer actively looks for them. 

  * Maintenance overhead: some developers are concerned that the addition of static tracepoints to the kernel code will complicate the maintenance of that code. Tracepoints clutter the code itself, and they must continue to work into the indefinite future. In a sense, each one can be thought of as a little system call which, once placed, cannot be changed. Developers also worry that there will be pressure to add increasing numbers of these tracepoints over time. 

On the other hand, dynamic tracepoints impose a different sort of overhead: everybody who is interested in a set of tracepoints must take on the maintenance of those tracepoints. As the kernel changes, the tracepoints will need to move around to follow those changes. Keeping a set of dynamic tracepoints current can, in fact, be a nontrivial and tiresome job. Tools like [SystemTap](<http://sourceware.org/systemtap/>) help in this regard, but they are far from a complete solution at this time. Static tracepoints placed into the kernel code, instead, will continue to work as that code changes. 

  * Flexibility: dynamic tracepoints can be placed anywhere at any time, but static tracepoints require, at a minimum, a source code edit, rebuild, and reboot. Dynamic tracepoints can more easily support runtime filtering of events as well. On the other hand, static tracepoints currently are better at accessing local variables. 

  * Architecture support: KProbes are not currently implemented on all architectures, so they are not available to all Linux users. Static tracepoints tend to require less architecture-specific trickiness, and are thus easier to support universally. On the other hand, it has been argued, the addition of static tracepoints would take away much of the incentive architecture maintainers might have to make KProbes work. 

Reading through the discussion, one could be forgiven for going into a state of complete despair. The interesting thing, though, is that the level of disagreement is lower than one might think. There is a near consensus among the participants that there is a place for _both_ static and dynamic tracepoints. Static tracing of events of interest will help a lot of people - user-space developers and system administrators, not just kernel developers - understand what is going on in the system. Making all of these people figure out where to place, for example, a tracepoint to report scheduler changes in a specific kernel makes things a lot harder. 

The key point, however, is that the value of the static point is not really its static placement, but the fact that it is a clear indicator of where the tracepoint needs to be. So it has been suggested that an answer which might please everybody is to insert "markers" rather than tracepoints. These markers, which could live in a different section of the kernel image, are simply signs pointing out where a dynamic tracepoint should be inserted, should the need exist. To this end, Mathieu has posted [a simple marker patch](<https://lwn.net/Articles/200016/>); it was promptly fired upon for implementation issues, but there are few people who are opposed to the idea. 

So markers may well be the way this work goes forward. If the LTT code could be reworked around the marker concept, then the way might be clear for a discussion of what else needs to happen before that code could be merged (there are a number of issues to talk about there which have been, thus far, overshadowed by the current debate). After suitable consideration, a carefully-selected set of markers/tracepoints could be added to the mainline kernel, enabling anybody to easily hook into and monitor well-known events. Once the smoke clears, there might just be a viable solution which will please almost everybody.
