---
title: Core scheduling
url: https://lwn.net/Articles/780703/
date: "February 28, 2019"
category: "Scheduler-Core scheduling"
author: "By Jonathan Corbet February 28, 2019"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
February 28, 2019

Kernel developers are used to having to defend their work when posting it to the mailing lists, so when a longtime kernel developer describes their own work as ""expensive and nasty"", one tends to wonder what is going on. The patch set in question is [core scheduling](<https://lwn.net/ml/linux-kernel/20190218165620.383905466@infradead.org/>) from Peter Zijlstra. It is intended to make simultaneous multithreading (SMT) usable on systems where cache-based side channels are a concern, but even its author is far from convinced that it should actually become part of the kernel. 

SMT increases performance by turning one physical CPU into two virtual CPUs that share the hardware; while one is waiting for data from memory, the other can be executing. Sharing a processor this closely has led to security issues and concerns for years, and many security-conscious users disable SMT entirely. The disclosure of the [L1 terminal fault](<https://lwn.net/Articles/762570/>) vulnerability in 2018 did not improve the situation; for many, SMT simply isn't worth the risks it brings with it. 

But performance matters too, so there is interest in finding ways to make SMT safe (or safer, at least) to use in environments with users who do not trust each other. The [coscheduling patch set](<https://lwn.net/Articles/764482/>) posted last September was one attempt to solve this problem, but it did not get far and has not been reposted. One obstacle to this patch set was almost certainly its complexity; it operated at every level of the scheduling domain hierarchy, and thus addressed more than just the SMT problem. 

Zijlstra's patch set is focused on scheduling at the core level only, meaning that it is intended to address SMT concerns but not to control higher-level groups of physical processors as a unit. Conceptually, it is simple enough. On kernels where core scheduling is enabled, a `core_cookie` field is added to the task structure; it is an `unsigned long` value. These cookies are used to define the trust boundaries; two processes with the same cookie value trust each other and can be allowed to run simultaneously on the same core. 

By default, all processes have a cookie value of zero, placing them all in the same trust group. Control groups are used to manage cookie values via a new `tag` field added to the CPU controller. By placing a group of processes into their own group and setting `tag` appropriately, the system administrator can ensure that this group will not share a core with any processes outside of the group. 

Underneath, of course, there is some complexity involved in making all of this work. In current kernels, each CPU in an SMT core schedules independently of the others, but that cannot happen when core scheduling is enabled; scheduling decisions must now take into account what is happening elsewhere in the core. So when one CPU in a core calls into the scheduler, it must evaluate the situation for all CPUs in that core. The highest-priority process eligible to run on any CPU in that core is chosen; if it has a cookie value that excludes processes currently running in other CPUs, those processes must be kicked out to ensure that no unwanted sharing takes place. Other, lower-priority processes might replace these outcasts, but only if they have the right cookie value. 

The CPU scheduler works hard to avoid moving processes between distant CPUs in an attempt to maximize cache effectiveness. Load balancing kicks in occasionally to even out the load on the system as a whole. The calculation changes a bit, though, when core scheduling is in use; moving a process is more likely to make sense if that process can run on a CPU that would otherwise sit idle, even if the moved process leaves a hot cache behind. Thus, if an SMT CPU is forced idle due to cookie exclusion, a new load balancing algorithm will look across other cores for a process with a matching cookie to move onto the idle CPU. 

The patch set has seen a fair amount of discussion. Greg Kerr, representing Chrome OS, [questioned](<https://lwn.net/ml/linux-kernel/CAJGSLMt_X97Ux=1YiZcXWXvBy4n=ExO=2yAJhfbvxDh+wnWPvQ@mail.gmail.com/>) the control-group interface. Making changes to control groups is a privileged operation, but he would like for unprivileged processes to be able to set their own cookies. To that end, he proposed an API based on `~~ptrace()~~ prctl()` calls. Zijlstra [replied](<https://lwn.net/ml/linux-kernel/20190220094255.GE32494@hirez.programming.kicks-ass.net/>) that the interface issues can be worked out later; first it's necessary to get everything working as desired. 

Whether that can be done remains to be seen. As Linus Torvalds [pointed out](<https://lwn.net/ml/linux-kernel/CAHk-=wh32Zgp+bN7G1KH7SuCiY1YSJ41Y-_eWpfWubGjqq2_dw@mail.gmail.com/>), performance is the key consideration here. Core scheduling only makes sense if it provides better throughput than simply running with SMT disabled, so the decision on whether to merge it depends heavily on benchmark results. There is not a lot of data available yet; it seems that perhaps it works better on certain types of virtualized loads (those that only rarely exit back to the hypervisor) and worse on others. Subhra Mazumdar also [reported](<https://lwn.net/ml/linux-kernel/e73e1b19-5418-32c8-584c-823618de67ce@oracle.com/>) a performance regression on database loads. 

Thus, even if core scheduling is eventually accepted, it seems unlikely to be something that everybody turns on. But it may yet be a tool that proves useful for some environments where there is a need to get the most out of the hardware, but where strong isolation between users is also needed. The process of finishing this work and figuring out if it justifies the costs seems likely to take a while in any case; this sort of major surgery to the CPU scheduler is not done lightly, even when its developer doesn't claim to ""hate it with a passion"". So security-conscious users seem likely to be without an alternative to disabling SMT for some time yet.
