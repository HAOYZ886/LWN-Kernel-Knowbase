---
title: Kernel tinification
url: https://lwn.net/Articles/608945/
date: "August 20, 2014"
category: Embedded systems
author: "By Jonathan Corbet August 20, 2014 Kernel Summit 2014"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
August 20, 2014

* * *

[Kernel Summit 2014](<https://lwn.net/Articles/KernelSummit2014/>)

Josh Triplett started his 2014 Kernel Summit session with an assertion: saying that "Linux runs on everything from cellphones to supercomputers" is trite and uninformative. In 2014, he said, a smartphone is not an embedded system. He would like to convince kernel developers to pay a bit more attention to the needs of _real_ embedded systems — those with only a few megabytes of flash and a tiny amount of memory. The Linux kernel, he says, has gotten far too big to run comfortably on such systems; he would like to build a consensus in favor of a "tinification" effort to shrink things back down. 

![\[Size chart\]](https://static.lwn.net/images/conf/2014/ks/size-chart.jpg) He put up a chart showing the in-memory size of the smallest possible kernel over time. That size, he said, has increased with almost every release. It rarely goes down; about the only time was when developers figured out how to configure out the TTY layer. A growing kernel makes it hard for the people who are trying to build tiny systems, forcing them to go to a proprietary real-time operating system instead. It should be possible to use Linux on such systems, he said, but we have to make it possible to build smaller kernels — perhaps as much as an order of magnitude smaller. 

Some people apparently suggest the use of 2.4 kernels for such use cases. The 2.4 kernel is indeed quite a bit smaller, but it still does not solve the problem. Just because somebody is using tiny hardware does not mean that they can do without (some of) the capabilities found in current kernels. 

Some notes on building tiny kernels can be found at [tiny.wiki.kernel.org](<http://tiny.wiki.kernel.org/>). 

Part of the problem, Josh said, is that the kernel is full of mandatory subsystems that cannot be configured out. If somebody wants to, say, build a kernel without support for signals, it should be possible for them to do so. We also suffer from a lack of good metrics on kernel size; most developers are blissfully unaware of how their changes affect the size of the kernel. 

Ted Ts'o mentioned a related discussion on the ext4 development list. Tiny systems often run without a memory management unit (MMU), but most filesystems do not work in a kernel configured to operate without an MMU. Is it worth the trouble to make filesystems work on such systems? After all, the high-end filesystems are developed for large systems and scalability; they may not have much to offer for tiny systems. Others agreed, noting that tiny systems often do not even have the block layer built into them. Josh said that there may be value in ext4 support, though. Among other things, there is work afoot to use Linux as a bootloader on small systems; ext4 support is important there. 

Grant Likely said that, while he is happy to see patches to enable the building of smaller kernels, he is worried about how all of the configuration options would work. We can't handle the complexity of the options we have now. Also, most kernel development is focused on the addition of new features. There is not a lot of pressure to make the kernel smaller, so nobody is working on it. The users who want this capability, he said, are not having a big impact on kernel development. 

Peter Zijlstra agreed that more configuration options would not be welcome; we cannot, he said, even build-test all of the combinations of the current options. There was some talk of trying to hide many of the options and to instead use dependencies to select the right features when needed. But there are a number of practical difficulties here, especially in situations where a number of kernel subsystems could conceivably satisfy a given dependency. 

Grant said, though, that to talk about configurations and dependency resolution was to get lost in side issues. People who are building tiny systems will have no choice but to get into the details of how their kernels are configured. What is needed are developers who are actively working on size issues. While other developers may have agreed with this point of view, the discussion quickly moved back to configuration issues and, in particular, the feared explosion of the size of the configuration space. 

It was suggested that a user of a tiny system could provide the configuration system with a list of the system calls that will actually be used on the deployed system. The configuration code would then decide how to create a minimal kernel that provides the needed capabilities. Ted said [![\[Josh Triplett\]](https://static.lwn.net/images/conf/2014/ks/JoshTriplett-sm.jpg)](<https://lwn.net/Articles/608949/>) that this might be a good project for somebody to work on out of tree. In the end, he said, kernel developers simply do not have the time to test this kind of feature. 

In general, Ted asked Josh, what would he like from the group? Josh answered that he would like developers to start seeing size increases as a regression and to be willing to accept reasonable patches to fix those regressions. Tejun Heo responded that this was not reasonable, given all of the other constraints that kernel developers must deal with. Worrying about size will just make the whole task even harder. 

Tim Bird said that at least some of this task could be automated. There has been some research work on the elimination of unneeded code. In the more immediate future, he would like to see the [link-time optimization](<https://lwn.net/Articles/512548/>) (LTO) work merged into the kernel. Josh agreed that LTO is useful; among other things, it should enable the removal of a number of size-oriented configuration options. Andi Kleen, who has done the bulk of the LTO work so far, said that it can help, but that the code has to be designed correctly to get the most value from LTO. "Crazy callback dependencies," he said, will thwart LTO. Linus said that, for now, he does not trust the LTO implementation in the current toolchain, so he does not want to let the changes into the kernel. Toolchain bugs can create no end of obscure problems, and he does not want to see those in the kernel. Until that code stabilizes, LTO will remain outside. 

It was also suggested that it would help to point to a real target or two for tiny kernels. As it is, developers have little understanding of the type of system that is being talked about. David Woodhouse suggested that the QEMU emulator could be set up as a reference target, but Grant asserted that a QEMU configuration is easy for developers to ignore. What is needed is a real hardware target. 

Ted pointed out that "you get what you measure." What is needed, he said, is more information on kernel bloat and what is causing it. Andi responded that developers do not really care about this issue; size issues are simply not on their radar. Chris Mason suggested that Fengguang Wu's build system could start providing size-change information; that would put developers more in the position of intentionally choosing to make the kernel bigger. Andi agreed that size increases are usually done by accident; making them more visible might help. 

As the session reached a close, Josh suggested that it would be useful to have a configuration option to turn absolutely everything off. It would not normally be possible to re-enable specific items, but developers working on tiny systems could apply a patch to get around that limitation. Once they have done that, though, the only way the development community would be interested in a problem report would be if it came accompanied with a patch to fix the problem. 

What will come of this discussion remains to be seen. But, as Josh pointed out, if nothing else, developers are starting to talk about the problem, even if they are just scoffing. One has to start somewhere. 

**Next** : [What makes Linus happy (or not)?](<https://lwn.net/Articles/608950/>)
