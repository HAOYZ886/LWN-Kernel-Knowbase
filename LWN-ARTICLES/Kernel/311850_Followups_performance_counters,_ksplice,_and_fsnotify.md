---
title: "Followups: performance counters, ksplice, and fsnotify"
url: https://lwn.net/Articles/311850/
date: "December 17, 2008"
category: "Ksplice; Live patching; Performance monitoring; Security-Security technologies"
author: "By Jonathan Corbet December 17, 2008"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
December 17, 2008

There's been progress in a few areas which LWN has covered in the past. Here's a quick followup on where things stand now. 

### Performance monitors

In [last week's episode](<http://lwn.net/Articles/310260/>), a new, out-of-the-blue performance monitoring patch had stirred up discussion and a certain amount of opposition. The simplicity of the new approach by Ingo Molnar and Thomas Gleixner had some appeal, but it is far from clear that this approach is sufficiently powerful to meet the needs of the wider performance monitoring community. 

Since then, [version 3](<http://lwn.net/Articles/310888/>) and [version 4](<http://lwn.net/Articles/311388/>) of the patch have been posted. A look at the changelogs shows that work on this code is progressing quickly. A number of change have been made, including: 

  * The addition of virtual performance counters for tracking clock time, page faults, context switches, and CPU migrations. 

  * A new "performance counter group" functionality. This feature is meant to address criticism that the original interface would not allow multiple counters to be read simultaneously, making it hard to correlate different counter values. Counters can now be associated into multiple groups which allow them to be manipulated as a unit. There's also a new mechanism allowing all counters to be turned on or off with a single system call. 

  * The system call interface has been reworked; see the version 3 announcement for description of the new API. 

  * The kerneltop utility has been enhanced to work with performance counter groups. 

  * "Performance counter inheritance" is now supported; essentially, this allows a performance monitoring utility to follow a process through a `fork()` and monitor the child process(es) as well. 

  * The new "timec" utility runs a process under performance monitoring, outputting a whole set of statistics on how the process ran. 

There are still concerns about this new approach to performance monitoring, naturally. Developers worry that users may not be able to get the information they need, and it still seems like it may be necessary to put a huge amount of hardware-specific programming information into the kernel. But, to your editor's eye, this patch set also seems to be gaining a bit of the sense of inevitability which usually attaches itself to patches from Ingo and company. It will probably be some time, though, before a decision is made here. 

### Ksplice

In November, we [looked at](<http://lwn.net/Articles/308409/>) a new version of the Ksplice code, which allows patches to be put into a running kernel. The Ksplice developers would like to see their work go into the mainline, so they recently poked Andrew Morton to see what the status was. His [response](<https://lwn.net/Articles/311856/>) was: 

It's quite a lot of tricky code, and fairly high maintenance, I expect. 

I'd have _thought_ that distros and their high-end customers would be interested in it, but I haven't noticed anything from them. Not that this means much - our processes for gathering this sort of information are rudimentary at best. 

The response on the list, such as it was, indicated that the distributors are, in fact, not greatly interested in this feature. Dave Jones [commented](<https://lwn.net/Articles/311861/>): 

It's a neat hack, but the idea of it being used by even a small percentage of our users gives me the creeps.... 

If distros can't get security updates out in a reasonable time, fix the process instead of adding mechanism that does an end-run around it. Which just leaves the "we can't afford downtime" argument, which leads me to question how well reviewed runtime patches are. Having seen some of the non-ksplice runtime patches that appear in the wake of a new security hole, I can't say I have a lot of faith. 

The Ksplice developers [agree](<https://lwn.net/Articles/311863/>) that the writing of custom code to fit patches into a running kernel is a scary proposition; that is why, they say, they've gone out of their way to make such code unnecessary most of the time. 

This discussion leaves Ksplice in a bit of a difficult position; in the absence of clear demand, the kernel developers are unlikely to be willing to merge a patch of this nature. If this is a feature that users really want, they should probably be communicating that fact to their distributors, who can then consider supporting it and working to get it into the mainline. 

### fsnotify

The file scanning mechanism known as [TALPA](<http://lwn.net/Articles/292872/>) got off to a rough start with the kernel development community. Many developers have a dim view of the malware scanning industry in general, and they did not like the implementation that was posted. It is clear, though, that the desire for this kind of functionality is not going away. So developer Eric Paris has been working toward an implementation which will pass review. 

His latest attempt can be seen in the form of the [fsnotify](<http://lwn.net/Articles/311350/>) patch set. This code does not, itself, support the malware scanning functionality, but, says Eric, ""you better know it's coming."" What it does, instead, is to create a new, low-level notification mechanism for filesystem events. 

At a first look, that may seem like an even more problematic approach than was taken before. Linux already has two separate file event notifiers: dnotify and inotify. Kernel developers tend to express their dissatisfaction with those interfaces, but there has not been a whole lot of outcry for somebody to add a third alternative. So why would fsnotify make sense? 

Eric's idea seems to be to make something that so clearly improves the kernel that people will lose the will to complain about the malware scanning functionality. So fsnotify has been written - employing a lot of input from filesystem developers - to be a better-thought-out, more supportable notification subsystem. Then the existing dnotify and inotify code is ripped out and reimplemented on top of fsnotify. The end result is that the impact on the rest of the VFS code is actually reduced; there is now only one set of notifier calls where, previously, there were two. And, despite that, the notification mechanism has become more general, being able to support functionality which was not there in the past. 

And, to top it off, Eric has managed to make the size of the in-core `inode` structure smaller. Given that there can be thousands of those structures in a running system, even a small size reduction in their size can make a big difference. So, claims Eric, ""That's right, my code is smaller and faster. Eat that."" 

What this code needs now is detailed review from the core VFS developers. Those developers tend to be a highly-contended resource, so it's not clear when they will be able to take a close look at fsnotify. But, sooner or later, it seems likely that this feature will find its way into the mainline.
