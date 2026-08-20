---
title: "One ring buffer to rule them all?"
url: https://lwn.net/Articles/388978/
date: "May 26, 2010"
category: "Development tools-Kernel tracing; Tracing"
author: "By Jake Edge May 26, 2010"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jake Edge**  
May 26, 2010

One of the outcomes from the Collaboration Summit in April was a plan to create a tracing ring buffer implementation that would work for both [Ftrace](<https://rt.wiki.kernel.org/index.php/Ftrace>) and [LTTng](<http://lttng.org>). There was also hope that perhaps the separate ring buffer added for [perf](<https://perf.wiki.kernel.org/index.php/Main_Page>) could be folded in as well, so that the [vision of a single ring buffer](<http://lwn.net/Articles/300992/>) implementation in the kernel--from the 2008 Kernel Summit and Linux Plumbers Conference--could come to fruition. To that end, Steven Rostedt [posted](<https://lwn.net/Articles/389199/>) an RFC for a unified ring buffer, but before that conversation could really get going, it was diverted. It seems that Ingo Molnar thinks there are much bigger issues to resolve in the world of Linux tracing. 

A [ring buffer](<http://en.wikipedia.org/wiki/Circular_buffer>) is a circular data structure that is often used to hold data that is produced and consumed by separate processes without synchronization. For tracing, the data is produced by the kernel outside of any specific process's context, but the consumer is in user space. The kernel needs to hand out pages from the head of the ring buffer to user space for consumption, while ensuring that it doesn't overwrite that data as it writes to the tail of the buffer. 

In his RFC, Rostedt recounts the history of tracing ring buffers, noting that the Ftrace ring buffer did not become lockless until after perf came along. Since perf needed to be able to record events in non-maskable interrupt (NMI) contexts, it couldn't use the Ftrace ring buffer; instead, it used one of its own, written by Peter Zijlstra. Eventually, Rostedt changed the Ftrace ring buffer to be [lockless](<http://lwn.net/Articles/340400/>), but at that point, it was too late for perf. In addition, the perf ring buffer allows user space to `mmap()` its pages, while the Ftrace ring buffer was geared to in-kernel users and `splice()`. 

So the kernel already has two tracing ring buffers, but there is also an out-of-tree ring buffer to consider, which is the one used by LTTng. That ring buffer has seen a lot of use in production Linux shops as well as being integrated into various embedded Linux distributions. In addition, much as was done with RCU, LTTng project lead Mathieu Desnoyers did a formal proof of the correctness of that ring buffer algorithm. 

At the Collaboration Summit, there was a belief that the LTTng ring buffer could be extended to support all of the Ftrace use cases. It would seem that since then, Desnoyers has come up with ways to support the perf ring buffer use cases as well. Both Rostedt and Desnoyers would like to nail down all of the requirements for (at least) tracing ring buffers and put together a single implementation that works for all of them. Desnoyers has [put together a git tree](<https://lwn.net/Articles/389206/>) that includes a bare bones ring buffer as a starting point. 

But Andi Kleen is [not convinced](<https://lwn.net/Articles/389208/>) of the need for a unified ring buffer, at least one that encompasses other non-tracing uses. The Ftrace ring buffer is very complex and ""too clever""; plenty of other subsystems use kfifo: 

In fact there are far more users of it than of your ring buffer. And it's really quite simple and easy to use. And it works fine for them. 

He goes on to ask ""If perf's current ring buffer works for it why not keep using it?"". Rostedt more or less [agrees](<https://lwn.net/Articles/389209/>) with the complexity argument, but notes that there tends to be a misconception when people first look at the problem: 

You also bring up a point that I try very hard to get across. When people think of a ring buffer, they think of the ones that they created in CS101, not realizing that when you are dealing with production systems, handling the requirements makes the buffering much more complex. 

Desnoyers also [points out](<https://lwn.net/Articles/389210/>) that tracing has some requirements that other ring buffer users may not have: 

One requirement seems to be shared for most tracing heavy users: it has to be blazingly fast. This is a requirement that is commonly overlooked by the "very simple" implementations. 

There are advantages to sharing a ring buffer implementation among the different tracing solutions beyond just fulfilling Linus Torvalds's mandate from the 2008 Kernel Summit. High-performance ring buffer implementations tend to be more complex than standard code according to Desnoyers, ""so it's good if we can focus our review efforts on a single ring buffer"". In addition, if there is a common implementation, any performance tweaks will help all of its users. 

There is another underlying reason for a single ring buffer, though. Molnar would like to see Ftrace phased out, with its functionality moved into perf. Rostedt is not necessarily opposed, but in order to do that, there needs to be some ring buffer implementation that supports both. The question is: how to get there? 

Rather than directly commenting on the idea of a unified ring buffer itself, Molnar [posted](<https://lwn.net/Articles/389212/>) a request for discussion on ""Future tracing/instrumentation directions"". In it, he makes the case for moving Ftrace functionality to perf and suggests that Rostedt and Desnoyers help Zijlstra with ""performance and simplification work"" of the perf ring buffer: 

[...] The last thing we need is yet another replace-everything event. 

If we really want to create a new ring buffer abstraction i'd suggest we start with Peter's, it has a quite sane design and stayed simple and flexible [...] 

Molnar believes that the performance of the current ring buffers ""sucks, in a big way. I've recently benchmarked it and it takes hundreds of instructions to trace a single event"". He also thinks that the current ""ftrace/perf duality"" is hurting developers and users. One of the main things he would like to eliminate is the debugfs interface for Ftrace, but that will take some time: 

The main detail here to be careful of is that lots of people are fond of the simplicity of the /debug/tracing/ debug UI, so when we replace it we want to do it by keeping that simple workflow (or best by making it even simpler). I have a few ideas how to do this. 

There's also the detail that in some cases we want to print events in the kernel in a human readable way: for example EDAC/MCE and other critical events, trace-on-oops, etc. This too can be solved. 

Thomas Gleixner and Ted Ts'o both spoke up in favor of the kernel events and tracepoints being discoverable from user space. Currently, that is well-supported by Ftrace using its debugfs interface, and both would like to see that continue. Gleixner wants simple tracing tools for embedded devices--eventually made a part of BusyBox--that don't have to change when new tracepoints or events are added. Ts'o on the other hand wants to be able to have bash-completion scripts that can figure out tracepoint names. Molnar [agreed](<https://lwn.net/Articles/389279/>) that it is important to maintain that ability going forward. 

There is some debate about how badly the Ftrace ring buffer performs. Molnar is quite [critical](<https://lwn.net/Articles/389286/>) of its performance, but Rostedt [disputes](<https://lwn.net/Articles/389288/>) some of those findings. In the end, there doesn't seem to be much disagreement that a better performing ring buffer could be created, there is just a question of how it should be approached. 

Rostedt does not think that starting with the perf ring buffer is the right approach: ""The current ring buffer in perf is very coupled with the perf design."" Molnar, though, is leery of bringing yet another ring buffer implementation into the picture: 

We've got two ring buffer implementations right now, one has to be replaced. 

It doesnt mean we should disrupt _two_ implementations and put in a third one, unless there are strong technical reasons for doing so. 

Those strong technical reasons may be found in the performance numbers for the various implementations. If Rostedt and Desnoyers can produce a ring buffer that works for Ftrace and perf, while performing better than either existing implementation, it seems likely that it would find a clear path into the kernel. As the discussion has trailed off, one gets the sense that the participants are now benchmarking and tweaking their implementations to try to achieve that. 

The ring buffer implementation is at the heart of any Linux tracing solution; its capabilities and performance will largely dictate how intrusive tracing is on the rest of the system, which in turn impacts how useful the tracing output is. The fact that several smart developers have yet to come up with a super-low-impact solution speaks volumes about the difficulty of the problem. With all of the work that is currently going on, though, it seems likely that one way or another a high-performance ring buffer--with lower overall complexity--will come about. 

Another interesting outcome from this discussion, short though it may have been, is that we are likely to see Ftrace fade away over time. The functionality won't disappear, it and much of the Ftrace code would be moved into perf, but Ftrace itself--which really started the (relatively) recent mainline tracing push--might well be gone sometime in the next few kernel development cycles.
