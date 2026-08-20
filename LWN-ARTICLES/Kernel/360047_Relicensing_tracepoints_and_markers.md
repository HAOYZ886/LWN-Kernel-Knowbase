---
title: Relicensing tracepoints and markers
url: https://lwn.net/Articles/360047/
date: "November 4, 2009"
category: Copyright issues
author: "By Jake Edge November 4, 2009"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jake Edge**  
November 4, 2009

Sharing code where it is possible is normally considered a good thing, but there are some limits to what can be shared. One of the limiting factors is often license compatibility; GPL code, in particular, often cannot be combined with code under other licenses and then distributed. The kernel is licensed under the GPL, but, since it's rare that anyone wants to combine its code with user-space applications, license incompatibilities have not been much of a problem. 

There is, however, some kernel tracing infrastructure that could be shared with user-space tracing applications--likely benefiting both--if those parts of the kernel were available under more permissive licenses. Mathieu Desnoyers, who has developed much of that infrastructure, has set out to [try to relicense](<https://lwn.net/Articles/360265/>) some fairly small portions of the kernel under dual licenses, so that the code can be shared. 

Essentially, Desnoyers would like to be able to use the kernel tracing infrastructure in the [Linux Trace Toolkit Next Generation](<http://ltt.polymtl.ca/?q=node/7>) (LTTng) user-space tracer (UST). He [describes](<https://lwn.net/Articles/360267/>) the need as follows: 

The intent is to allow the tracer code developed both on the kernel-side as part of Ftrace and LTTng and on the userspace side within UST to be shared when appropriate. As a result, we can consider userland-only solutions to user-space tracing without rewriting all the kernel tracing infrastructure from scratch. 

All of the files are currently licensed under the GPLv2, but Desnoyers would like to see the C files available under a dual GPLv2/LGPLv2.1 license, and the header files under a dual GPLv2/BSD license. In order to do that--at least under the most inclusive interpretation of copyright--he must get permission for the relicensing from each contributor to those files. His message to linux-kernel listed the few remaining contributors that he had not yet heard from. 

The files of interest are `kernel/marker.c` and `kernel/tracepoint.c`, along with the corresponding header files in `include/linux`. For 2.6.32, kernel markers have been removed, with all users converted over to use trace events, but `marker.[ch]` are still used by UST. The idea is that the C files could be turned into a user-space library that could be dynamically linked to applications that required it, while the header files (with an even more permissive license) could be used to add static tracepoints to any application, proprietary or free. 

For the most part, the relicensing has been met with approval from the developers who responded, with several saying that they didn't think their contributions warranted requiring their approval, but they gave it anyway. Steven Rostedt ran the C file relicensing by Red Hat's legal department and was granted permission for all of the Red Hat contributions to be dual licensed under the GPLv2/LGPLv2.1. The header file GPLv2/BSD dual licensing is still pending with Red Hat, according to Desnoyers. 

There are still a few developers who have not responded, but their contributions are quite small, and could be rewritten rather easily if necessary. A bigger stumbling block may be [opposition](<https://lwn.net/Articles/360266/>) from Ingo Molnar, who seems to consider the relicensing process to be legally dubious: ""the legality of such relicensing is questionable as that code was never developed outside of the kernel but as part of the kernel"". In addition, he has technical concerns: 

But i also disagree with it on a technical level: code duplication is _bad_. Why does the code have to be duplicated in user-space like that? I'd like Linux tracing code to be in the kernel repo. Why isn't this done properly, as part of the kernel project - to make sure it all stays in sync? 

So for those two grounds i cannot give my permission for this relicensing, sorry. 

Whether Molnar's permission is actually required is something of an open question as his employer (Red Hat) has already given permission for his work to be relicensed. But, if there are serious concerns that lead to a "nack" from him on the relicensing patch, things get rather murky. It may be that there is a disconnect between Desnoyers and Molnar such that Desnoyers's intent is not clear. As Pierre-Marc Fournier [points out](<https://lwn.net/Articles/360368/>), not relicensing the code leads to code duplication as well: 

So the GPL code will have to be rewritten. And this will result in the exact same drawbacks you are trying to avoid by being against dual-licensing. The goal of dual-licensing is to make it possible to keep the code in sync between kernel and userspace, not the opposite! 

Essentially, Desnoyers wants user-space applications to be able to contain tracepoints that are based on the same code that is used now in the kernel. Those applications may be under a variety of free or proprietary licenses, but the tracepoints are just a static instrumentation technique that could be shared. As Rostedt [puts it](<https://lwn.net/Articles/360403/>): 

But what I think is trying to be done here is to use the same types of MACROS that we have in the kernel to do tracing in userspace. That a userspace program can add their own "TRACE_EVENT" and that the headers there will create a tracepoint for them the same way we currently do in the kernel. 

Molnar has gone quiet on the topic, as has the thread, but the idea, overall, seems reasonable. While it does expose a kernel interface to user space, it doesn't tie the kernel to any ABI/API for the future. If the kernel needs to change, either the user-space libraries will change right along with it, or there will be a fork. Given that the players involved work on both the kernel and user-space sides of the problem, that seems somewhat unlikely to happen, but it certainly doesn't seem like that split need happen now.
