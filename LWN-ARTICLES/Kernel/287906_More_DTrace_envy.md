---
title: More DTrace envy
url: https://lwn.net/Articles/287906/
date: "July 2, 2008"
category: SystemTap; Tracing
author: "By Jake Edge July 2, 2008"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jake Edge**  
July 2, 2008

Nearly a year ago, we [looked at the status of SystemTap](<http://lwn.net/Articles/244536/>) in the context of Sun's much-hyped DTrace tool. Since that time there has been progress, but the basic problem still remains: Linux does not have a good, ready-to-run answer to those wanting the equivalent functionality of DTrace. Due to an apparent disconnect between the developers of SystemTap and the kernel hackers, tracing for the Linux kernel--never mind user space programs--is not up to the competition. 

Both [SystemTap](<http://sourceware.org/systemtap/>) and [DTrace](<http://opensolaris.org/os/community/dtrace/>) are tools meant to help administrators track down performance and other problems on production systems by instrumenting the kernel. Because SystemTap has not matured to the point of easy usability, DTrace is often seen as a prime differentiator between Linux and Solaris. In a posting to the ksummit-2008-discuss mailing list--where [Kernel Summit](<https://www.linuxfoundation.org/events/kernel>) topics are considered--Matthew Wilcox [brought up the subject](<https://lists.linux-foundation.org/pipermail/ksummit-2008-discuss/2008-June/000149.html>) based on his experience at a recent PostgreSQL conference: 

There was a lot of buzz around DTrace. Sun and a couple of other companies have put DTrace hooks into postgres, so they now have some really useful canned queries. If you're running Solaris or MacOS, of course. 

So there was a lot of talk about switching away from Linux. This can't possibly be a good thing for us. I don't personally know what the state of our competing projects are, but clearly they haven't got their hooks into postgres ... at least not upstream. 

Typically Linux has been in the forefront of interesting new technologies for free operating systems. When Sun opened up Solaris, though, a few features jumped ahead of their Linux counterparts, in particular the ZFS filesystem and DTrace. SystemTap is supposed to provide the tracing functionality while [Btrfs](<http://btrfs.wiki.kernel.org/index.php/Main_Page>) is the leading candidate for a "next generation" filesystem. But, so far, SystemTap has not lived up to its potential. 

There are a few reasons for disappointment with SystemTap, some of which were [pointed out](<https://lists.linux-foundation.org/pipermail/ksummit-2008-discuss/2008-June/000153.html>) by James Bottomley: 

When I go around end users, I find people in two camps: The ones who've drunk the sun coolaid and won't take anything on linux that isn't a fully replicated dtrace (sort of like windows people who demand the availability of outlook on linux) and people who are migrating to Linux and trying to use systemtap for tracing. These latter seem to have a number of genuine concerns including latency, the time it takes to actually go from command executing to functional trace, the inability to trace user programs (dtrace can) and concerns about the amount of perturbation the probes actually place inside the kernel. 

Those are all valid concerns, but the biggest problem for users is that, unless they are knowledgeable about kernel internals, it is difficult to know how to use SystemTap. A more simplified interface, one that is less reliant on kernel internals, needs to be created; the way to do that is through the placement of static trace points in the kernel and the creation of "tapsets" to make them easily usable. The SystemTap developers think the kernel hackers are in the best position to do that work. Ted Ts'o [agrees](<https://lists.linux-foundation.org/pipermail/ksummit-2008-discuss/2008-June/000171.html>) but sees some barriers: 

The big thing that are missing are the tapsets -- the macro libraries that allow a system administrator to use it to find and solve performance problems without being a kernel developer, and more importantly, the documentation for said macro libraries so a system administrator can actually use it. 

[ ... ] the real problem isn't as much kernel developers, it's that (a) it's too hard for many kernel developers to use (and so many kernel developers are [not] using it), and (b) there aren't enough tapsets. The latter is something that kernel developers can help solve, but unfortunately I'm not sure discussing it at the Kernel Summit will necessarily lead to making forward progress. 

If the kernel developers have trouble using SystemTap, they are unlikely to add the tapsets that would make it more usable for system administrators and others who have some general kernel knowledge but not enough to sensibly instrument it. For people using distribution kernels--at least for the enterprise distributions and Fedora--it is only somewhat painful to get SystemTap up and running. But kernel hackers tend to run their own kernels, often many different versions in a short period of time, so they need to be able to be easily build one that works with SystemTap and includes all of the debugging information that it requires. 

SystemTap developer Frank Ch. Eigler has a [long reply](<https://lists.linux-foundation.org/pipermail/ksummit-2008-discuss/2008-June/000192.html>) to many of the complaints in the thread. It seems clear that the SystemTap folks and the kernel hackers have not been communicating--there are solutions to many of the problems that were cited. They are in various states of readiness, but are mostly working. So SystemTap is most of the way there for kernel tracing as long as you are well-versed in kernel internals, but that has been true for some time. 

In order to get SystemTap to where it needs to be, the kernel hackers need to be involved. Building the infrastructure and waiting for tapsets to magically appear is not a recipe for success. The SystemTap hackers need to be engaging the kernel community, as well as distributions, to make the tool into something that gets _used_. 

SystemTap can use static probe points, [kernel markers](<http://lwn.net/Articles/245671/>)--merged into 2.6.24--but it is notable that no one has, as yet, made use of them. A concerted effort needs to be made to make the tool more usable for the kernel developers who can, in turn, help make it more usable for others. There is a clear problem when folks like Ts'o regularly try, but [find it too difficult to be useful](<https://lists.linux-foundation.org/pipermail/ksummit-2008-discuss/2008-June/000185.html>): 

But maybe as more people try using it, they'll discover some of these rough edges, and will start trying to fix it. Every couple of months, I've tried using it, and because it [h]as so many rough edges, I've normally found it less work to debug the kernel using manual methods rather trying to make Systemtap work on my system and with my kernel development workflow. 

It is a commonly heard complaint that while SystemTap is difficult to use, DTrace "just works" for Solaris; Eigler responds: 

Yeah, so I hear, but think about how different their target environment is. Their kernel hardly changes (several fixed APIs, ABIs): this has huge implications. Their kernel was willing to insert probes (~ markers), a bunch of build system changes (debug info subset transcribing). Here in linux land, we suffer multifaceted tensions and it is hard to go toward a goal without obstructions (well-meaning as they may be). 

A bunch of third-party scripts are often conflated with "dtrace", which is just a matter of growing the user community enough, and giving them a good tool to build on top of. A growing set of runnable end-user scripts is already packaged with systemtap, intended for use by nonexperts, more help (e.g. concise problem statements about what you'd like to measure/see) would be welcome. 

Many administrators and other users of tracing facilities are not necessarily interested in kernel-level tracing, but would really like to be able to use the instrumented versions of things like PostgreSQL. That is in the plan according to Eigler: ""We aim to piggyback on these efforts by reusing the dtrace instrumentation calls embedded into postgres etc., if at all possible."" 

Until the rough edges can be smoothed on the kernel side, Bottomley [wonders](<https://lists.linux-foundation.org/pipermail/ksummit-2008-discuss/2008-June/000181.html>) if it even makes sense to start considering user space: 

Although there are differing opinions about what systemtap could and should do, it's clear that it's not working incredibly well for its design space: the kernel, so talking about extending it to userspace is a premature. 

DTrace sounds like a nice working solution that has many uses and many happy users. If one can ignore the self-congratulatory [postings from its lead developer](<http://blogs.sun.com/bmc/entry/dtrace_on_linux>), it might be worth having in Linux, but that simply is not going to happen. Paul Fox is [working on a port of DTrace to Linux](<http://www.crisp.demon.co.uk/blog/archives/2008-06.html#e2008-06-29T22_09_45.txt>), but that ignores the licensing realities that would never allow it to become part of Linux. It also ignores the difficult path a DTrace port would face getting merged into the mainline. (We hope to have an article from Mr. Fox on his DTrace porting work soon, stay tuned). 

For all of the talk out of Sun about how they would love to make DTrace a part of Linux, they clearly made a choice to ensure that could not happen. Even if any technical barriers were lifted, the CDDL is not compatible with the GPL. It is perfectly fine as a free software license, but if you wish to get things into Linux, they must be licensed in a GPL-compatible way. This was well understood at the time Sun freed Solaris, so this must have been a conscious decision. Given how much their marketing organization likes to tout DTrace, it would seem to be a choice that Sun is quite happy with. 

Linux will eventually get the tracing support it needs, in a way that is easily accessible to users, but it may take some time. Conversations like the recent one on ksummit-2008-discuss are an important part of getting there. It would appear that better support for the use cases of kernel developers will be forthcoming. It is mostly a matter of documentation along with simplifying some of the building and installation issues. Once the kernel hackers actually start using it, progress is likely to be fairly swift. 

This is the way free software development works; it generally does not track a straight path to a solution, but often wanders about in the solution space for a while. It is highly unlikely that a development like DTrace could have come about in the way that it did in a true community-developed operating system. For that you need everyone pulling in the same exact direction, which may be why Sun is reluctant to turn over much of the governance of Solaris to the community. That may help them develop things more quickly, because there will be fewer barriers, but it won't help them to foster the kind of development community that characterizes Linux.
