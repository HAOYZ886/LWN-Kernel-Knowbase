---
title: Replacing ptrace()
url: https://lwn.net/Articles/371501/
date: "January 27, 2010"
category: Utrace; ptrace
author: "By Jonathan Corbet January 27, 2010"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
January 27, 2010

Much of the POSIX system call interface is known for the elegance and simplicity of its design; that is what has enabled this API to endure and thrive for decades. The `ptrace()` system call has no such reputation. One of the many motivations behind the development of the utrace layer (see [the accompanying article](<http://lwn.net/Articles/371210/>)) was first to clean up the implementation of `ptrace()`, but then to enable it to be replaced entirely. Subsequent discussion shows that this is a distant hope, though, and that we will be struck with `ptrace()` for a long time. 

The purpose of `ptrace()` is to allow one process to monitor and modify the state of another. It exists to support interactive debuggers and related utilities like `strace`, but other users exist as well. User-mode Linux uses `ptrace()` for its internal management, and there are various sandboxing schemes which use it. In general, users are able to get `ptrace()` to do what they want, but they rarely come away pleased with the experience. 

What are the problems with `ptrace()`? Whenever system calls have to work with extended state within the kernel, the preferred mechanism for referring to that state in user space is the file descriptor. With file descriptors, many of the existing system calls do natural things, and well-defined mechanisms exist for event multiplexing. But `ptrace()` doesn't use file descriptors; it depends, instead, on a rather more arcane mechanism. A process to be traced is removed from its normal place in the process tree; the process doing the tracing becomes its new parent. In other words, `ptrace()` sets up a sort of temporary foster home for children under scrutiny. The new parent can then learn about events in the child through the `wait()` system call. 

This API is hard to fit into normal application event loops. It also implies that any given process can be traced by only one other process at any given time. This may not seem like a problem - how often does one want to run two debuggers on a process? - but it does get in the way. Developers working on debugging tools and users wanting to trace a sandboxed process are two types of users who cannot do what they want with `ptrace()`. It is also defined as a complex, multiplexer call (see [the man page](<http://www.kernel.org/doc/man-pages/online/pages/man2/ptrace.2.html>) for details) which is hard to understand and hard to use efficiently. 

Finally, `ptrace()` is hard to implement correctly and consistently. As a result, there has been a long history of obnoxious bugs associated with it, and user-space code which uses `ptrace()` tends to become encrusted with non-portable workarounds. It is, in summary, not surprising that there is interest in creating a replacement. Oleg Nesterov [expressed things](<https://lwn.net/Articles/371505/>) succinctly: ""I must admit that personally I think the current ptrace api is unfixable, we need the new one in the long term."" 

Getting to the new one could be hard, though. The first problem is that `ptrace()` is a standard function which is part of the kernel ABI. As long as users exist, it really cannot be removed from the kernel. So a `ptrace()` replacement will not improve life for the kernel development community anytime in the near future; indeed, it will make it harder, since there will be two tracing interfaces to support instead of one. Duplicating functionality in this way can be done when the need is strong enough, but it's not something that the community will rush into without a great deal of thought. 

Maintaining `ptrace()` as a compatibility interface might be acceptable if it were clearly a temporary thing with a clear possibility of removal in the future, and if there were clear advantages of doing so. But it's not entirely clear where the advantages are. For example, Kyle Moffett [said](<https://lwn.net/Articles/371507/>): 

The killer app for this will be the ability to delete thousands of lines of code from GDB, strace, and all the various other tools that have to painfully work around the major interface gotchas of ptrace(), while at the same time making their handling of complex processes much more robust. 

There are a couple of related problems with this idea, starting with the fact that tools like GDB don't just run on Linux systems with shiny new kernels. They need to work on older kernels indefinitely, not to mention on all those other platforms which lack the good taste to implement every new system call created for Linux. So those "thousands of lines" (and it really is that much code) will not be going anywhere; the GDB developers will have to maintain them forever - or something fairly close to that. 

So for GDB, too, a new tracing API would represent an increase in the maintenance load - if they use it. But the fact of the matter is that special, Linux-only interfaces tend to have very limited uptake. As [expressed](<https://lwn.net/Articles/371509/>) by Ingo Molnar: 

Special Linux system calls have a checkered past, they tend to not be used by much anything, and thus they tend to be a breeding ground of both bugs, maintenance complexity and security problems. Lack of attention is never good. 

That said, Tom Tromey has [indicated](<https://lwn.net/Articles/371510/>) that GDB might use a new API if there were clear advantages to doing so: 

Nevertheless, if the Linux kernel were to present a new user-space API, and if it had an advantage over ptrace, then we would port GDB to use it. There are other platforms where, IIRC, we now use some /proc thing instead of ptrace. 

Tom goes on to list a few features that he would like to see in a replacement for `ptrace()`. That highlights one final obstacle to any kind of new API: no such thing has been implemented or even specified by anybody. The creation of a new system call - especially for a task as complicated as tracing - is not an easy thing to do. Without a great deal of care, we risk creating yet another substandard API with its own warts which must be maintained forever. So a proposed replacement would have to get through an extended process of criticism, argument, and opposition, and it would have to demonstrate some real users \- a GDB port, for example. That, alone, ensures that any `ptrace()` replacement will be years away. 

So it's not surprising that justifying utrace as a means to replace `ptrace()` is not working very well, and it's not surprising that developers are talking about possible ways of extending `ptrace()` instead. Playing with the `ptrace()` API is not without its risks - code which uses it tends to be a bit of a house of cards which can be broken by subtle changes in semantics. But it may still be an easier route to moderately more sane and usable tracing in the relatively near future.
