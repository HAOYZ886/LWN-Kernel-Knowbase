---
title: "KS2007: The customer panel"
url: https://lwn.net/Articles/248878/
date: "September 8, 2007"
category: Head Bubba; Latency
author: "By Jonathan Corbet September 8, 2007 LWN.net Kernel Summit 2007 coverage"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
September 8, 2007

* * *

LWN.net [Kernel Summit 2007](<https://lwn.net/Articles/KernelSummit2007/>) coverage 

An occasional kernel summit feature is the customer panel, which gives people doing interesting things with Linux the opportunity to share their experiences (and frustrations) with the developers. At the 2007 gathering, this panel was made up of Sean Kamath (Dreamworks), Head Bubba (Credit Suisse), and Marcus Rex, who represented the Linux Foundation vendor and user advisory councils. Together, they presented an interesting picture of the sorts of troubles one runs into when pushing the edge with Linux. 

### Dreamworks

Sean lead off the group. Dreamworks uses about 2700 systems to perform rendering, along with some 1200 desktops - and they all run Linux. They are mostly multi-core SMP 64-bit systems with a lot of shared data, so Dreamworks makes heavy use of NFS and the automounter. 

Animation design and rendering involves the use of large amounts of memory. So Dreamworks has a lot of people running very large applications, in the form of interactive animation tools and batch rendering engines. One of their biggest problems appears to be swapping; most machines tend to run into swap most of the time despite being equipped with generous amounts of memory. The batch rendering jobs which run during the night can push things to the point that the out-of-memory killer comes into play, with the usual results: somehow the wrong process always gets killed. It is only recently that they discovered the `/proc` OOM-control parameters and started making use of them. It was noted that this is a common problem; we provide many useful features in the kernel, but they remain unused because people do not know about them. 

Even when the OOM killer does not come into play, Dreamworks employees have the morning hangover problem in a big way. The overnight rendering cranker will have shoved everything useful for interactive work (including, possibly, a large rendering application) out to swap, where it languishes until somebody tries to resume work the next morning. It takes a long time for everything to swap back in, to the point that it is often quicker to just restart the application. This is, of course, the classic use case for the [swap prefetch](<http://lwn.net/Articles/242765/>) patch. There was a brief discussion of swap prefetch, but everybody seemed to realize that there was little point in trying to resolve that question in this setting. So there is still no decision on swap prefetch - a situation which should not be surprising at this point. 

Sean would like better ways to quantify memory usage, and to control it as well. There appears to be help on the way in the form of the [memory controller](<http://lwn.net/Articles/243795/>) patch, once that makes its way into the mainline. With the exception of a small number of other issues (an NFS mounting regression which should be fixed in 2.6.23, for example), Dreamworks appears to be happy with its Linux-based rendering setup. 

### Credit Suisse

Coming next was the Credit Suisse IT manager known as Head Bubba. Credit Suisse, a bank with some 45,000 employees, uses Linux to manage and execute literally millions of dollars per second in trades. Most interestingly, he said that the in-house developers are working increasingly with current, upstream kernels, often enhanced with the realtime patches. These developers would rather work with the community than with the distributors, who, they feel, are holding things back. So Credit Suisse may set up an internal "center of excellence" for Linux to support the use of stock mainline kernels in its operation. Mr. Bubba did not say this himself, but there would appear to be a message here that the distributors need to hear. 

That said, there can be challenges associated with the use of mainline kernels. The applications run at Credit Suisse are very carefully tuned to work with the Linux scheduler. If the behavior of the scheduler changes, things do not work so well anymore. So, among other things, Credit Suisse developers need good information about the changes which are coming. Your editor resisted the temptation to try to sell him an LWN subscription on the spot. 

They like the realtime patches; as it happens, running in a real time mode not only reduces latency (a critical consideration for them), but improves throughput at the same time. What they _really_ want is the combination of realtime and RDMA. It's not necessarily RDMA itself that they like, but, at this time, RDMA seems to best provide the sort of performance characteristics that Credit Suisse needs. 

Better diagnostic tools are needed; offerings like [Concurrent NightStar](<http://www.ccur.com/isd_solutions_nightstartools.asp?o14=1>) and [Intel ATOM](<http://www.intel.com/cd/software/products/asmo-na/eng/219608.htm>) were mentioned. Mr. Bubba asked for a better SystemTap implementation, setting off a small side-discussion on the quality of that tool. Some developers discuss those patches in terms which are hard to reproduce in a work-safe publication; others feel that most of the work is reasonable. Tracing tools are a problem area; this subject came up again later in the session. 

TCP/IP jitter is a big problem for Credit Suisse. In some situations, TCP connections are subject to pauses of up to 40ms caused by the [Nagle algorithm](<http://en.wikipedia.org/wiki/Nagle's_algorithm>) and the congestion avoidance code. TCP slow start, it seems, just does not work well for some applications. This is not a small issue; Credit Suisse has a lot of customers who lurk for some time, then decide to spring into action when the market conditions are to their liking. 40ms can be enough for those conditions to move on, robbing a trader of the intended profit from the deal. 

When brief pauses so visibly cause your customers to lose money, you tend to put significant resources into tracking them down and fixing them. Head Bubba had a set of plots demonstrating the problem with a variety of network interfaces and parameter settings. Part of the appeal of RDMA come from the fact that RDMA seems to be less subject to this kind of problem. Credit Suisse is also playing with things like user-space TCP/IP stacks. But it would be nicer to get the TCP problems fixed. 

Developers at Credit Suisse understand that saying "our top-secret proprietary in-house application isn't working" is a hard way to get results from the development community. A little more information is required before a serious attempt to solve the problem can be made. Part of the problem would appear to be that Credit Suisse would like to get full bandwidth out of high-performance network adapters while sending large numbers of small (64-byte) packets. Network stack tuning tends to be tuned more for transfers of larger amounts of data. 

Finding a way to demonstrate the problem for the development community would be a most useful thing to do. So Credit Suisse's developers are working on a set of test cases which will reproduce the sort of behavior exhibited by those applications; the test cases will then be made available to the community. David Miller, the networking maintainer, welcomed this plan; any problem which he can reproduce, he says, will get fixed immediately. 

### Linux Foundation

Finally, Marcus Rex presented the wishlists which have been developed for the Linux Foundation by its advisory councils. Many of these have been seen before and won't be covered in great detail here. They would like to see a single kernel which can work with all of the various virtualization options out there - a problem which is pretty well solved at this point. Better power management - "green computing" and all that - remains on the wishlist. Better hardware support is an item which will probably never go away; there are still members asking for better binary module support, unfortunately. 

There is interest in better security options - and, in particular, security which is easier to manage. Some users are asking for an integrated kernel debugger, though nobody is entirely sure why. A related wish has to do with tracing tools - this led to another discussion of SystemTap. There are problems with the fact that much of the tracing code remains outside of the mainline kernel. It is hard to blame the tracing hackers, though - they have been trying, in one way or another, to merge tracing for several years. At this point, many of them have despaired of ever getting tracing into the mainline. We may see, in the near future, increased pressure to get some of the tracing code merged even if there are still developers who are opposed to it. 

Also on the wishlist were the usual scalability stuff, especially for the sorts of loads (databases, for example) which don't currently scale quite as well on Linux. IPv6 readiness is wanted; there are increasingly stringent governmental requirements for IPv6 coming into force which must be addressed. They would like to see more formal testing happening. And there was a request around ZFS: it seems that it's not the filesystem itself they want, but the relatively easy administration that ZFS offers. RDMA was also on the list. These topics did not generate a whole lot of discussion, though; the belated arrival of coffee outside of the meeting room appeared to be a strong distraction by this stage of the session.
