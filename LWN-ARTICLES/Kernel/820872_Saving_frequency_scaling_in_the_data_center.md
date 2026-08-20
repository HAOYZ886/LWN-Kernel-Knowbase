---
title: Saving frequency scaling in the data center
url: https://lwn.net/Articles/820872/
date: "May 21, 2020"
category: "Power management-Frequency scaling; Schedutil governor; cpufreq"
author: "By Jonathan Corbet May 21, 2020 OSPM"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
May 21, 2020

* * *

[OSPM](<https://lwn.net/Articles/820337/>)

Frequency scaling — adjusting a CPU's operating frequency to save power when the workload demands are low — is common practice across systems supported by Linux. It is, however, viewed with some suspicion in data-center settings, where power consumption is less of a concern and there is a strong emphasis on getting the most performance out of the hardware. At the 2020 [Power Management and Scheduling in the Linux Kernel summit](<http://retis.sssup.it/ospm-summit/>) (OSPM), Giovanni Gherdovich worried that frequency scaling may be about to go extinct in data centers; he made a plea for improving its behavior for such workloads while there is still time. 

He started with a quote from a car-rally driver: "if in doubt, go flat out". This may not actually be the best advice for drivers of motor vehicles, he said, but it may be the right approach for frequency scaling. Users in data centers often switch to the "performance" CPU-frequency governor, which is _not_ frequency scaling. This governor, which simply runs at full speed all the time, reflects one outcome of the tension between energy efficiency and raw performance. Program managers tend to be interested in performance first, and performance is the first thing that customers see. The cost of power usage is only discovered later, resulting in attempts to hack efficiency into a data-center deployment as an afterthought. It would be better to have that efficiency there from the outset, he said. 

He asked the audience a question: assume you are a regional bank running an on-premises data center. Which CPU-frequency governor would you choose? The [intel_pstate](<https://www.kernel.org/doc/html/latest/admin-guide/pm/intel_pstate.html>) powersave governor would be the smart choice for now. But the intel_pstate performance governor beckons. The "schedutil" governor is the upcoming next generation. Would you pick one of those, or just go with whatever default the distribution picked? The choice is not obvious. Frequency scaling looks like a risk for a data-center user. Can the distribution vendor be trusted to have made the right choice? For distributors, the bank is a customer who must be catered to. Which governor would you set as the default? 

#### Mobile-first considered harmful

He would like to see frequency scaling be the obvious choice for data-center users and make the performance governor obsolete. Eventually he would like to see the schedutil governor win universally; it's "too cool not to win". But that is hampered by the (in his view) mobile-first viewpoint taken by the developers working with frequency scaling. The object is to save every last bit of energy with the idea that the performance governor exists for users who don't share that goal. That [![\[Giovanni
Gherdovich\]](https://static.lwn.net/images/conf/2020/ospm/GiovanniGherdovich-sm.png)](<https://lwn.net/Articles/820880/>) results in frequency scaling stagnating on the x86 architecture, which is relatively rare in power-sensitive settings. 

(Your editor, who has been watching for a long time, was amused by this. For many years the complaint was that "big iron" dominated kernel-development priorities; that situation would appear to have changed.) 

So what happens if distributors default to the performance governor for x86 systems? One advantage would be that the task of getting the powersave governor into shape could be dropped, along with the complexity that governor brings. On the other hand, he said, the x86 community will lose its grip on technology that it will certainly need someday. Avoiding the powersave governor on server systems will simply paper over bugs that, in the long run, need to be fixed. The last time this topic came up at SUSE (where Gherdovich works) the powersave contingent won, but the issue will come up again. 

It seems, though, that the performance governor isn't an obvious choice even now. Dhaval Giani said that it can prevent a CPU from going into "turbo" mode, causing some benchmarks to regress. Rafael Wysocki pointed out that frequency scaling is increasingly done in the processor itself, which can cause strange results when the performance governor is selected. 

Gherdovich answered that there appears to be some tension here. The schedutil governor is getting smarter, but the "hardware-managed p-states" feature (called HWP) is pushing things the other way and taking the kernel out of the decision loop. It's not clear how things will play out, and whether frequency scaling will ultimately be controlled in the operating system or the firmware. Wysocki said that the two approaches are not mutually exclusive, though; the operating system works at different time scales than HWP does. It is possible to bring the two into agreement, but there aren't many ways to provide feedback between them now. He has a patch that tries to improve the situation; he will attempt to post it soon. 

Continuing, Gherdovich said that defaults set by distributors are critically important; they are the first thing that users see. A distribution will be evaluated with its default settings; if the results are bad, users will move on without ever getting to the point where they try to tune things. So distributors tend to emphasize their default settings when running tests, resulting in far fewer bug reports for non-default CPU-frequency governors. If performance is the default, powersave will get little attention. Additionally, regressions are not something that can be tolerated; if frequency scaling is ever turned off, it will be almost impossible to turn it back on. The chances of creating performance regressions would just be too high. 

That, he said, leads to a downward spiral for non-performance CPU-frequency governors. The algorithms in those governors will increasingly be tuned for settings outside of the data center, causing data-center users to lose confidence in them entirely. Distributors will just default to performance, there will be no bug reports, bugs won't get fixed, and frequency scaling will just get worse. 

#### Compromise needed

How do we avoid this dark future? Frequency scaling needs to compromise a bit in the direction of performance, he said, if it wants to win the data center. Often the correct choice for the CPU frequency is obvious, and the governor should go with it. But if there is no information available or the indications conflict with each other, that is the time to favor performance. Any other algorithm will be irrelevant on servers. 

For now, he said, the process that is making frequency scaling unsuitable for data centers has not advanced far, but he worries that the priority for upstream developers seems to strongly favor saving power, and he would like to change that somewhat. Wysocki said that anybody who sends patches to the kernel has an agenda — it's why they wrote the patch in the first place. What data-center advocates need to do is to respond to patches that show an agenda falling too heavily on the battery side. 

Gherdovich was seemingly ready for that; he countered by bringing up [this patch](<https://lwn.net/ml/linux-kernel/16144228.tcT5YVROcV%40aspire.rjw.lan/>) merged by Wysocki in 2019. The "I/O boost" heuristic in the powersave governor assumes that, if a task has been waiting for I/O, it will have work to do once that I/O completes, so the governor increases the processor's operating frequency to get that work done quickly. Prior to the patch, it increased the frequency all the way to the maximum; afterward, the frequency ramps up more slowly. This patch regresses the dbench benchmark, Gherdovich said. Wysocki responded that the purpose of the patch was to avoid starving the integrated GPU of power, and to match [an equivalent change](<https://git.kernel.org/linus/a5a0809bc58e>) made to the schedutil governor. 

There may be good reasons for the change, Gherdovich said, but that patch is currently reverted in SUSE kernels, which is clearly a stopgap solution. There are a couple of other out-of-tree patches in those kernels as well, as it turns out. The "idle boost" patch works like I/O boost; it temporarily increases the frequency when a processor exits the idle state. The "ramp up faster" change is an old patch that nobody likes; it causes the frequency to ramp up more quickly when utilization hits a threshold. These patches are expensive to maintain, and SUSE would much rather stick with the mainline. 

Patrick Bellasi asked whether any attempt had been made to use [uclamp](<https://lwn.net/Articles/762043/>) to get the desired performance results; that has not been done. Mel Gorman added that uclamp is disabled in SUSE kernels since it imposes a significant (3-4%) overhead even when it is not used. Bellasi (the author of the uclamp work) was evidently surprised by this and asked for further information, so that problem, at least, may eventually be fixed. 

Gherdovich concluded by putting up some numbers. Reverting the I/O-boost patch increases dbench performance by 10%, he said. The performance per watt of power used drops by 23%, which is not a big problem on a server system; users typically do not want to lose that 10% of throughput even if it's costly in energy terms. The full "spicy-powersave" patch set — the I/O boost revert plus "idle boost" and "ramp up faster" — improves kernel build times by 10% with no power cost at all. 

At the end, Wysocki asked how much performance data-center users were willing to lose to save some power; Gherdovich didn't have a precise answer but did say that 10% is too much. Wysocki expressed a wish that the CPU-frequency governor work would, in the end, converge on a single solution for everybody, probably in the form of the schedutil governor. 

[See [Gherdovich's slides [PDF]](</images/conf/2020/ospm/gherdovich-slides.pdf>) for details and all the performance results.]
