---
title: Controlling the CPU scheduler with BPF
url: https://lwn.net/Articles/873244/
date: "October 21, 2021"
category: BPF; Scheduler
author: "By Jonathan Corbet October 21, 2021"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
October 21, 2021

While the BPF virtual machine has been supported by Linux for most of the kernel's existence, its role for much of that time was limited to, as its full name (Berkeley packet filter) would suggest, filtering packets. That [began to change](<https://lwn.net/Articles/475043/>) in 2012 with the introduction of `seccomp()` filtering, and [the pace picked up](<https://lwn.net/Articles/599755/>) in 2014 with the arrival of the extended BPF virtual machine. At this point, BPF hooks have found their way into many kernel subsystems. One area that has remained BPF-free, though, is the CPU scheduler; that could change if some version of [this patch set](<https://lwn.net/ml/linux-kernel/20210916162451.709260-1-guro@fb.com/>) from Roman Gushchin finds its way into the mainline. 

There are several CPU schedulers in the kernel, each of which works cooperatively to handle specific types of workloads. In systems without realtime processes, though, almost all scheduling is done by the Completely Fair Scheduler (CFS), to the point that most people probably just think of it as "the scheduler". CFS is a complicated beast; it embodies a set of hard-learned heuristics that seek to maximize performance for a wide variety of workloads, and has a number of knobs to tweak for the cases where the heuristics need help. CPU scheduling is a complex task, though, and it is not surprising that the results from CFS are not always seen as being optimal by all users. 

Gushchin started the cover letter for the patch set by observing that an extensive look at the effects of the various CFS tuning knobs revealed that most of them have little effect on the performance of the workload. In the end, it came down to a couple of relatively simple decisions: 

> In other words, some our workloads benefit by having long running tasks preempted by tasks handling short running requests, and some workloads that run only short term requests which benefit from never being preempted. 

The best scheduling policy varies from one workload to the next, so there is value in being able to tweak the policy as needed. That said, Gushchin noted most workloads are well served by CFS as it is now; it may not make much sense to add more tweaks for the relatively small set of workloads that can benefit from them. 

This is just the sort of situation where BPF has made inroads into other parts of the kernel. It gives users the flexibility to change policies to meet their needs while being fast enough that it can sensibly be used in performance-critical subsystems like the CPU scheduler while not increasing overhead for systems where it is not in use. It is somewhat surprising that there have been no serious attempts to integrate BPF into the scheduler until now. 

Gushchin's patch set creates a new BPF program type (`BPF_PROG_TYPE_SCHED`) for programs that influence CPU-scheduler decisions. There are three attachment points for these programs: 

  * `cfs_check_preempt_tick` is called during the handling of the scheduler's periodic timer tick; a BPF program attached here can then look at which process is running. If that process should be allowed to continue to run, the hook can return a negative number to prevent preemption. A positive return value, instead, informs the scheduler that it should switch to a different process, thus forcing preemption to happen. Returning zero leaves the decision up to the scheduler as if the hook hadn't been run. 
  * `cfs_check_preempt_wakeup` is called when a process is woken by the kernel; a negative return value will prevent this process from preempting the currently running process, a positive value will force preemption, and zero leaves it up to the scheduler. 
  * `cfs_wakeup_preempt_entity` is similar to `cfs_check_preempt_wakeup`, but it is called whenever a new process is being selected for execution and can influence the decision. A negative return indicates no preemption, positive forces it, and zero leaves the decision to other parts of the scheduler. 

Gushchin notes that, at Facebook, the first experiments using these hooks ""look very promising"". By posting the patch set, he hoped to start a conversation on how BPF could be used within the scheduler. 

For the most part, it seems that this goal has not been attained; the conversation around these patches has been relatively muted. The most significant comments have come from Qais Yousef who, since he comes from the mobile world, has a different perspective on scheduler issues. He noted that, in that realm, vendors tend to heavily modify the CPU scheduler (see [this article](<https://lwn.net/Articles/820825/>) for a look at one vendor's scheduler changes). Yousef would like to see the scheduler improved to the point that these vendor changes are no longer necessary; he [worried](<https://lwn.net/ml/linux-kernel/20211006163949.zwze5du6szdabxos@e107158-lin.cambridge.arm.com/>) that the addition of BPF hooks could thwart that effort: 

> So my worry is that this will open the gate for these hooks to get more than just micro-optimization done in a platform specific way. And that it will discourage having the right discussion to fix real problems in the scheduler because the easy path is to do whatever you want in userspace. I am not sure we can control how these hooks are used. 

Yousef later [recognized](<https://lwn.net/ml/linux-kernel/20211011163852.s4pq45rs2j3qhdwl@e107158-lin.cambridge.arm.com/>) that there could be value in this feature, but suggested it should be tightly controlled. Among other things, he said, BPF programs used as scheduler hooks should be distributed within the kernel tree itself, with any out-of-tree hooks causing the kernel to become tainted, much like how loadable modules work. 

Gushchin's [position](<https://lwn.net/ml/linux-kernel/YV3v3RkxOB6g/O+8@carbon.lan/>) was that, by making it easy to try out scheduler changes, the new BPF hooks could accelerate scheduler development rather than slowing it down. Meanwhile, he suggested, having vendors apply their scheduler changes as BPF programs might be better than the sorts of patches they create now. 

Beyond this exchange, the patch set has not yet received any significant feedback from either the core scheduler developers or the BPF community. That will clearly need to change if this work is to ever be considered for merging into the mainline kernel. Allowing user space to hook into the scheduler is likely to be a hard sell at best, but it's an idea that seems unlikely to go away anytime soon. For better or for worse, the Linux kernel serves a wide variety of users; providing the best solution for every one of them out of the box is always going to be a challenge.
