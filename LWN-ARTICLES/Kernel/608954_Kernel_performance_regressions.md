---
title: Kernel performance regressions
url: https://lwn.net/Articles/608954/
date: "August 20, 2014"
category: Performance regressions
author: "By Jonathan Corbet August 20, 2014 Kernel Summit 2014"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
August 20, 2014

* * *

[Kernel Summit 2014](<https://lwn.net/Articles/KernelSummit2014/>)

Performance regressions are some of the nastiest problems that can afflict the Linux kernel. Often they go unnoticed when they are introduced. Some time — often years — later, some large user attempts to upgrade to a new kernel and notices that things have slowed down considerably. By then, the original problem (or series of problems) may be hard to track down. For this reason, performance regressions are a common Kernel Summit topic, and 2014 was no exception. What might be exceptional is that, it would appear, the kernel community is getting far better at avoiding the creation of new performance problems. 

Chris Mason started off the session by noting that, at his employer (Facebook), Linux is used anywhere that it is faster than FreeBSD — which, he said, is everywhere. Facebook tends to keep its working sets in RAM, so its workloads tend to be CPU-, memory-, or network-bound. Performance is an important concern there, so the company maintains extensive metrics of how its systems and applications are performing. 

Most of the production systems at Facebook are currently running on 3.10-stable kernels with about 75 additional patches. There are systems running older kernels there, but the Facebook kernel group is slowly forcing them to move to more recent systems, mostly through the simple expedient of refusing to fix problems in older kernels. 

When Facebook first moved to 3.10, its developers felt the usual concerns about performance regressions. As it turned out, this kernel had far fewer problems than expected — but some problems still popped up. There was one 10% performance regression in the [![\[Chris Mason\]](https://static.lwn.net/images/conf/2014/ks/ChrisMason-sm.jpg)](<https://lwn.net/Articles/608957/>) IPv6 stack, but somebody else had already fixed it upstream before Chris was able to track it down. Beyond that, there are some issues with CPU frequency governors, which tend to choose lower-than-optimal frequencies and create unwanted latencies. So Facebook is currently using the ACPI-based CPU frequency governor while trying to figure out how to get the newer `intel_pstate` code to do the right thing. Another problem is contention for the futex bucket lock, which has increased in newer kernels. Within Facebook, Chris has fixed this problem by moving some significant work out of the critical section protected by that lock. Rik van Riel suggest that increasing the number of buckets can also help. 

So what happened when Chris tried running a Facebook workload on a 3.16 kernel? The numbers, he said, were "really good." The workload on that kernel gets 2.5% more queries per second and shows 5% lower latency. It does, however, also use about 4.5% more time in the kernel. That is _after_ applying Chris's futex bucket lock fix; otherwise more than half of all the system time was spent contending for that lock and the system was unusable. 

Going back to the company-wide 3.10 migration, Chris repeated a point from an earlier session: there have been zero regressions caused by patches from the stable tree. They have seen a few problems with the out-of-memory killer locking up the machine when killing POSIX-threads-based programs; that problem has been fixed upstream. There is also a case where combining direct and buffered I/O on a file can cause data corruption, leaving zero-filled pages in the page cache. Chris expressed surprise that existing tests did not find this problem, especially since it's not even caused by a race condition. He plans to look at the xfstests suite to figure out why this problem is not being caught. 

But in general, he said, going to 3.10 was the easiest kernel move ever. 

In a digression from the main topic, Arnd Bergmann asked about the additional patches that Facebook applies to its kernel. Chris responded that one of the more significant ones speeds the task of getting a thread's CPU usage by moving the relevant system calls into the VDSO area. That code, he said, should go upstream soon, but his forward port needs fixing up first; he "will not admit to having worked on it" in its current state. Another one allows the memory management system to avoid zeroing pages in memory-mapped regions during a page fault; he allowed as to how that one might be hard to send upstream. There is also a change to reduce the amount of IPv6 routing table information exported via `/proc`. The company runs its entire internal network on IPv6, so that routing tables are very large; it seems that Java programs have a tendency to query that information and slow everything down. 

[![\[Jan Kara\]](https://static.lwn.net/images/conf/2014/ks/JanKara-sm.jpg)](<https://lwn.net/Articles/608958/>) Getting back to performance issues, Jan Kara, who is working on stabilizing 3.12 for the SLES 12 release, agreed that things have gotten easier recently. His biggest concern seemed to be changes in behavior that benefit some workloads at the expense of others. These changes are not necessarily bad, unless, of course, you are one of the people whose systems slow down. He pointed to changes in the CFQ I/O scheduler and the NUMA balancing work as examples of this kind of change. 

Andi Kleen asked why Chris thought that things were getting better. After all, the kernel process is certainly not slowing down. James Bottomley echoed that question, wondering why regressions are declining when we have not even been tracking them for the last few years. There appear to be a few aspects to the answer, but the most significant factor seems to be easy to identify: there is a lot more performance testing going on now than there was in the past. If a performance problem is introduced, it is far more likely to be caught and fixed long before it finds its way into a stable kernel release. 

Chris added that both Red Hat and SUSE have recently gone through their enterprise distribution stabilization cycles; the extra focus on fixing problems certainly helped to stabilize things. Mel Gorman added that a number of hardware vendors have been introducing new hardware platforms. They put a lot of effort into making their systems go fast, but everybody benefited from the work. But that, he warned, might be a one-time boost that will not always be present. 

Chris closed the session by noting that 3.10, in the end, is the fastest kernel ever run at Facebook. The developers in the room, who have perhaps grown used to being admonished over their tendency to introduce performance regressions, can only have been cheered by that news. 

**Next** : [Kernel self tests](<https://lwn.net/Articles/608959/>).
