---
title: "Kernel regression tracking, part 2"
url: https://lwn.net/Articles/738216/
date: "November 6, 2017"
category: Regression tracking
author: "By Jonathan Corbet November 6, 2017 2017 Maintainers Summit"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
November 6, 2017

* * *

[2017 Maintainers Summit](<https://lwn.net/Articles/KernelSummit2017>)

The tracking of kernel regressions was [discussed at the 2017 Kernel Summit](<https://lwn.net/Articles/737666/>); the topic made a second appearance at the first-ever Maintainers Summit two days later. This session was partly a repeat of what came before for the benefit of those (including Linus Torvalds) who weren't at the first discussion, but some new ground was covered as well. 

Thorsten Leemhuis started with a reprise of the Kernel Summit discussion, noting that he has been doing regression tracking for the last year and has found it to be rather harder than he had expected. The core of the problem, he said, is that nobody tells him anything about outstanding regressions or the progress that has been made in fixing them, forcing him to dig through the lists to discover that information on his own. He had, though, come to a few conclusions on how he wants to proceed. 

First, he will try again to establish the use of special tags to identify regressions. His first attempt had failed to gain traction, but he agreed that he perhaps had not tried hard enough to publicize the scheme and get developers to use it. He will be looking into using the kernel Bugzilla again, even though it still seems like unpleasant work to him. He'll try to improve the documentation of how regressions should be tracked and [![\[Thorsten
Leemhuis\]](https://static.lwn.net/images/conf/2017/osse/ThorstenLeemhuis2-sm.jpg)](<https://lwn.net/Articles/738220/>) handled. There is a plan to create a new mailing list on vger.kernel.org, with the idea that regression reports would be copied there. He will put more effort into poking maintainers about open regressions. 

The discussion quickly turned to the problem (as seen by some) of the many kernel subsystems that do not use the kernel.org Bugzilla instance for tracking bugs. Peter Anvin said that many developers don't see much value in that system. Reported bugs tend to say something like "my laptop doesn't boot" with no further information; that tends not to be useful for the identification of any actual bugs. Beyond that, many bugs reported against the core kernel or x86 architecture turn out to be driver bugs in the end. 

Users, it was suggested, should be explicitly directed to the mailing lists when reporting bugs for the subsystems that do not use Bugzilla. Laura Abbott said that this would be just a beginning; the kernel is lacking more general guidance on where and how to report bugs. Ted Ts'o, though, suggested that many kernel developers like the current system, which tends to filter out reports from relatively non-technical users who are unable to create useful reports. It could be seen as a feature, he said; perhaps such users would be better directed to distributor bug trackers. 

One ongoing problem is that many of the less technical users are unable to build their own kernel to test a patch intended to fix their problem. Ben Herrenschmidt said there might be value in a facility that would automatically generate a package containing a distribution kernel with a patch added. But Greg Kroah-Hartman said that this discussion (as a whole) had come up many times before. Before creating elaborate systems, it might be best to create a better landing page on the kernel Bugzilla to help users report their bugs. Arnd Bergmann said that this landing page could perhaps be a wiki so that maintainers could easily add information on how to report bugs in their own subsystems. Takashi Iwai suggested that more subsystems should use Bugzilla; it can host useful materials like screenshots that are not really suitable for the mailing lists, but Kroah-Hartman said that those subsystems have managed without Bugzilla so far. 

As in the previous session on this topic, it was noted that the linux-kernel mailing list is a black hole; reports sent there without copies to the relevant maintainers are likely to go unread. Once again, it was suggested that a bot should be set up to reply to such postings with suggestions on how to reach an actual human. Another echo from that session was the notion that some subsystem maintainers are resistant to having their bugs called "regressions", since it requires them to respond to them more quickly. Linus Torvalds said that he should be told about any such maintainers, who would then be able to expect a strongly worded message from him. Chris Mason, instead, said that some maintainers like to tag problems as regressions, since regressions are a ticket to get patches into a late -rc kernel release. 

Torvalds said that the problem that led to Rafael Wysocki ceasing his regression-tracking work was that he was the only one doing it. Leemhuis, too, is doing this job on his own, and it's a grind. There's only so much help that can be had from more scripting and documentation, what really needs to happen is that more people need to be involved in tracking regressions. In that regard, the Bugzilla is useful because it helps people to work together, even though it is "hell" in general. 

The session wound down with a couple of brief side discussions. Many regressions in the kernel are related to specific hardware, which makes them hard to write tests for. Dan Williams said that, using mocking, unit tests can be created to enable at least partial testing for drivers. Finally, it was noted that it would be useful to know which subsystems, in particular, have been prone to regressions; that could help identify parts of the kernel that could use some refactoring, better self tests, or changes in maintenance style. At the moment, nobody really knows which subsystems those are. 

[Your editor would like to thank the Linux Foundation, LWN's travel sponsor, for supporting his travel to this event].
