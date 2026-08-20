---
title: Another look at the new development model
url: https://lwn.net/Articles/95312/
date: "July 27, 2004"
category: Development model; Releases
author: 
---

Discussions held at OLS, on the mailing lists, and elsewhere have made it clear that a certain degree of confusion still exists regarding the new kernel development process and what has really changed. In an attempt to clear things up, we'll take one more look at what was decided at this year's kernel summit. 

The old process, in use since the 1.0 kernel release, worked with two major forks. The even-numbered fork was the "stable" series, managed in a way which (most of the time) attempted to keep the number of changes to a minimum. The odd-numbered fork, instead, was the development series, where anything goes. The idea was that most users would use the stable kernels, and that those kernels could be expected to be as bug-free as possible. 

This mechanism has been made to work, but it has a number of problems which have been noticed over the years. These include: 

  * The stable and development trees diverge from each other quickly, especially since big API changes have tended to be saved for early in the development series. This divergence makes it hard to port code between the two trees. As a result, backporting new features into the stable series is hard, and forward-porting fixes is also a challenge. 2.6.0 came out with a number of bugs which had long been fixed in 2.4. 

  * The stable tree, after a short while, lacks fixes, features, and improvements which have been added to the development tree. That code may well have proved itself stable in the development series, but it often does not make it into a stable kernel for years. The kernels that people are told to use can run far behind the state of the art. 

  * The stable kernels are often very heavily patched by the distributors. These patches include necessary fixes, backports of development kernel features, and more. As a result, stock distribution kernels diverge significantly from the mainline, and from each other. Distributor kernels sometimes are shipped with early implementations of features which evolve significantly before appearing in an official stable kernel, leading to compatibility problems for users. 

The focus on keeping changes out of the stable kernel tree is now seen as being a bit misdirected. Well-tested patches can be safely merged, most of the time. Blocking patches, instead, creates an immense "patch pressure" which leads to divergent kernels and a major destabilizing flood whenever the door is opened a little. 

So how have things changed? The "new" process is really just an acknowledgment of how things have been done since the 2.6.0 release - or, perhaps, a little before. It looks like this: 

  * New patches which appear to be nearing prime-time readiness are added to Andrew Morton's -mm tree. This addition can be done by Andrew himself, or by way of a growing number of BitKeeper repositories which are automatically merged into -mm. 

  * Each patch lives in -mm and is tested, commented on, refined, etc. Eventually, if the patch proves to be both useful and stable, it is forwarded on to Linus for merging into the mainline. If, instead, it causes problems or does not bring significant benefit, the patch will eventually be dropped from -mm. 

The -mm tree has proved to be a truly novel addition to the development process. Each patch in this tree continues to be tracked as an independent contribution; it can be changed or removed at any time. The ability to drop patches is the real change; patches merged into the mainline lose their identity and become difficult to revert. The -mm tree provides a sort of proving ground which the kernel process has never quite had before. Alan Cox's -ac trees were similar, but they (1) were less experimental than -mm (distributors often merged -ac almost directly into their stock kernels), and (2) -mm does a much better job of tracking each patch independently. 

In essence, -mm has become the new kernel development tree. The old process created a hard fork and was not designed to merge changes back into the "old" stable tree. -mm is much more dynamic; it exists as a set of patches to the mainline, and any individual patch can move over to the mainline at any time. New features get the testing they need, then graduate to the mainline when they are ready. New developments move into the stable kernel quickly, the development kernel benefits from all fixes made to the stable branch, and the whole process moves in a much faster and smoother way. 

More than one observer in Ottawa made this ironic observation: it would appear that Andrew Morton is now in charge of the development kernel, while Linus manages the stable kernel. That is not quite how things were expected to turn out, but it seems to be working. Consider some of the changes which have been merged since 2.6.0: 

  * 4K kernel stacks 
  * NX page protection and ia32e architecture support 
  * The NUMA API 
  * Laptop mode 
  * The lightweight auditing framework 
  * The CFQ disk I/O scheduler 
  * Netpoll 
  * Cryptoloop, snapshot, and mirroring in the device mapper 
  * Scheduling domains 
  * The object-based reverse mapping VM 

Some of these changes are truly significant, and things have not stopped there: new patches are going into the kernel at a rate of about 10MB/month. Yet 2.6.7 was, arguably, the most stable 2.6 kernel yet. It contains many of the latest features, has few performance problems, and the number of bug reports has been quite small. The new process is yielding some good results. 

Naturally, there are some issues to resolve. One of those is the deprecation of features, which used to be tied to the timing of the old process. The new plan, it seems, is to give users a one-year notice, including a `printk()` warning in the kernel. The first features to be removed by this path are likely to be devfs and cryptoloop. There is also the question of changes which are simply too disruptive to merge anytime soon. Page clustering, if it is merged, could be one of those. When such a feature comes along, we may yet see the creation of a 2.7 tree to host it. Even then, however, 2.7 will track 2.6 as closely as possible, and it may go away when the feature which drove its existence becomes ready to go into the mainline. 

This change to the development process is significant. It is not particularly new, however. The actual change happened the better part of a year ago; it was simply hidden in plain sight. All that has really happened in Ottawa is that the developers have acknowledged that the process is working well. One can easily argue, in fact, that the kernel development process has never functioned better than it does now. So, rather than break such a successful model, the developers are going to let it run.
