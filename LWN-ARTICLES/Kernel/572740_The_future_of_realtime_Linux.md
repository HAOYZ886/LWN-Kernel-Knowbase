---
title: The future of realtime Linux
url: https://lwn.net/Articles/572740/
date: "November 6, 2013"
category: Realtime
author: "By Jake Edge November 6, 2013 Real Time Linux Workshop"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jake Edge**  
November 6, 2013

* * *

[Real Time Linux Workshop](<https://lwn.net/Archives/ConferenceByYear/#2013-Realtime_Linux_Workshop>)

The future of the realtime (aka `PREEMPT_RT`) kernel patch set was on the agenda for the Realtime Linux minisummit—as usual—but this year's edition had a bit more urgency than in years past. It is clear that Thomas Gleixner, who is doing most of the development work on the patch set, is concerned about the future of the remaining pieces. There appears to be minimal interest in furthering the development of realtime Linux outside of its main sponsor, Red Hat, and that may not be a sustainable model, he reported to both the minisummit and the concurrent [15th Real Time Linux Workshop](<https://www.osadl.org/RTLWS-2013.rtlws-2013.0.html>) (RTLWS). 

[ ![\[Group photo\]](https://static.lwn.net/images/2013/rtlws-minisummit-sm.jpg) ](<https://lwn.net/Articles/572949/>)

The RTLWS is a four-day gathering of those interested in using Linux for realtime applications. The minisummit was an all-day meeting on the first day of the workshop for nine kernel developers who are involved in the `PREEMPT_RT` work: Ingo Molnar, Sebastian Siewior, Frédéric Weisbecker, John Kacur, Paul McKenney, Steven Rostedt, Paul Gortmaker, Gleixner, and Darren Hart. Gleixner also reported on the minisummit to the participants of the workshop on its second day. The conclusion was the same in both cases: the `PREEMPT_RT` project will be done in 2014, "one way or another". The first way would be to get most of the rest of the code upstream, but that will require more of an effort, from a wider group than is currently involved. The alternative is to decide that the 95% of the realtime work already upstream is good enough and to drop further efforts. 

#### Status

Since [last year's minisummit](<https://lwn.net/Articles/520704/>), not much has changed for the realtime patch set, Gleixner said. He spent three months trying to clean up some of the last pieces to get them ready for mainline, but that quickly ballooned into nine separate projects with circular dependencies. That code was "stashed in the horror cabinet", he said, but the work was not entirely wasted as he has a good idea of what needs to be done, "but it's not pretty". 

In order to focus on the development side of things, Gleixner has handed off doing the -rt releases to Siewior. The "whole problem" over the last couple of years has been the lack of developers, paid developers in particular, Gleixner said. Red Hat pays him to work half-time on realtime development and it pays the [messaging, realtime, and grid](<http://www.redhat.com/products/mrg/>) (MRG) team to test and productize the realtime patch set. Beyond that, there are a few other contributors, including McKenney for read-copy-update (RCU) development as well as Hart and Gortmaker. Both of the latter two indicated they would try to encourage their employers (Intel and Wind River, respectively) to contribute more to realtime development moving forward. 

Gleixner said that he spoke with Linus Torvalds and Andrew Morton at the recently completed [Kernel Summit](<https://lwn.net/Articles/KernelSummit2013/>) about the remaining realtime code. Both were favorably disposed toward getting those pieces upstream, but Gleixner is concerned that the lack of a dedicated realtime development team to sustain the code may be an impediment. Torvalds worries less about drivers or filesystems that "can bitrot away at the edge of the kernel" without affecting anything else, Gleixner said, but core kernel code is different. If Red Hat were to decide that realtime was no longer of interest, that code might go largely unmaintained. 

Part of the problem may be that it is smaller and smaller segments of the user base that is being served with each addition of realtime code to the mainline. McKenney said that he has seen that with RCU—the mainline version works for a lot of people, which is a sign of maturity. Newer features are just for specialized situations, which means it is hard to get a groundswell of support behind them. 

Similarly, Hart said that there are customers who are talking about using `PREEMPT_RT` with the Yocto project, but there would need to be a whole lot more of them before he is likely to be able to convince his management to have him work half-time on realtime. The [Open Source Automation Development Lab](<https://www.osadl.org/Home.1.0.html>) (OSADL)—sponsor of the RTLWS—was originally set up to gather enough funds to support four or so full-time engineers working on realtime Linux, Gleixner said. Unfortunately, the number of members has never risen to the point where that became a reality and the organization is just able to support its [QA test farm](<https://www.osadl.org/QA-Farm-Realtime.qa-farm-about.0.html>) alongside its other activities, he said. 

From statistics gathered by Gleixner while he was hosting the -rt patches during the kernel.org outage, it is clear that there is interest in realtime Linux. Within five days of the announcement of a new -rt kernel, he would see around 3000 downloads from addresses that read like a "who's who" of the computer industry, he said. In particular, Germany—where OSADL is based—was well represented, with 45% or so of the downloads coming from there. 

As Hart noted, this is a different kind of problem than the realtime project has faced in the past. It is not a technical problem like those that the project has tackled—largely successfully. There may be some help on the horizon, however, as there are plans to put the MRG offering on top of the RHEL kernel (with the -rt patches, of course), Kacur said. That may free up some people who are currently working to test and stabilize the -rt kernels because much of the RHEL driver testing and the like can be reused. 

Gortmaker asked what the "end game" for realtime Linux looked like, was it going to be like linux-tiny with some out-of-tree patches? Gleixner acknowledged that was likely to be the case; some "special case" pieces would need to be maintained out of tree. 

Something that might help realtime get the final push it needs to be mainlined and to have enough maintenance support would be a contract that required mainline realtime support, McKenney suggested. The original realtime patches came about due to a contract that IBM and Red Hat had with the US Navy, so something like that might come along again. 

While the financial industry was once a hotbed for interest in realtime, that seems to have cooled somewhat. The traders are more interested in throughput and are willing to allow Linux to miss some latency deadlines in the interests of throughput, Hart said. The embedded use cases for realtime seem to be where the "action" is today, Gleixner and others said, but there has been little effort to fund realtime development coming from that community. 

#### Technical hurdles

After that, the conversation moved on to the remaining technical hurdles to clear to "finish" the realtime work. Gortmaker noted that Ted Ts'o was [not opposed](<http://www.spinics.net/lists/linux-rt-users/msg10138.html>) to replacing the bit spinlocks (single bits used as spinlocks to save space) in the ext4 filesystem with regular spinlocks if it could be shown there were no performance impacts or, better still, that performance increases offset the extra space used in buffer-heads. Bit spinlocks are a problem for the realtime code because they cannot be turned into sleeping spinlocks as is done with other mainline spinlocks. Gortmaker said that he plans to pursue the conversion with Ts'o as a possible step toward eliminating bit spinlocks. 

Gleixner suggested that a patch which converted bit spinlocks to regular spinlocks when lockdep is enabled might be one approach to solving the bit spinlock problem for realtime. Right now, bit spinlocks are not tracked by lockdep, so a cleanup that tracked them could be sold as a debug feature and the realtime patches could enable that mode as well. Molnar said that doing so might well find locking bugs, which would demonstrate its usefulness. 

Rostedt asked about getting rid of uses of the `cpu_chill()`, which can lead to livelock situations. Gleixner said that it is a replacement for `cpu_relax()` that just does an `msleep(1)` to allow "nasty trylock loops" to continue to work. By delaying the looping task for a tick, it allows a preempted task to make progress which will, eventually, allow it to release a lock the looping task is waiting on. 

Rostedt called `cpu_chill()` a "hail Mary" that just hopes whoever has the lock will let go of it. He suggested that the waiting task temporarily give its priority to the lock holder, but others thought that fixing the directory entry cache (dcache) code where `cpu_chill()` is used would be a better approach. For now, the `msleep()` is reasonable. 

The bottleneck of turning reader-writer locks (rwlock) and semaphores (rwsem) into a single spinlock was another issue that Rostedt raised. He noted that there was some ongoing work by Mathieu Desnoyers to turn the oft-contended mmap_sem into something else. "Any heavily threaded application" works poorly on the realtime kernel, Rostedt said, because of mmap_sem contention. Either the realtime kernel can do something realtime-specific to alleviate the problem or there could be an effort to get rid of rwsems in the mainline kernel. 

McKenney said that he had put Desnoyers and Peter Zijlstra together at the Kernel Summit to discuss the effort to rework mmap_sem. Zijlstra had made an attempt at the rework earlier, based on a paper from MIT that applied RCU in the page fault path to avoid the need for mmap_sem. It used a different kind of tree that required less rebalancing to track the mappings. He wasn't sure if anything came out of that conversation but that is one possible approach. 

Another approach, suggested by Molnar, is to just eliminate rwsem in the realtime patch set and allow memory management to be non-deterministic. If an application cares about deterministic memory management behavior, it should be using `mlock()`, Gleixner said. Just removing rwsem and "see who complains" is probably best, Molnar said. If someone truly needs the functionality in the realtime kernel, they can fund the development of a realtime variant of rwsem, he said. 

There was also discussion of restricting which softirqs are run when softirqs are processed as a result of a call to `local_bh_enable()`. A mask value could be provided to the `local_bh_disable()` call that would specify _which_ types of softirqs were being disabled. Typically it would just be those for the subsystem doing the disabling, but others could be added if they needed to be held off. 

Gortmaker plans to create an API to post as an RFC soon that would add the mask, but hide it behind a higher-level interface (like `local_bh_disable_net()` or similar). It would also convert one type of softirq (scheduler, RCU, or timer were suggested) to use the new API. That way, people can "scream" right away, Gleixner said, if they don't like the idea and another way to handle softirq processing for realtime can be found. 

#### Where to next?

In summary, Gleixner said, much of the "low-hanging fruit" from last year's to-do list has been dealt with. Gortmaker picked up many of those and got them to the right subsystem maintainer, Gleixner said. What's left are the "nasty and intrusive" pieces that he keeps trying to "wrap my brain around", but the lack of developer time has really hurt that process. 

[ ![\[Thomas Gleixner\]](https://static.lwn.net/images/2013/rtlws-gleixner-sm.jpg) ](<https://lwn.net/Articles/572952/>)

Gleixner expanded on that problem some more in the report to the RTLWS. He believes a team of four or five full-time developers is needed to make realtime Linux truly sustainable. Without that kind of commitment, he is concerned that Torvalds will be unwilling to take the kind of core changes required by the remaining pieces of the realtime patch set. 

The community contribution to the realtime patch set "amounts roughly to zero", he said. There is a fair amount of frustration on the team in always chasing mainline and being unable to stop realtime-unfriendly mainline features because the code is out of tree. In his mind, that should not continue past 2014; either a larger group steps up to work on the code or the project can be declared "finished", he said. "I could live with either outcome." 

Gleixner said that he was trying to scare the audience a little bit with his proclamation, but it is clear he has nearly reached the end of his rope. While he would like to see the project continue—and prosper—he has real concerns about making the kinds of changes to the kernel that are required without a deeper and wider group to maintain it all. It seems that it is now up to those who use the realtime kernel to either step up or prepare for a future where the mainline kernel will have to serve their needs.
