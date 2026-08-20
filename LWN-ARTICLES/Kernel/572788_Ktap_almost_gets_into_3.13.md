---
title: Ktap almost gets into 3.13
url: https://lwn.net/Articles/572788/
date: "November 6, 2013"
category: "Development tools-Kernel tracing; Tracing"
author: "By Jonathan Corbet November 6, 2013"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
November 6, 2013

The [ktap project](<http://www.ktap.org/>) surprised almost everybody when it [made its 0.1 release](<https://lwn.net/Articles/551314/>) last May. In short, ktap is a dynamic tracing tool; it works by embedding a Lua interpreter within the kernel and hooking it into the existing tracepoint mechanism. A suitably privileged user can load a script into the kernel; that script can enable tracepoints, boil down the resulting data, and return information back to user space. It thus fills a niche similar to that of SystemTap or DTrace, but with a smaller, simpler, and (for now) less-functional code base. 

Ktap was reasonably well received from the outset, with a number of developers welcoming this functionality. The work was presented at LinuxCon Japan, and the [0.2 release](<https://lwn.net/Articles/561568/>) followed at the end of July. After that, things quieted down for a bit until mid-October, when Greg Kroah-Hartman [announced](<https://plus.google.com/111049168280159033135/posts/Nhzb1FQ8gkz>) (on Google+) that ktap had been merged into the staging tree for the 3.13 release. This release, which already looked to be a relatively feature-heavy development cycle, appeared to be set to acquire a dynamic tracing framework as well. 

It subsequently became clear, though, that some developers hadn't seen Greg's post and weren't happy. On October 24, Ingo Molnar [sent a protest](<https://lwn.net/Articles/572793/>) over the imminent merging of ktap. He added a "Nacked-by" for good measure; Steven Rostedt then followed up with [a NACK of his own](<https://lwn.net/Articles/572794/>). After a Kernel Summit conversation with Ingo, Greg [reverted the changes](<https://lwn.net/Articles/572795/>), removing ktap from the staging tree and, thus, from the queue for the 3.13 kernel. Those who have been waiting for this kind of dynamic tracing functionality in the Linux kernel will have to wait a bit longer. 

#### What happened, and what's next

At a first glance, this incident could be mistaken as an example of a newcomer being rejected by the incumbent developers of the existing tracing functionality. One need not look much further, though, to see that a rather different explanation applies. The rejection of ktap, which looks highly likely to be a temporary affair, is more a story about the right and wrong ways to get new functionality into the kernel. 

The first and foremost mistake is that the patches were never actually posted to the linux-kernel mailing list for review. Ktap developer Jovi Zhangwei has posted his release announcements there, and the code has always been available in a public git repository. But getting review for code in a repository is never easy; that certainly proved to be true in this case. While some developers knew about ktap and what it could do, few had actually looked at the implementation. 

As Ingo pointed out in his objection, the form of the code in the staging tree (a single changeset adding 16,000 lines of code) did not make the review process any easier. Neither did the lack of documentation about the design of ktap. 

The biggest complaint, though, was about the lack of integration with the kernel's existing tracing functionality. Ingo pointed out a couple of areas where that kind of integration might be useful: 

  * The kernel currently has a simple interpreter used to implement filter conditions on tracepoints. The Lua bytecode interpreter offers a rather richer execution environment; it could perhaps be put to work implementing a more comprehensive conditional evaluation mechanism for tracepoints. 

  * Rather than requiring a new set of commands, ktap scripts could be made part of the [perf probe command](<http://linux.die.net/man/1/perf-probe>), tying them more firmly into the existing system. Ingo's goal is clearly to have a single set of comprehensive tracing functionality that can all be exercised without having to know about the separate subsystems that implement it. 

  * Ingo also [requested](<https://lwn.net/Articles/572801/>) the ability to extract ktap scripts from the kernel and turn them back into their source form. That, he said, would improve security by making it possible to examine any scripts running within the kernel. 

In summary, Ingo said, the code looks like it could be useful and make the kernel's instrumentation better: 

Despite my criticism, I'm actually a big proponent of safe kernel probing concepts and this code does have many of the qualities that I always wanted the tracepoint filter code to have in the long run. 

So it appears that ktap should be able to get in eventually, but first Jovi has some work cut out for him to get this code into proper shape for merging. The good news is that [he seems prepared to do that work](<https://lwn.net/Articles/572802/>). The hope is to have a reworked version of ktap reviewed and ready for the 3.14 development cycle; at that point, it should be able to go directly into the core kernel without a stay in the staging tree. This delay is likely to be frustrating for everybody involved, but the end result will almost certainly be an improved, in-kernel scripting facility for dynamic tracing.
