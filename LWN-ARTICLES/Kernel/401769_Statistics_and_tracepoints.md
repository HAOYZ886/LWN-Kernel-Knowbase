---
title: Statistics and tracepoints
url: https://lwn.net/Articles/401769/
date: "August 24, 2010"
category: "Development model-User-space ABI; Tracing"
author: "By Jonathan Corbet August 24, 2010"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
August 24, 2010

One thing that kernels do is collect statistics. If one wishes to know how many multicast packets have been received, page faults have been incurred, disk reads have been performed, or interrupts have been received, the kernel has the answer. This role is not normally questioned, but, recently, there have been occasional suggestions that the handling of statistics should be changed somewhat. The result is a changing view of how information should be extracted from the kernel - and some interesting ABI questions. 

Back in July, Gleb Natapov submitted [a patch](<https://lwn.net/Articles/401772/>) changing the way paging is handled in KVM-virtualized guests. Included in the patch was the collection of a couple of new statistics on page faults handled in each virtual CPU. More than one month later (virtualization does make things slower), Avi Kivity [reviewed the patch](<https://lwn.net/Articles/401776/>); one of his suggestions was: 

Please don't add more stats, instead add tracepoints which can be converted to stats by userspace. 

Nobody questioned this particular bit of advice. Perhaps that's because virtualization seems boring to a lot of developers. But it is also indicative of a wider trend. 

That trend is, of course, the migration of much kernel data collection and processing to the "perf events" subsystem. It has only been one year since perf showed up in a released kernel, but it has seen sustained development and growth since then. Some developers have been known to suggest that, eventually, the core kernel will be an obscure bit of code that must be kept around in order to make perf run. 

Moving statistics collection to tracepoints brings some obvious advantages. If nobody is paying attention to the statistics, no data is collected and the overhead is nearly zero. When individual events can be captured, their correlation with other events can be investigated, timing can be analyzed, associated data can be captured, etc. So it makes some sense to export the actual events instead of boiling them down to a small set of numbers. 

The down side of using tracepoints to replace counters is that it is no longer possible to query statistics maintained over the lifetime of the system. As Matt Mackall [observed](<http://lkml.indiana.edu/hypermail/linux/kernel/0904.3/01935.html>) over a year ago: 

Tracing is great for looking at changes, but it completely falls down for static system-wide measurements because it would require integrating from time=0 to get a meaningful summation. That's completely useless for taking a measurement on a system that already has an uptime of months. 

Most often, your editor would surmise, administrators and developers are looking for changes in counters and do not need to integrate from time=0. There are times, though, when that information can be useful to have. One could come close by enabling the tracepoints of interest during the bootstrap process and continuously collecting the events, but that can be expensive, especially for high-frequency events. 

There is another important issue which has been raised in the past and which has never really been resolved. Tracepoints are generally seen as debugging aids used mainly by kernel developers. They are often tied into low-level kernel implementation details; changes to the code can often force changes to nearby tracepoints, or make them entirely obsolete. Tracepoints, in other words, are likely to be nearly as volatile as the kernel that they are instrumenting. The kernel changes rapidly, so it stands to reason that the tracepoints would change rapidly as well. 

Needless to say, changing tracepoints will create problems for any user-space utilities which make use of those tracepoints. Thus far, kernel developers have not encouraged widespread use of tracepoints; the kernel still does not have that many of them, and, as noted above, they are mainly debugging tools. If tracepoints are made into a replacement for kernel statistics, though, then the number of user-space tools using tracepoints can only increase. And that will lead to resistance to patches which change those tracepoints and break the tools. 

In other words, tracepoints are becoming part of the user-space ABI. Despite the fact that concerns about the ABI status tracepoints have been raised in the past, this change seems to be coming in through the back door with no real planning. As Linus has [pointed out](<http://lwn.net/Articles/309305/>) in the past, the fact that nobody has designated tracepoints as part of the official ABI or documented them does not really change things. Once an interface has been exposed to user space and come into wider use, it's part of the ABI regardless of the developers' intentions. If user-space tools use tracepoints, kernel developers will have to support those tracepoints indefinitely into the future. 

Past discussions have included [suggestions](<http://lwn.net/Articles/354865/>) for ways to mark tracepoints which are intended to be stable, but no conclusions have resulted. So the situation remains murky. It may well be that things will stay that way until some future kernel change breaks somebody's tools. Then the kernel community will be forced to choose between restoring compatibility for the broken tracepoints or overtly changing its longstanding promise not to break the user-space ABI (too often). It might be better to figure things out before they get to that point.
