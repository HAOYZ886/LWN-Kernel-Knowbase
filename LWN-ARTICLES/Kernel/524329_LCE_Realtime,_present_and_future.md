---
title: "LCE: Realtime, present and future"
url: https://lwn.net/Articles/524329/
date: "November 13, 2012"
category: Realtime
author: "By Jonathan Corbet November 13, 2012"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
November 13, 2012

As the standing-room-only crowd at Thomas Gleixner's LinuxCon Europe 2012 talk showed, there is still a lot of interest in the progress of the realtime preemption patch set. Your editor attended with the main purpose of heckling Thomas, thinking that our recent [Realtime Linux Workshop coverage](<https://lwn.net/Articles/520704/>) would include most of the information to be heard in Barcelona. As it turns out, though, there were some new things to be learned, along with some concerns about the possible return of an old problem in a new form. 

At the moment, realtime development is concentrated in three areas. The first is the ongoing work to mitigate problems with software interrupts; that has been [covered here before](<https://lwn.net/Articles/520076/>) and has not changed much since then. On the memory management front, the SLUB allocator is now the default for realtime kernels. A few years ago, SLUB was considered hopeless for the realtime environment, but it has improved considerably since then. It now works well when allocating objects from the caches. Anytime it becomes necessary to drop into the page allocator, though, behavior will not be deterministic; there is little to be done about that. 

Finally, the latest realtime patches include a new option called PREEMPT_LAZY. Thomas has been increasingly irritated by the throughput penalty experienced by realtime users; PREEMPT_LAZY is an attempt to improve that situation. It is an option that applies only to the scheduling of SCHED_OTHER tasks (the non-realtime part of the workload); it works by deferring context switches after a task is awakened, even if the newly-awakened task has a higher priority. Doing so reduces determinism, but SCHED_OTHER was never meant to be deterministic in the first place. The benefit is a reduction in context switches and a corresponding improvement in SCHED_OTHER throughput. 

When SLUB and PREEMPT_LAZY are enabled, the realtime kernel shows a 60% throughput increase with some workloads. Someday, Thomas said (not entirely facetiously), realtime will be faster than mainline, especially for workloads involving a lot of networking. He is looking forward to the day when the realtime kernel offers superior network performance; there should be some interesting conversations with the networking developers when that happens. 

The realtime kernel, he claimed in summary, is at production quality. 

Looking at all the code that has been produced in the realtime effort, Thomas concluded that, at this point, 95% of it has been upstreamed into the mainline kernel. What is missing before the rest can go upstream is "mainline sellable" solutions for memory management, high-resolution timers (hrtimers), and software interrupts. The memory management work is the most complex, and the patches are still "butt ugly." A significant amount of cleanup work will be required before those patches can make it into the mainline. 

The hrtimer code, instead, just requires harassing the maintainer (a certain Thomas Gleixner) to get it into shape; it's just a matter of time. There needs to be a "less horrible" way to solve the software interrupt problem; the search is on. The rest of the realtime tree, including the infamous locking patches, is all nicely self-contained and should not be a problem for upstreaming. 

So what is coming in the future? The next big feature looks to be CPU isolation. This is not strictly a realtime need, but it is useful for some realtime users. CPU isolation gives one or more processors over to user-space code, with no kernel overhead at all (as long as that code does not make any system calls, naturally). It is useful for applications that cannot wait even for a deterministic interrupt response; instead, they poll a device so that they can respond even more quickly to events. There are also high-performance users who pour vast amounts of money into expensive hardware; these users are willing to expend great effort for a 1% performance gain. For some kinds of workloads, the increased cache locality offered by CPU isolation can give an improvement of 3-4%, so it is unsurprising that these users are interested. A number of developers are working on this problem; some sort of solution should be ready before too long. 

Also coming is the long-awaited [deadline scheduler](<https://lwn.net/Articles/356576/>). According to Thomas, this code shows that, sometimes, it is possible to get useful work out of academic institutions. The solution is close, he said, and could possibly even be ready for the 3.8 merge window. There is also interest in doing realtime work from a KVM guest system. That will allow us to offload our realtime automation tasks into the cloud. Thomas clearly thought that virtualized realtime was a bit of a silly idea, but there is evidently a user community looking for this capability. 

#### Where are the contributors?

Given that things seem so close, Thomas asked himself why things were taking so long; the realtime effort has been going for some eight years now. The answer is that the problems are hard and that the manpower to solve them has been lacking. Noting that few developers have been contributing to the realtime tree, Thomas started to wonder if there was a lack of interest in the concept overall. Perhaps all this work was being done, but nobody was using it? 

An opportunity to answer that question presented itself when kernel.org went down for an extended period in 2011. It became necessary to provide an alternative site for people wanting to grab the realtime patches; that, in turn, was the perfect opportunity to obtain download statistics. It turns out that most realtime patch set releases saw about 3,000 downloads within the first three days. About 30% of those went to European corporations, 25% to American corporations, 20% to Asian corporations, 5% to academic institutions, and 20% "all over." 75% of the downloads, he said, were done by users at identifiable corporations constituting a "who's who" of the industry. 

All told, there were 2,250 corporations that downloaded the realtime patch set while this experiment was taking place. Of all those companies, less than 5% reported back to the realtime developers in any way, be it a patch, a bug report, or even an "it works, thanks" sort of message. A 5% response rate may seem good; it should be enough to get the bugs fixed. But a further look showed that 80% of the reports came from Red Hat and the Open Source Automation Development Laboratory; add in Thomas's company Linutronix, and the number goes up to 90%. "What," he asked the audience, "are the rest of you doing?" 

Thomas's conclusion is that something is wrong. Perhaps we are seeing a return of the [embedded nightmare](<https://lwn.net/Articles/232379/>) in a new form? As in those days, he does see private reports from companies that are keeping all of their work secret. Private reports are better than nothing, but he would really like to see more participation in the community: more success reports, bug reports, documentation contributions, and fixes. Even incorrect fixes are a great thing; they give a lot of information about the problem and ease the process of making a proper fix. 

To conclude, Thomas noted that some people have complained that his roadmap slides are insufficiently serious. In response, he said, he took a few days off and took a marketing course; that has allowed him to produce a more suitable roadmap that looks like this: 

> ![\[Thomas's new roadmap\]](https://static.lwn.net/images/2012/rt-roadmap-bcn.png)

Perhaps the best conclusion to be drawn from this roadmap is that Thomas is unlikely to switch from development to marketing anytime in the near future. That is almost certainly good news for Linux users — and probably for marketing people as well. 

[Your editor would like to thank the Linux Foundation for funding his travel to Barcelona.]
