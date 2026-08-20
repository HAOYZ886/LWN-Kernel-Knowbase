---
title: "Per-cgroup TCP buffer limits"
url: https://lwn.net/Articles/470656/
date: "December 6, 2011"
category: "Control groups; Memory management-Control groups"
author: "By Jonathan Corbet December 6, 2011"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
December 6, 2011

Control groups are used for a number of purposes, but the most important role for many is the application of resource limits to groups of processes. The memory usage controller has seen a lot of development effort in recent years, but it still has a number of shortcomings, one of which being that it only applies to user-space memory. Some workloads can cause the use of large amounts of memory in the kernel, leading to overall system memory pressure, but the memory controller is unable to place limits on that memory usage. A first step toward fixing that problem may yet find its way into the 3.3 kernel, but it shows how hard the job can be. 

Glauber Costa's [per-cgroup TCP memory pressure controls patch](<https://lwn.net/Articles/470689/>) starts by adding a bit of infrastructure to the memory controller for the tracking of kernel-space memory use. A pair of new knobs is added: `memory.kmem.limit_in_bytes` controls how much kernel memory a control group is allowed to use, while `memory.independent_kmem_limit` is a boolean controlling whether the kernel limits are to be managed separately from the user-space limits. If that boolean is false, kernel memory usage is simply added to user-space memory usage and the kernel memory limit is ignored. There is also a new `memory.kmem.usage_in_bytes` value that can be read to see what the current memory usage is. 

This infrastructure is nice, but there is one other little difficulty: there are thousands of places in the kernel where memory is allocated and none of them are instrumented in a way that allows those allocations to be charged to a specific control group. An implementation of that accounting on a kernel-wide basis may never happen; it certainly will not happen at the outset. So developers seeking to add this functionality need to focus on specific data structures. Some past work has been done to apply limits to the dentry cache, but Glauber chose a different limit: buffers used in the implementation of the TCP network protocol. 

Those buffers hold packet data as it passes through a socket; in some situations they can become quite large, so they are a logical place to try to apply limits. What is even better is that the networking stack already has logic to place limits on buffer sizes when memory is tight. In current kernels, if the system as a whole is seen to be under memory pressure, the networking code will do its best to help. Its responses can include decisions not to increase the size of TCP windows, to drop packets, or to refuse to expand buffers for outgoing data. 

Given that this infrastructure was in place, all that Glauber needed to do was to enhance the stack's idea of what constitutes "memory pressure." That means instrumenting the places that allocate and free buffer memory to charge those allocations to the owning control group. Then, "memory pressure" becomes a combination of the previous global value and the current control group's status with regard to its kernel memory limit. If that limit has been exceeded, the stack will behave (with regard to that control group's sockets) as if the system as a whole were under memory pressure, even if the global state is fine. 

The first patches did exactly that and seemed to be on their way toward inclusion in the 3.2 merge window. It ran into a bit of a snag when the core networking developers took a look at it, though. Networking maintainer David Miller [rejected the patch](<https://lwn.net/Articles/470660/>) out of hand, complaining about the overhead it adds to the networking fast paths even in cases where the feature is not in use. He [added](<https://lwn.net/Articles/470661/>): 

People keep asking every few releases "where the heck has my performance gone" and it's because of creeping features like this. This socket cgroup feature is a prime example of where that kind of stuff comes from. 

I really get irritated when people go "oh, it's just one indirect function call" and "oh, it's just one more pointer in struct sock" We work really hard to _remove_ elements from structures and make them smaller, and to remove expensive operations from the fast paths. 

There are a lot of important memory-allocation sites in the kernel that can be found in relatively hot execution paths; the networking developers may obsess about adding overhead to those paths, but they are certainly not alone in that concern. So a solution had to be found that did not impose that overhead on systems where the feature is not in use. 

The words "in use" are important here as well. If kernel memory usage limits are useful, distributors will want to enable them in the kernels they ship. But it may still be that most users will not actually make use of that feature. So it is not sufficient to remove its overhead only in cases where it has been configured out of the kernel. A feature like this really needs to avoid imposing costs when it is not in use, even when it is configured into the running kernel. 

Glauber had to make some significant changes to the patch set to meet these requirements. TCP buffer limits are now managed separately from kernel limits as a whole; there is a new knob (`kmem.tcp.limit_in_bytes`) to control that limit. All of the fast-path code is now contained within a [static branch](<https://lwn.net/Articles/436041/>), meaning that, when the code is not enabled, it can be skipped over with almost no overhead at all. That static branch is only turned on when a TCP buffer limit is established for a non-root control group. So, as required, it should not have significant effects in kernels where the feature is configured in but not being used. 

As of this writing, no decision to merge these patches for 3.3 has been announced, but the number of criticisms has been steadily dropping. So it may well get into the mainline in the next merge window. But Glauber's experience shows how hard it will be to add more kernel memory accounting; the requirement that its impact be imperceptible will lead to more work and trickier code. For that reason alone, accounting for all kernel memory use seems unlikely to ever happen. Developers will focus on the few cases where the application of limits can bring about significant changes in behavior and not worry about the rest.
