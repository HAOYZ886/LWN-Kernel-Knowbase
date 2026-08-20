---
title: Dynamic function tracing events
url: https://lwn.net/Articles/747256/
date: "February 15, 2018"
category: "Tracing-ABI issues"
author: "By Jonathan Corbet February 15, 2018"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
February 15, 2018

For as long as the kernel has included tracepoints, developers have argued over whether those tracepoints are part of the kernel's ABI. Tracepoint changes have had to be reverted in the past because they broke existing user-space programs that had come to depend on them; meanwhile, fears of setting internal code in stone have made it difficult to add tracepoints to a number of kernel subsystems. Now, a new tracing functionality is being proposed as a way to circumvent all of those problems. 

Whether tracepoints are part of the kernel ABI is not an insignificant issue. The kernel's ABI promise states that working programs will not be broken by updated kernels. It has become clear in the past that this promise extends to tracepoints, most notably in 2011 when [a tracepoint change broke powertop](<https://lwn.net/Articles/442113/>) and had to be reverted. Some kernel maintainers prohibit or severely restrict the addition of tracepoints to their subsystems out of fear that a similar thing could happen to them. As a result, the kernel lacks tracepoints that users would find useful. 

This topic has found its way onto the agenda at a number of meetings, including the [2017 Maintainers Summit](<https://lwn.net/Articles/737530/>). At that time, a clever idea had been raised: rather than place tracepoints in sensitive locations, developers could just put markers that would have to be explicitly connected to and converted to tracepoints at run time. By adding some hoops to be jumped through, it was hoped, this new mechanism would not create any new ABI guarantees. Then things went quiet for a couple of months. 

Recently, though, tracing maintainer Steve Rostedt surfaced with [a variation on that proposal](<https://lwn.net/Articles/747262/>) that he is calling "dynamically created function-based events". The details have changed, but the basic nature of the ABI dodge remains the same. The key detail that is different comes from the observation that the kernel already has a form of marker in place that the tracing code can make use of. 

Kernel code is usually compiled with options that are normally used for code profiling. As a result, each function begins with a call to a function called `mcount()` (or `__fentry()__` when a newer compiler is in use). When a user-space program is being profiled, `mcount()` tracks calls to each function and the time spent there. The kernel, though, replaces `mcount()` with its own version that supports features like function tracing. Most of the time, the `mcount()` calls are patched out entirely, but they can be enabled at run time when there is a need to trace calls into a specific function. 

There are other possible uses for this function-entry hook. Rostedt's patch uses it to enable the creation of a tracepoint at the beginning of any kernel function at run time. With the `tracefs` control filesystem mounted, a new tracepoint can be created with a command like: 

```
echo 'SyS_openat(int dfd, string path, x32 flags, x16 mode)' \
        	 > /sys/kernel/tracing/function_events
```

This command requests the creation of a tracepoint at the entry to `SyS_openat()`, the kernel's implementation of the `openat()` system call. Four values will be reported from the tracepoint: the directory file descriptor (`dfd`), the given pathname (`path`), and the `flags` and `mode` arguments. This tracepoint will show up under `events/functions` and will look like any other tracepoint in the kernel. It can be queried, enabled, and disabled in the usual ways. Interestingly, `path` in this case points into user space, but the tracing system properly fetches and prints the data anyway. 

There is evidently some work yet to be done: ""I need to rewrite the function graph tracer, and be able to add dynamic events on function return."". But the core is seemingly in place and working. That leaves an important question, though: will it be enough to avoid creating a new set of ABI-guaranteed interfaces to the kernel? Mathieu Desnoyers [worried](<https://lwn.net/Articles/747273/>) that it might not: 

Having those tools hook on function names/arguments will not make this magically go away. As soon as kernel code changes, widely used trace analysis tools will start breaking left and right, and we will be back to square one. Only this time, it's the internal function signature which will have become an ABI. 

Linus Torvalds [disagreed](<https://lwn.net/Articles/747275/>) with this worry, though. The extra step required to hook into the kernel implies a different view of the status of that hook: 

Everybody *understands* that this is like a debugger: if you have a gdb script that shows some information, and then you go around and change the source code, then *obviously* you'll have to change your debugger script too. You don't keep the source code static just to make your gdb script happy., That would be silly. 

In contrast, the explicit tracepoints really made people believe that they have some long-term meaning. 

If reality matches this view, then the new dynamic tracepoint mechanism could go a long way toward defusing the ABI issues. The number of new tracepoints being added to the kernel would be likely to drop, as developers could simply use the dynamic variety instead. When tracepoints are added in the future, it is relatively likely that they will be designed to support some sort of system-management tool and, thus, be viewed as a part of the ABI from the outset. 

That assumes that this patch series is eventually merged, of course. There was [some dissent](<https://lwn.net/Articles/747375/>) from Alexei Starovoitov, who complained that the new interface adds little to what can already be had with kprobes. He also disliked the text-oriented interface, suggesting (unsurprisingly) that BPF should be used instead to extract specific bits of data from the kernel. Rostedt [noted](<https://lwn.net/Articles/747376/>), though, that many developers are put off by the complexity of getting started with BPF and would prefer something simpler. 

Rostedt said that he thought the interface would be useful, but that he would not continue its development if others did not agree: ""If others think this would be helpful, I would ask them to speak up now"". Thus far, few people have spoken. If the dynamic function tracing mechanism is indeed something that other developers would like to have available, they might want to make their feelings known.
