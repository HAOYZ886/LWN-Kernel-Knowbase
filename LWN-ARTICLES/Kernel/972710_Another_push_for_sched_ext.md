---
title: Another push for sched_ext
url: https://lwn.net/Articles/972710/
date: "May 9, 2024"
category: "BPF-CPU scheduling; Scheduler-Extensible scheduler class"
author: "By Jonathan Corbet May 9, 2024"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
May 9, 2024

The extensible scheduler class (or "sched_ext") is a comprehensive framework that enables the implementation of CPU schedulers as a set of BPF programs that can be loaded at run time. Despite having attracted a fair amount of interest from the development community, sched_ext has run into considerable opposition and seems far from acceptance into the mainline. The posting by Tejun Heo of [a new version of the sched_ext series](<https://lwn.net/ml/linux-kernel/20240501151312.635565-1-tj@kernel.org/>) at the beginning of May has restarted this long-running discussion, but it is not clear what the end result will be. 

As a quick refresher: sched_ext allows the creation of BPF programs that handle almost every aspect of the scheduling problem; these programs can be loaded (and unloaded) at run time. Sched_ext is designed to safely fall back to the completely fair scheduler should something go wrong (if a process fails to be run within a time limit, for example). It has been used to create a number of special-purpose schedulers, often with impressive performance benefits for the intended workload. See [this 2023 article](<https://lwn.net/Articles/922405/>) for a more detailed overview of this work. 

Heo lists a number of changes that have been made to sched_ext since [the previous version](<https://lwn.net/ml/linux-kernel/20231111024835.2164816-1-tj@kernel.org/>) was posted in November. For the most part, these appear to be adjustments to the BPF API to make the writing of schedulers easier. There is also a new shutdown mechanism that, among other things, disables the BPF scheduler during power-management events like system suspend. There is now support for CPU-frequency scaling, and some debugging interfaces have been added to make developing schedulers easier. The core design of sched_ext appears to have stabilized, though. 

#### Increasing interest

Even before getting to the changes, though, Heo called attention to the increasing interest in sched_ext that is being shown across the community and beyond. Valve is planning to use sched_ext for better game scheduling on the Steam Deck. Ubuntu [is considering shipping it](<https://bugs.launchpad.net/ubuntu/+source/linux/+bug/2045503>) in the 24.10 release. Meta and Google are increasing their use of it in their production fleets. There is also evidently interest in using it in ChromeOS, and Occulus is looking at it as well. Heo concludes that section with: 

> Given that there already is substantial adoption which continues to grow and sched_ext doesn't affect the built-in schedulers or the rest of kernel in an invasive manner, I believe it's reasonable to consider sched_ext for inclusion. 

Whether that inclusion will happen remains an open question, though. The posting of [version 4](<https://lwn.net/ml/linux-kernel/20230711011412.100319-1-tj@kernel.org/>) of the patch set in July 2023 led to a slow-burning discussion on the merits of this development. Scheduler maintainer Peter Zijlstra [rejected the patches outright](<https://lwn.net/ml/linux-kernel/20230726091752.GA3802077@hirez.programming.kicks-ass.net/>), saying: 

> There is not a single doubt in my mind that if I were to merge this, there will be Enterprise software out there that will mandate its own BPF sched thing, or else it won't work. 
> 
> They will not care, they will not contribute, they might even pull a RedHat and only share the code to customers. 

He added that he saw no value in merging the code, and dropped out of the conversation. Mel Gorman also [expressed his opposition](<https://lwn.net/ml/linux-kernel/20230817124457.b5dca734zcixqctu@suse.de/>) to merging sched_ext, echoing Zijlstra's concern that enterprise software would start requiring the use of special-purpose schedulers. He later [added](<https://lwn.net/ml/linux-kernel/20230926092020.3alsvg6vwnc4g3td@suse.de/>) that, in his opinion (one shared with Zijlstra), sched_ext would work actively against the improvement of the current scheduler: 

> I generally worry that certain things may not have existed in the shipped scheduler if plugging was an option including EAS, throttling control, schedutil integration, big.Little, adapting to chiplets and picking preferred SMT siblings for turbo boost. In each case, integrating support was time consuming painful and a pluggable scheduler would have been a relatively easy out that would ultimately cost us if it was never properly integrated. 

Heo, naturally, [disagreed](<https://lwn.net/ml/linux-kernel/ZSXLqNKajmeFRT8x@slm.duckdns.org/>) with a lot of the concerns that had been raised. There are, he said, scheduling problems that cannot be addressed with tweaks to the current scheduler, especially in "hyperscaling" environments like Meta. He disagreed that sched_ext would impose a maintenance burden, arguing that the intrusion of BPF into other parts of the kernel has not had that result. Making it possible for users to do something new is beneficial, even if there will inevitably be ""stupid cases"" resulting from how some choose to use the new feature. In summary, he said, opponents are focused on the potential (and, in his opinion, overstated) costs of sched_ext without taking into account the benefits it would bring. 

#### Restarting the conversation

That message, in October, was the end of the conversation at the time. Heo is clearly hoping for a better result this time around, but Zijlstra's [response](<https://lwn.net/ml/linux-kernel/20240503085232.GC30852@noisy.programming.kicks-ass.net/>) was not encouraging: 

> I fundamentally believe the approach to be detrimental to the scheduler eco-system. Witness the metric ton of toy schedulers written for it, that's all effort not put into improving the existing code. 

He said that he would not accept any part of this patch series until ""the cgroup situation"" has been resolved. That "situation" is a performance problem that affects certain workloads when a number of control groups are in use. Rik van Riel had put together [a patch series](<https://lwn.net/ml/linux-kernel/20190822021740.15554-1-riel@surriel.com/>) to address this problem in 2019, but it never reached the point of being merged; Zijlstra seems to be insisting that this work be completed before sched_ext can be considered, and he gave little encouragement that it would be more favorably considered even afterward. 

Heo [expressed a willingness](<https://lwn.net/ml/linux-kernel/ZjgWzhruwo8euPC0@slm.duckdns.org/>) (albeit reluctantly) to work on the control-group problem if it would clear the way for sched_ext. He strongly disagreed with Zijlstra's characterization of sched_ext schedulers as "toy schedulers" and the claim that working on sched_ext will take effort away from the mainline scheduler, though. There is, he said, no perfect CPU scheduler, so the mainline scheduler has to settle for being good enough for all users. That makes it almost impossible to experiment with ""radical ideas"", and severely limits the pool of people who can work on the scheduler. Much of the energy that goes into sched_ext schedulers, he said, is otherwise unavailable for scheduler development at all. 

There is, he said, value in some of those radical ideas: 

> Yet, the many different ways that even simple schedulers can demonstrates sometimes significant behavior and performance benefits for specific workloads suggest that there are a lot of low hanging fruits in the area. Low hanging fruits that we can't easily reach from our current local optimum. A single implementation which has to satisfy all users all the time is unlikely to be an effective vehicle for mapping out such landscape. 

Igalia developer Changwoo Min, who is working with Valve on gaming-oriented scheduling, [supported Heo's argument](<https://lwn.net/ml/linux-kernel/344145d4-ec56-423f-a016-cbddada8abe5@igalia.com/>), saying that: ""The successful implementation of sched_ext enriches the scheduler community with fresh insights, ideas, and code"". That, as of this writing, is where this conversation stands. 

#### What next?

Sched_ext is on the schedule for the BPF track of the [Linux Storage, Filesystem, Memory-Management, and BPF Summit](<https://events.linuxfoundation.org/lsfmmbpf/>), which begins on May 13\. That discussion will cover the future development of sched_ext but, most likely, will not be able to address the question of whether this work should be merged at all. _That_ discussion could continue, on the mailing lists and elsewhere, for some time yet. 

Sometimes, when a significant kernel development stalls in this way, distributors that see value in it will ship the patches anyway, as Ubuntu, Valve, and ChromeOS are considering doing. While shipping out-of-tree code is often discouraged, it can also serve to demonstrate interest in a feature and flush out any early problems that result from its inclusion. If things go well, this practice can strengthen the argument for merging the code into the mainline, albeit with the ever-present possibility of changes that create pain for the early adopters. 

Whether that will be the path taken for sched_ext remains to be seen. What is certain is that this work has attracted a lot of interest and is unlikely to go away anytime soon. Sched_ext has the potential to enable a new level of creativity in scheduler development, even if it remains out of the mainline — but that potential will be stronger if it does end up being merged. Significant scheduler patches are not merged quickly even when they are uncontroversial; this one will be slower than most if it is accepted at all.
