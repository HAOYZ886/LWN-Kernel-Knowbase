---
title: "Code tagging and memory-allocation profiling"
url: https://lwn.net/Articles/932402/
date: "May 31, 2023"
category: "Build system; Development tools-Kernel tracing"
author: "By Jonathan Corbet May 31, 2023 LSFMM+BPF"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
May 31, 2023

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2023>)

The [code-tagging mechanism](<https://lwn.net/Articles/906660/>) proposed last year by Suren Baghdasaryan and Kent Overstreet has been the subject of a number of (sometimes tense) discussions. That conversation came to the memory-management track at the [2023 Linux Storage, Filesystem, Memory-Management and BPF Summit](<https://lwn.net/Articles/lsfmmbpf2023>), where its developers (Baghdasaryan attending in-person and Overstreet remotely) tried to convince the attendees that its benefits justify its cost. 

Baghdasaryan started by saying that the use case for code tagging was memory-allocation profiling — accounting for all kernel allocations in order to monitor usage and find leaks. Any solution in this area, he said, must have a low-enough overhead that it can be used on production systems. It must also produce enough information to be useful; achieving both objectives can be hard. The proposal is a two-level solution, providing a high-level view with low overhead and the ability to get a detailed view for specific call sites. 

[![\[Suren
Baghdasaryan\]](https://static.lwn.net/images/conf/2023/lsfmm/SurenBaghdasaryan-sm.png)](<https://lwn.net/Articles/932403/>) The [proposed implementation](<https://lwn.net/ml/linux-kernel/20230501165450.15352-1-surenb@google.com/>) uses code tagging, which works by injecting a structure into a specific code location to identify that location. Application-specific fields can be attached to these tags; they can be used for allocation profiling, fault injection, latency tracking, and more. A special macro is used to put the structures into a separate executable section, but some inline code is also needed to associate the structure and the call. 

The performance overhead of this mechanism, he said, is 36% for slab allocations and 26% for page allocations. That may seem high, but he argued that developers should consider that the code in question is highly optimized. Enabling memory control groups add ten times the overhead that allocation profiling does. The memory overhead depends on the number of CPUs in the system; it was about 0.3% of memory on an eight-core Android device with about 10,000 allocation call sites. 

The prepared part of the discussion ended with Baghdasaryan asking the developers in the room if they would use this tool. 

Steve Rostedt said that he had been asked whether it might be possible to implement the tagging more efficiently with static calls, which can be patched in or out at run time. The proposed code-tagging feature, he said, is adding macros around other macros, and must be explicitly added for every interface to be profiled. It injects code into every call site, which will lead to poorer locality and worse performance. He suggested that an alternative could possibly be created using `objtool`; it could find all of the call locations for a function of interest and create a trampoline for each in a separate section. That trampoline would log the data of interest, then call the target function. In normal operation, the trampoline would be unused; to turn the monitoring on, the call sites would be patched at run time to, instead, jump to the trampoline. 

Overstreet responded that this solution replaced magic macros with something even more magic; Rostedt answered that this is how ftrace and a number of other functionalities work now. It is well-tested and can be expected to work. 

Overstreet said that there is value to placing annotations in the source code; it allows the programmer to choose which functions are annotated and serves as a sort of documentation. The code tags can also be used for fault injection, allowing, for example, the writing of a test that would exercise the error handling code at each call site. Rostedt answered that all of this could be done in the trampoline as well; there could even be a BPF hook to make it more flexible. 

John Hubbard said that 36% is a high overhead; the ability to turn that off would be an important feature. He said that he prefers the approach taken by tools like [bpftrace](<https://github.com/iovisor/bpftrace>), which attaches probes at run time. Overstreet said that one can't enable counters at run time and expect them to have any meaning; Baghdasaryan added that, if the counters are not enabled at boot, the system would see — and potentially be confused by — memory being freed that had been allocated before monitoring was enabled. Rostedt said that this problem can be addressed by booting the system with monitoring enabled, then turning it off once the needed data had been collected. 

Overstreet complained that the trampoline idea would impose a greater overhead when it was turned on; Rostedt disagreed, pointing out that the trampoline would be entered with a direct jump, so no extra function calls are added. There followed an extended and sometimes heated discussion on the details that, in your editor's opinion, is not really worth reproducing here. 

Michal Hocko brought the discussion to a close by noting that those details were not the important issue at hand; the developers needed to consider the overall design of any instrumentation mechanism and decide which would work best. Overstreet did not help his case by saying, at this point, that he would like to add some counters to `struct page` for more data collection. That idea was summarily rejected by the group. 

The session ended with nobody, seemingly, satisfied with how it went. This seems like a conversation that is destined to continue for some time.
