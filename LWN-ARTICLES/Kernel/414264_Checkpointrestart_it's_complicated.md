---
title: "Checkpoint/restart: it's complicated"
url: https://lwn.net/Articles/414264/
date: "November 9, 2010"
category: Checkpointing; DMTCP
author: "By Jonathan Corbet November 9, 2010"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
November 9, 2010

At the recent [Kernel Summit checkpoint/restart discussion](<https://lwn.net/Articles/412749/>), developer Oren Laadan was asked to submit a trimmed-down version of the patch which would just show the modifications to existing core kernel code. Oren duly responded with a "[naked patch](<https://lwn.net/Articles/412887/>)" which, as one might have expected, kicked off a new round of discussion. What many observers may not have expected was the appearance of an alternative approach to the problem which has seemingly been under development for years. Now we have two clearly different ways of solving this problem but no apparent increase in clarity; the checkpoint/restart problem, it seems, is simply complicated. 

The responses to Oren's patch will not have been surprising to anybody who has been following the discussion. Kernel developers are nervous about the broad range of core code which is changed by this patch. They don't like the idea of spreading serialization hooks around the kernel which, the authors' claims to the contrary notwithstanding, look like they could be a significant maintenance burden over time. It is clear that kernel checkpoint/restart can never handle all processes; kernel developers wonder where the real-world limits are and how useful the capability will be in the end. The idea of moving checkpointed processes between kernel versions by rewriting the checkpoint image with a user-space tool causes kernel hackers to shiver. And so on; none of these worries are new. 

Tejun Heo [raised all these issues](<https://lwn.net/Articles/414268/>) and more. He also called out an interesting alternative checkpoint/restart implementation called [DMTCP](<http://dmtcp.sourceforge.net/>), which solves the problem entirely in user space. With DMTCP in mind, Tejun concluded: 

I think in-kernel checkpointing is in awkward place in terms of tradeoff between its benefits and the added complexities to implement it. If you give up coverage slightly, userland checkpointing is there. If you need reliable coverage, proper virtualization isn't too far away. As such, FWIW, I fail to see enough justification for the added complexity. 

As one might imagine, this post was followed by an extended conversation between the in-kernel checkpoint/restart developers and the DMTCP developers, who had previously not put in an appearance on the kernel mailing lists. It seems that the two projects were each surprised to learn of the other's existence. 

The idea behind DMTCP is to checkpoint a distributed set of processes without any special support from the kernel. Doing so requires support from the processes themselves; a checkpointing tool is injected into their address spaces using the `LD_PRELOAD` mechanism. DMTCP is able to checkpoint (and, importantly, restart) a wide variety of programs, including those running in the Python or Perl interpreters and those using GNU Screen. DMTCP is also used to support the [universal reversible debugger project](<http://urdb.sourceforge.net/>). It is, in other words, a capable tool with real-world uses. 

Kernel developers naturally like the idea of eliminating a bunch of in-kernel complexity and solving a problem in user space, where things are always simpler. The only problem is that, in this case, it's not necessarily simpler. There is a surprising amount that DMTCP can do with the available interfaces, but there are also some real obstacles. Quite a bit of information about a process's history is not readily available from user space, but that history is often needed for checkpoint/restart; consider tracking whether two file descriptors are shared as the result of a `fork()` call or not. To keep the requisite information around, DMTCP must place wrappers around a number of system calls. Those wrappers interpose significant new functionality and may change semantics in unpredictable ways. 

Pipes are hard for DMTCP to handle, so the `pipe()` wrapper has to turn them into full Unix-domain sockets. There is also an interesting dance required to get those sockets into the proper state at restart time. The handling of signals - not always straightforward even in the simplest of applications - is made more complicated by DMTCP, which also must reserve one signal (`SIGUSR2` by default) for its own uses. The system call wrappers try to hide that signal handler from the application; there is also the little problem that signals which are pending at checkpoint time may be lost. Checkpointing will interrupt system calls, leading to unexpected `EINTR` returns; the wrappers try to compensate by automatically redoing the call when this happens. A second VDSO page must be introduced into a restarted process because it's not possible to control where the kernel places that page. There's a "virtual PID" layer which tries to fool restarted processes into thinking that they are still running with the same process ID they had when they were checkpointed. 

There is an interesting plan for restarting programs which have a connection to an X server: they will wrap Xlib (not a small interface) and use those wrappers to obtain the state of the window(s) maintained by the application. That state can then be recreated at restart time before reconnecting the application with the server. Meanwhile, applications talking to an xterm are forced to reinitialize themselves at restart time by sending two `SIGWINCH` signals to them. And so on. 

Given all of that, it is not surprising that the kernel checkpoint/restart developers see their approach as being a simpler, more robust, and more general solution to the problem. To them, DMTCP looks like a shaky attempt to reimplement a great deal of kernel functionality in user space. Matt Helsley [summarized](<https://lwn.net/Articles/414278/>) it this way: 

Frankly it sounds like we're being asked to pin our hopes on a house of cards -- weird userspace hacks involving extra processes, hodge-podge combinations of ptrace, LD_PRELOAD, signal hijacking, brk hacks, scanning passes in /proc (possibly at numerous times which begs for races), etc.... 

In contrast, kernel-based cr is rather straight forward when you bother to read the patches. It doesn't require using combinations of obscure userspace interfaces to intercept and emulate those very same interfaces. It doesn't add a scattered set of new ABIs. 

Seasoned LWN readers will be shocked to learn that few minds appear to have been changed by this discussion. Most developers seem to agree that some sort of checkpoint/restart functionality would be a useful addition to Linux, but they differ on how it should be done. Some see a kernel-side implementation as the only way to get even close to a full solution to the problem and as the simplest and most maintainable option. Others think that the user-space approach makes more sense, and that, if necessary, a small number of system calls can be added to simplify the implementation. It has the look of the sort of standoff that can keep a project like this out of the kernel indefinitely. 

That said, something interesting may happen here. One thing that became reasonably clear in the discussion is that a complete, performant, and robust checkpoint/restart implementation will almost certainly require components in both kernel and user space. And it seems that the developers behind the two implementations [will be getting together](<https://lwn.net/Articles/414285/>) to talk about the problem in a less public setting. With luck, determination, and enough beer, they might just figure out a way to solve the problem using the best parts of both approaches. That would be a worthy outcome by any measure.
