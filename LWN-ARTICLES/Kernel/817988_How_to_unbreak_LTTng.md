---
title: How to unbreak LTTng
url: https://lwn.net/Articles/817988/
date: "April 20, 2020"
category: "Development tools-Kernel tracing; Modules-Exported symbols; Tracing"
author: "By Jonathan Corbet April 20, 2020"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
April 20, 2020

Back in February, the kernel community [discussed](<https://lwn.net/Articles/813350/>) the removal of a couple of functions that could be used by loadable modules to gain access to symbols (functions and data structures) that were not meant to be available to them. That change was [merged](<https://git.kernel.org/linus/0bd476e6c671>) during the 5.7 merge window. This change will break a number of external modules that depended on the removed functions; since many of those modules are proprietary, this fact does not cause a great deal of anguish in the kernel community. But there are a few out-of-tree modules with GPL-compatible licenses that are also affected by this change; one of those is [LTTng](<https://lttng.org/>). Fixing LTTng may not be entirely straightforward. 

LTTng is a tracing subsystem; to carry out that sort of task, it must be able to hook into the kernel in a number of fairly deep places. It is unsurprising that LTTng was accessing parts of the kernel that are not deemed suitable for export to modules in general. Losing access to `kallsyms_on_each_symbol()` deprived LTTng of the ability to find those addresses, thus breaking much of its functionality. That is not welcome news to those who work on — or use — LTTng. 

LTTng developer Mathieu Desnoyers has responded to this change with [a patch series](<https://lwn.net/ml/linux-kernel/20200409193543.18115-1-mathieu.desnoyers@efficios.com/>) exporting a number of new symbols; with those available, LTTng can do what it needs to do without using the rather more general `kallsyms_on_each_symbol()` function. For example, LTTng needs access to [`stack_trace_save_user()`](<https://elixir.bootlin.com/linux/v5.6.5/source/kernel/stacktrace.c#L215>) to be able to save user-space stack traces. It also needs access to functions like [`task_prio()`](<https://elixir.bootlin.com/linux/v5.6.5/source/kernel/sched/core.c#L4605>), [`disk_name()`](<https://elixir.bootlin.com/linux/v5.7-rc2/source/block/genhd.c#L73>), and [`get_pfn_blocks_mask()`](<https://elixir.bootlin.com/linux/v5.6.5/source/mm/page_alloc.c#L470>). LTTng obtains kernel information from tracepoints as well, of course, and that usage will increase as tracepoints replace some of the direct internal accesses that were used before. The patch set raises the number of arguments that can be passed to a BPF program from a tracepoint to an eye-opening 13 (to allow more information to be passed out via a specific tracepoint), but that change may prove to be unnecessary in the end. 

Anybody who has watched the kernel community for any period of time can probably guess what sort of reception this patch series received. Christoph Hellwig was [characteristically blunt](<https://lwn.net/ml/linux-kernel/20200412083151.GA17521@infradead.org/>): ""Which part of every added export needs an in-tree user did you not get?"" The kernel community as a whole is strongly resistant to the idea of adding any sort of support for code that is outside of the kernel repository. Much of that resistance comes from a dislike for proprietary kernel modules in general, but there is a bit more to it than that. 

LTTng, being free software, should not be affected by any antipathy for proprietary kernel code. But, as Greg Kroah-Hartman [explained](<https://lwn.net/ml/linux-kernel/20200411064503.GA2576039@kroah.com/>), there are still reasons to avoid adding support for free, out-of-tree modules. Once those modules are supported in some way, they add constraints to what kernel developers can do. Internal kernel interfaces can be changed as needed; since all of the users of those interfaces are present in the same code base, they can be changed at the same time. If external modules have to be supported, though, it becomes harder to make such changes, since the users cannot be changed to match. Indeed, it becomes difficult to even know when a change might cause problems elsewhere. 

Thus, Kroah-Hartman said: 

We can't do anything for out-of-tree modules as they suddenly become "higher priority" than in-tree code if you have to not do specific changes or extra work for them. Which is not fair at all to the in-tree code developers at all. 

This all suggests that there is not much of a path forward for LTTng. It is unable to function without access to kernel internals, and that access is being expressly denied. 

There is, of course, one other option that was first [raised](<https://lwn.net/ml/linux-kernel/20200410114444.73be173a@gandalf.local.home/>) by Steve Rostedt: ""I guess we should be open to allowing LTTng modules in the kernel as well"". If LTTng were actually a part of the mainline kernel, there would no longer be problems with giving access to the resources that it needs. 

This is not a new idea. Numerous attempts have been made to get the LTTng code into the mainline kernel, without success. In the early days, before the kernel had any sort of tracing capability at all, adding that feature was a hard sell. Kernel developers now are heavily dependent on tracing for their own work and would strongly resist any attempt to take that capability away, but it was not that long ago that many of the same developers were unconvinced that tracing was needed at all. During that time, getting _any_ tracing features into the kernel was not easy. 

Over time, some low-level LTTng code found its way in, but LTTng as a whole has not followed. More recently, in 2011, LTTng was [brought into the staging tree](<https://lwn.net/Articles/471973/>) by Kroah-Hartman as a first step toward merging it. That move brought about a great deal of hostility, some of which seems familiar; a rather lengthy thread was set off by [an attempt to export `task_prio()`](<https://lore.kernel.org/lkml/1322775683-8741-10-git-send-email-mathieu.desnoyers@efficios.com/>), for example. In the end, LTTng was pushed back out of the staging tree — as it was before and has been ever since. 

So LTTng would appear to be in a difficult position: unable to function outside of the kernel, and unable to be merged. Leaving LTTng broken would cause serious harm to a lot of users, though, and seems unlikely to advance the cause of Linux or free software in general. So perhaps the time has come for something to give. If a handful of symbols truly cannot be exported for this subsystem, perhaps some space could be found in the mainline for a widely used tracing subsystem, even if it somehow duplicates some of the functionality that is already there.
