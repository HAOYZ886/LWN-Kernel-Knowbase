---
title: When real validation begins
url: https://lwn.net/Articles/629742/
date: "January 21, 2015"
category: "Read-copy-update"
author: "By Jonathan Corbet January 21, 2015 LCA 2015"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
January 21, 2015

* * *

[LCA 2015](<https://lwn.net/Archives/ConferenceByYear/#2015-linux.conf.au>)

No computer-oriented conference is complete without a good war-story presentation or two. Paul McKenney's [LCA 2015 talk](<http://lca2015.linux.org.au/schedule/30141/view_talk?day=thursday>) on the implications of enabling full dynamic tick support for all users fit the bill nicely. The result was an overview of what can happen when your code is unexpectedly enabled on vast numbers of machines — and some lessons for how to avoid disasters in the future. 

#### Some history

Paul started by noting that, in the 1990s, there was little concern about CPU energy efficiency. In fact, in those days, an idle CPU tended to consume more power than one that was doing useful work. That's because an idle processor would sit in a tight loop waiting for something to do; there were no cache misses, so the CPU ran without a break. Delivering regular clock interrupts to an idle processor thus increased its energy efficiency; it was still doing nothing useful, but it would burn less power in the process. 

Things have changed since then, he continued. CPUs are designed to be powered off when they have nothing to do, so clock interrupts to an idle CPU are bad news. But until the early 2000s, that's exactly what was happening on Linux systems. One of the changes merged for the 2.6.21 release in 2007 was [partial dynamic tick support](<https://lwn.net/Articles/223185/>), which removed those idle clock interrupts. 

That was a good step forward, but was not a full solution; delivering regular scheduling interrupts to a busy CPU can also be a problem. Realtime developers don't like clock ticks because they can introduce unwanted latency. High-performance computing users also complain; they are trying to get the most out of their CPUs, and any work that is not directed [![\[Paul McKenney\]](https://static.lwn.net/images/conf/2015/lca/PaulMcKenney-sm.jpg)](<https://lwn.net/Articles/629756/>) toward their problems is just overhead. Beyond that, high-performance computing workloads often communicate results between processors; delaying work on one processor can cause others to wait, creating cascading delays. The scheduling interrupt is often necessary, but, in a high-performance environment, there will only be one process running on a given CPU and no other work to do, so those interrupts can only slow things down. 

Full dynamic tick support was first prototyped by Josh Triplett in 2009; it resulted in a 3% performance gain for CPU-intensive workloads. For people determined to get maximal performance from their systems, 3% is a big deal. But this patch, which was mostly a proof of concept, had some problems. Without the scheduling interrupt, a single task could monopolize the CPU and starve others. There was no process accounting, and read-copy-update (RCU) grace periods could go on forever, with the result that the system could run out of memory. So Frederic Weisbecker's decision to work on a production-ready version of the patch was welcome. 

That code was merged for the 3.10 kernel. It works well, in that there will be no scheduler interrupt while only one task is running on the CPU. There is a residual once-per-second interrupt that, Paul said, serves as a sort of security blanket to make sure nothing slips through the cracks. It can be disabled, but that is not recommended at this time. 

Paul did some of the work to ensure that RCU worked properly in a full dynamic tick environment. He had thought of full dynamic tick as a specialty feature that would only be enabled by users building their own kernels. Still, he was pleasantly surprised to hear that the feature was enabled for all users in the RHEL7 kernel. But, he said ruefully, you would think he would know better after his many years of experience in this industry. Turning on the feature in a major distribution means that everybody is using it. That, he said, is when the real validation begins — validation by users running workloads that he had not thought to test his patches against. 

#### The fun begins

He soon got an email from Rik van Riel asking why the `rcu_sched` process was taking 40% of the CPU. This was happening on a workload that had lots of context switches — a completely different environment than the one the dynamic tick feature was designed for. Paul's first thought was that grace periods were maybe completing too quickly in the presence of a lot of context switches, increasing the amount of grace-period processing that needed to be done. He tried slowing grace-period completion down artificially, but that did not help the problem. Thus, he said, he was forced to actually analyze what was going on. 

The real problem had to do with the [RCU callback offloading mechanism](<https://lwn.net/Articles/522262/>), which moves RCU cleanup work off the CPUs that are being used in the dynamic-tick mode. This cleanup work is done in dedicated kernel threads that can be run on whichever CPU makes the most sense. It's a useful feature for high-performance workloads, but it isn't all that useful for everybody else; indeed, it appeared to be causing problems for other workloads. To address that problem, Paul put in a patch to only enable callback offloading if the `nohz_full` boot parameter (requesting full dynamic tick behavior) is set. 

According to Paul, industry experience shows that one out of six fixes introduces a new bug of its own. This was, he said, one of those fixes. It turns out that RCU is used earlier in the boot process than he had thought, and the switch to the offloaded mode would cause early callbacks from the offloaded CPUs to be lost. The result would certainly be leaked memory, but it can also result in a full system hang if processes are waiting for a specific callback to complete. So another fix went in to make the decision on which CPUs to offload earlier. 

So "now that the bleeding was stopped," he said, it was time to fix the real bug. After all, 40% CPU usage on an 80-CPU system is a bit excessive, and the problem would get worse as the number of CPUs increases. By the time the CPU count got up to 4000 or so, the system simply would not be able to keep up with the load. Since he already gets complaints about RCU performance on 4096-CPU machines, this was a real problem in need of a solution. 

It turned out that a big part of the overhead was the simple process of waking up all of the offload threads at the beginning and end of grace periods. So he decided to hide the problem a bit; rather than wake all threads from the central RCU scheduling thread, he organized them into a tree and made a subset of threads responsible for waking the rest. The idea was to spread the load around the system a bit, but it also happened to reduce the total number of wakeups since it turned out to only be necessary to wake the first level of threads at the beginning of the grace period. 

One of six fixes may introduce a new bug, but in this case, Paul admitted, it was two out of six. Some callbacks that were posted early in the life of the system were not being executed, leading to occasional system hangs. [Yet another fix](<http://git.kernel.org/linus/11ed7f934cb807f26da09547b5946c2e534d1dac>) ensured that they got run, and everything was well again. 

At least, all was well until somebody looked at their system and wondered why there were hundreds of callback-offload threads on a machine with a handful of CPUs. It turns out that some systems have firmware that lies about the number of installed CPUs, and "RCU was stupid enough to believe it." Changing the callback-offload code to defer starting the offload threads until the relevant CPU actually comes online dealt with that one. 

At this point, the callback-offload code passed all of its tests. At least, it passed them all if loadable kernel modules were not enabled — the situation on Paul's machines. The problem was that a module could post callbacks that would still be outstanding when the module was removed. That would lead to the kernel jumping into code that was no longer present — an "embarrassing failure" that can lead to calls (of the telephone variety) back to the relevant kernel developers instead. The solution was to wait for all existing callbacks to be invoked before completing the removal of the module; that wait is done by posting a special callback on each CPU in the system and waiting for them all to report completion. 

As mentioned above, the code was fixed to only run callback threads for online CPUs. That last fix would put callbacks on all CPUs, including those that are currently offline. Since an offline CPU has no offload thread, those callbacks will wait forever. So [yet another fix](<http://git.kernel.org/linus/d7e29933969e5ca7c112ce1368a07911f4485dc2>) ensured that never-online CPUs would not get callbacks posted. 

#### Lessons learned

At this point, as far as anybody knows, things have stabilized and no remaining bugs lurk to attack innocent users. There are, Paul said, a number of lessons that one can learn from his experience. The first of these is to limit the scope of all changes to avoid putting innocent bystanders at risk. Turning on full dynamic tick behavior for all users went against that lesson with unfortunate consequences. We should also recognize that the Linux kernel serves a wide variety of workloads; it will never be possible to test them all. 

Fixes can — and will — generate more bugs. Fixes for minor bugs require _more_ caution before they are applied; since they address a problem seen by only a small subset of users, they have a high probability of creating unforeseen problems for the larger majority. And, Paul said, it is not enough to simply check one's assumptions; one may have built "towers of logic" upon those assumptions and formed habits of thought that are hard to break out of. In this case, the assumption that all users of the dynamic tick code would be building their own kernels led to some unfortunate consequences. And finally, he said, people probably trust him too much. 

[Your editor would like to thank linux.conf.au for funding his travel to the event.]
