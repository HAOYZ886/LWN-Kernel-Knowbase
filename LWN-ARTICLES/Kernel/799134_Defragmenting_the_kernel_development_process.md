---
title: Defragmenting the kernel development process
url: https://lwn.net/Articles/799134/
date: "September 14, 2019"
category: "Development model-Patch management; Development tools-Testing"
author: "By Jonathan Corbet September 14, 2019 Maintainers Summit"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
September 14, 2019

* * *

[Maintainers Summit](<https://lwn.net/Articles/799060>)

The first session at the 2019 Linux Kernel Maintainers Summit was a last-minute addition to the schedule. Dmitry Vyukov's Linux Plumbers Conference session on the kernel development process ([slides [PDF]](<https://linuxplumbersconf.org/event/4/contributions/554/attachments/353/584/Reflections__Kernel_Summit_2019.pdf>)) had inspired a number of discussions that, it was agreed, should carry over into the summit. The result was a wide-ranging conversation about the kernel's development tools and what could be done to improve them. 

Ted Ts'o introduced the topic by noting that his employer, Google, has a group dedicated to the creation of development tools, and that a lot of good things have come from that. The kernel community also has a lot of tools aimed at making developers more productive, but rather than having a single group creating those tools, we have many competing groups. While competition is good, he said, it also diffuses the available development time and may not be, in the end, the best way to go. He then turned the session over to Vyukov. 

#### Lots of bugs

The kernel community has a lot of bugs, he began; various subsystems are often broken for several releases in a row. The community adds new vulnerabilities to the stable releases far too often. The 4.9 kernel, to take one example, has had many thousands of fixes backported to it. There are a lot of kernel forks out there, each of which replicates each bug, so keeping up with these fixes adds up to a great deal of work for the industry as a whole. The security of our [![\[Dmitry Vyukov\]](https://static.lwn.net/images/conf/2019/lpc/DmitryVyukov-sm.jpg)](<https://lwn.net/Articles/799137/>) kernels is "not perfect"; as we fix five holes, ten more are introduced — on the order of 20,000 bugs per release. We need to reduce the inflow of bugs into the kernel, he said, in order to get on top of this problem. 

These bugs hurt developer productivity and reduce satisfaction all around. Many of them can be attributed to the tools and the processes we use. We have many of the necessary pieces to do better, but there is also a lot of fragmentation. Every kernel subsystem does things differently; there is a distinct lack of identity for the kernel as a whole. 

More testing is good, but testing is hard, he said; it's not just a matter of running a directory full of tests. There are a lot of issues that come up. People run subsystem-specific tests, but often fail to detect the failures that happen. Many groups only do build testing. About 15 engineer-years of effort are needed to get to a functional testing setup; the kernel community is spending more than that, but that effort is not being directed effectively. There are at least seven major testing systems out there (he listed 0day, kernelci.org, CKI, LKFT, ktest, syzbot, and kerneltests) when we should just have one good system. 

When the testing systems work, a single problem can result in seven bug reports, each of which must be understood and answered separately. So developers have to learn how to interact with each testing system. Christoph Hellwig interjected that he has never gotten a duplicate bug report from an automated testing system, but others have had different experiences. And, to the extent that duplicate reports do not happen, it indicates that the testing systems are not functional — they are not detecting the problems. 

Laura Abbott pointed out that many of the testing systems are not doing the same thing; their coverage is different, but they still have to reimplement much of the same infrastructure. Thomas Gleixner replied that the problem is at the other end: there is no centralized view of the testing that is happening. There are far more than seven systems, he said; many companies have their own internal testing operations. There is no way to figure out whether the same problem has been observed by multiple systems. Some years ago, the kerneloops site provided a clear view of where the problem hotspots were; now a developer might get a random email and can't correlate reports even when they refer to the same problem. 

Ts'o said that these systems will have to learn to talk to each other, which will require a lot of engineering work. But beyond that, even the systems we have now appear to be overwhelmed. Once upon a time, the 0day robot was effective, testing patches and sending results within hours. Now he will get a report five days later for a bug in a patch that has already been superseded by two new versions. Since there is no unique ID attached to patches, there is no way for the testing system to recognize updated versions. Gerrit has solved this problem, but "everybody hates it", and there is [little acceptance of change IDs](<https://lwn.net/Articles/797613/>) for patches. It all works nicely within Google, he said, but it requires a lot of internal infrastructure. He wondered whether the kernel community could ever have the same thing. 

Dave Miller said that many companies and individuals are replicating this kind of testing infrastructure; that is the source of the scalability problem. Now, he said, he has to merge patches in batches before doing a test build; if something is bad, he will lose a bunch of time unwinding that work. He would much rather get pull requests with build reports attached so that he can act on them without running into trivial problems. The 0day robot used to help in that regard, but it has lost its effectiveness. Abbott wondered if all this effort could be centralized somewhere; given that the kernelci.org effort is [moving into the Linux Foundation](<https://lwn.net/Articles/772525/>), perhaps efforts could be focused there. 

#### Buy-in needed

Vyukov continued by agreeing that, when a maintainer receives a change for merging, they should know that it has passed the tests. Applying should be a simple matter of saying that it is ready to go in during the next merge window. But when individual subsystems try to improve their processes by switching to centralized hosting sites, they just make things worse for the community as a whole by increasing fragmentation. He doesn't know how to fix all of this, but he does know that it has to be an explicit effort with buy-in across the community. There should be some sort of working group, he said; the [proposal](<https://lwn.net/ml/ksummit-discuss/20190912120602.GC29277@pure.paranoia.local/>) from Konstantin Ryabitsev could be a good foundation to build on. 

Alexei Starovoitov was quick to say that no sort of community-wide buy-in to a new system is going to happen; people have too many strong opinions for that. But, if there is a better tool out there, he will try it. The discussion so far has been all "doom and gloom" he said, but the truth is that the kernel has been getting better and development is getting easier; we are rolling out kernels quickly and each is better than the one that came before. It quickly became clear that this view was not universally shared across the room. Steve Rostedt did acknowledge, though, that the -rc1 releases have become more stable than they once were; he credited Vyukov's [syzbot](<https://github.com/google/syzkaller/blob/master/docs/syzbot.md>) work for helping in that regard. 

Dave Airlie pointed out that, after all these years, we still don't have universal adoption of basic tools like Git. Linus Torvalds said that email is still a wonderful tool for stuff that is in development and in need of discussion; work at that stage can't really be put into Git. Miller agreed that email is "fantastic" for early-stage code, but pointed out that the usability of email as a whole is no longer under our control. 

#### Starting with patchwork

Torvalds said that the discussion made it sound like the sky is falling. Our current automated testing infrastructure generates a lot of "crap", but 1% of it is "gold". He encouraged the room to concentrate on concrete solutions to the problems; he liked Ryabitsev's suggestion, which starts by improving the [patchwork](<http://jk.ozlabs.org/projects/patchwork/>) system. That, he said, should be something that everybody can agree on. Airlie said that the [freedesktop.org](<https://www.freedesktop.org/wiki/>) community has done this, though, with fully funded improvements to patchwork, but it is still "an unmanageable mess" that loses patches and has a number of other problems. Miller said that he was one of the first users of patchwork in the beginning. Back then, the patchwork developer was enthusiastic about improving the system to make developers' lives better. But the situation has long since changed, and it is hard to get patchwork improvements now. 

Torvalds said that, if patchwork were to get smarter, more people might use it. There are ways that developers could help it work better. His life got easier, he said, when Andrew Morton started telling him the base for the patches he was sending for merging; the same could be done for patchwork. But patchwork is focused on email, and Miller argued that "email's days are numbered". A full 90% of linux-kernel subscribers are using Gmail, he said, and Google is turning email into a social network with a web site. That gives Google a lot of control over what the community can do. 

Ts'o said that, rather than focusing on a specific tool, it would be better to talk about what works. Gerrit tracks the base of patches now and can easily show the latest version of any given patch series. Patchwork requires a lot more manual work, with result that he has thousands of messages there that he is unlikely to ever get to. Olof Johansson pointed out that Gerrit only understands individual patches and cannot track a series, which is a problem for the kernel community. Peter Zijlstra, said that, instead, its biggest problem is that it is web-based. Miller replied that he wants new developers to have a web form they can use to write commit messages; he spends a lot of time now correcting email formatting. Gleixner said that the content of the messages is the real problem, but Miller insisted that developers, especially drive-by contributors, do have trouble with the mechanics. 

#### Git as the transport layer

If patchwork could put multiple versions of a patch series into a Git repository, Ts'o said, it would enable a lot of interesting functionality, such as showing the differences between versions. That is something Gerrit can do now, and it makes life easier. Torvalds said that about half of kernel maintainers are using patchwork; there is no need to enforce its use, but it is a good starting point for future work that people can live with. But, he repeated, there needs to be a concrete goal rather than the vague complaining about the process that has been going on for years. Ryabitsev's proposal might be a good starting point, he said. 

Greg Kroah-Hartman agreed that patchwork would be a good foundation to build on. But it's not the whole solution. For continuous integration, he said, the focus should be on kernelci.org; it's the only system out there that is not closed. Johansson, though, said that he does not want to have to go into both patchwork and kernelci.org to see whether something works or not. 

One problem with systems like patchwork is the inability to work with them offline. Miller said that, if patchwork stored its data in a Git repository, developers could pull the latest version before getting onto a plane and everybody would be happy. Hellwig said that he has never understood why people like patchwork. It would be better, he continued, to agree on a data format rather than a specific tool. 

Ryabitsev worried that centralized tools would make "a tasty target" for attackers and should perhaps be avoided. He also said that, with regard to data formats, the [public-inbox](<https://lwn.net/Articles/748184/>) system used to implement [lore.kernel.org](<https://lore.kernel.org/>) can provide a mailing-list archive as a Git repository. Torvalds said that lore.kernel.org works so well for him that he is considering unsubscribing from linux-kernel entirely. Ts'o said that a number of interesting possibilities open up if Git is used as the transport layer for some future tool. Among other things, Ryabitsev has already done a lot of work at kernel.org to provide control over who can push to a specific repository. Ryabitsev remains leery of creating a centralized site for kernel development, though. 

As the discussion wound down, Abbott suggested that what is needed is a kernel DevOps team populated with developers who are good at creating that sort of infrastructure. Hellwig put in a good word for the Debian bug-tracking system, which allows most things to be done using email. Ts'o summarized the requirements he had heard so far: tools must be compatible with email-based patch review and must work offline. If the requirements can be set down, he said, perhaps developers will come along to implement them, and perhaps funding can be found as well. 

The session closed with the creation of a new "workflows" mailing list on [vger.kernel.org](<https://vger.kernel.org/>) where developers can discuss how they work and share their scripts. That seems likely to be the place where this conversation will continue going forward. 

[Your editor thanks the Linux Foundation, LWN's travel sponsor, for supporting travel to this event.]
