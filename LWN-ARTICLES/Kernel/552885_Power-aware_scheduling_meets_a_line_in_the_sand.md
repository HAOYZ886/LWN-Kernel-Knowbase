---
title: "Power-aware scheduling meets a line in the sand"
url: https://lwn.net/Articles/552885/
date: "June 5, 2013"
category: "Power management-CPU scheduling; Scheduler-and power management"
author: "By Jonathan Corbet June 5, 2013"
---

By **Jonathan Corbet**  
June 5, 2013

As mobile and embedded processors get more complex — and more numerous — the interest in improving the power efficiency of the scheduler has increased. While [a number of power-related scheduler patches](<https://lwn.net/Articles/546664/>) exist, none seem all that close to merging into the mainline. Getting something upstream always looked like a daunting task; scheduler changes are hard to make in general, these changes come from a constituency that the scheduler maintainers are not used to serving, and the existence of competing patches muddies the water somewhat. But now it seems that the complexity of the situation has increased again, to the point that the merging of any power-efficiency patches may have gotten even harder. 

The current discussion started at the end of May, when Morten Rasmussen posted [some performance measurements](<https://lwn.net/Articles/552323/>) comparing a few of the existing patch sets. The idea was clearly to push the discussion forward so that a decision could be made regarding which of those patches to push into the mainline. The numbers were useful, showing how the patch sets differ over a small set of workloads, but the apparent final result is unlikely to be pleasing to any of the developers involved: it is entirely possible that none of those patch sets will be merged in anything close to their current form, after Ingo Molnar posted [a strongly-worded "line in the sand" message](<https://lwn.net/Articles/552889/>) on how power-aware scheduling should be designed. 

Ingo's complaint is not really about the current patches; instead, he is unhappy with how CPU power management is implemented in the kernel now. Responsibility for CPU power management is currently divided among three independent components: 

  * The scheduler itself clearly has a role in the system's power usage characteristics. Features like [deferrable timers](<https://lwn.net/Articles/228143/>) and [suppressing the timer tick when idle](<https://lwn.net/Articles/223185/>) have been added to the scheduler over the years in an attempt to improve the power efficiency of the system. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

  * The CPU frequency ("cpufreq") subsystem regulates the clock frequency of the processors in response to each processor's measured idle time. If the processor is idle much of the time, the frequency (and, thus, power consumption) can be lowered; an always-busy processor, instead, should run at a higher frequency if possible. Most systems probably use the [on-demand cpufreq governor](<https://lwn.net/Articles/384132/>), but others exist. The [big.LITTLE switcher](<https://lwn.net/Articles/539840/>) operates at this level by disguising the difference between "big" and "little" processors to look like a wide range of frequency options. 

  * The [cpuidle subsystem](<https://lwn.net/Articles/384146/>) is charged with managing processor sleep states. One might be tempted to regard sleeping as just another frequency option (0Hz, to be exact), but sleep is rather more complicated than that. Contemporary processors have a wide range of sleep states, each of which differs in the amount of power consumed, the damage inflicted upon CPU caches, and the time required to enter and leave that state. 

Ingo's point is that splitting the responsibility for power management decisions among three components leads to a situation where no clear policy can be implemented: 

Today the power saving landscape is fragmented and sad: we just randomly interface scheduler task packing changes with some idle policy (and cpufreq policy), which might or might not combine correctly. Even when the numbers improve, it's an entirely random, essentially unmaintainable property: because there's no clear split (possible) between 'scheduler policy' and 'idle policy'. 

He would like to see a new design wherein the responsibility for all of these aspects of CPU operation has been moved into the scheduler itself. That, he claims, is where the necessary knowledge about the current workload and CPU topology lives, so that is where the decisions should be made. Any power-related patches, he asserts, must move the system in that direction: 

This is a "line in the sand", a 'must have' design property for any scheduler power saving patches to be acceptable - and I'm NAK-ing incomplete approaches that don't solve the root design cause of our power saving troubles. 

Needless to say, none of the current patch sets include a fundamental redesign of the scheduler, cpuidle, and cpufreq subsystems. So, for all practical purposes, all of those patches have just been rejected in their current form — probably not the result the developers of those patches were hoping for. 

Morten [responded](<https://lwn.net/Articles/552892/>) with a discussion of the kinds of issues that an integrated power-aware scheduler would have to deal with. It starts with basic challenges like defining scheduling policies for power-efficient operation and defining a mechanism by which a specific policy can be chosen and implemented. There would be a need to represent the system's power topology within the scheduler; that topology might not match the cache hierarchy represented by the existing [scheduling domains](<https://lwn.net/Articles/80911/>) data structure. Thermal management, which often involves reducing CPU frequencies or powering down processors entirely, would have to be factored in. And so on. In summary, Morten said: 

This is not a complete list. My point is that moving all policy to the scheduler will significantly increase the complexity of the scheduler. It is my impression that the general opinion is that the scheduler is already too complicated. Correct me if I'm wrong. 

In his view, the existing patch sets are part of an incremental solution to the problem and a step toward the overall goal. Whether Ingo will see things the same way is, as of this writing, unclear. His words were quite firm, but lines in the sand are also relatively easy to relocate. If he holds fast to his expressed position, though, the addition of power-aware scheduling could be delayed indefinitely. 

It is not unheard of for subsystem maintainers to insist on improvements to existing code as a precondition to merging a new feature. At past kernel summits, such requirements have been seen as being unfair, but they sometimes persist anyway. In this case, Ingo's message, on its face, demands a redesign of one of the most complex core kernel subsystems before (more) power awareness can be added. That is a significant raising of the bar for developers who were already struggling to get their code looked at and merged. A successful redesign on that scale is unlikely to happen unless the current scheduler maintainers put a fair amount of their own time into the requested redesign. 

The cynical among us could certainly see this position as an easy way to simply make the power-aware scheduling work go away. That is certainly an incorrect interpretation, though. The more straightforward explanation — that the scheduler maintainers want to see the code get better and more maintainable over time — is far more likely. What has to happen now is the identification of a path toward that better scheduler that allows for power management improvements in the short term. The alternative is to see the power-aware scheduler code relegated to vendor and distributor trees, which seems like a suboptimal outcome.
