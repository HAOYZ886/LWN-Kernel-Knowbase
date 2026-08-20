---
title: Revisiting CPU hotplug locking
url: https://lwn.net/Articles/570386/
date: "October 16, 2013"
category: Hotplug; Scalability
author: "By Jonathan Corbet October 16, 2013"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
October 16, 2013

Last week's Kernel Page included [an article on a new CPU hotplugging locking mechanism](<https://lwn.net/Articles/569686/>) designed to minimize the overhead of "read-locking" the set of available CPUs on the system. That article remains valid as a description of a clever and elaborate special-purpose locking system, but it seems unlikely that it describes code that will be merged into the mainline. Further discussion — along with an intervention by Linus — has caused this particular project to take a new direction. 

The CPU hotplug locking patch was designed with a couple of requirements in mind: (1) actual CPU hotplug operations are rare, so that is where the locking overhead should be concentrated, and (2) as the number of CPUs in commonly used systems grows, it is no longer acceptable to iterate over the full set of CPUs with preemption disabled. That is why `get_online_cpus()` was designed to be cheap, but also to serve as a sort of sleeping lock. Both of those requirements came into question once other developers started looking at the patch set. 

#### CPU hotplugging as a rare action

Peter Zijlstra's patch set (examined last week), in response to the above-mentioned requirements, went out of its way to minimize the cost of calls to `get_online_cpus()` and `put_online_cpus()` — the locking functions that ensure that no changes will be made to the set of online CPUs during the critical section. Interestingly, one of the first questions came from [Ingo Molnar](<https://lwn.net/Articles/570390/>), who thought that `get_online_cpus()` _still_ wasn't cheap enough. He suggested that read-locking the set of online CPUs should cost nothing, while actual hotplug operations should avoid contention by freezing all tasks in the system. Freezing all tasks is an expensive operation, but, as Ingo put it: 

Actual CPU hot unplugging and replugging is _ridiculously_ rare in a system, I don't understand how we tolerate _any_ overhead from this utter slowpath. 

It was then pointed out ([in the LWN comments too](<https://lwn.net/Articles/570032/>)) that Android systems use CPU hotplug as a crude form of CPU power management. Ingo [dismissed](<https://lwn.net/Articles/570391/>) that use as ""very broken to begin with"", saying that proper power-aware scheduling should be used instead. That may be true, but it doesn't change the fact that hotplugging is used that way — or that the kernel lacks proper power-aware scheduling at the moment anyway. Paul McKenney posted [an interesting look at the situation](<https://lwn.net/Articles/570392/>), noting that CPU hotplugging can serve as an effective defense against scheduler bugs that could otherwise ruin a system's battery life. 

The end result is that, for the next few years at least, CPU hotplugging as a power management technique seems likely to stay around. So, while it still makes sense to put the expense of the necessary locking on that side — actually adding or removing CPUs is not going to be a hugely fast operation in the best of conditions — it would hurt some users to make hotplugging a lot slower. 

#### A different way

This was about the point where Linus [came along](<https://lwn.net/Articles/570399/>) with a suggestion of his own. Rather than set up complex locking, why not use the normal read-copy-update (RCU) mechanism to protect CPU removals? In short, if a thread sees a bit set indicating that a particular CPU exists, all data associated with that CPU will continue to be valid for as long as the reading thread holds an RCU read lock. When a CPU is removed, the bit can be cleared, but the removal of the associated data would have to wait until after an RCU grace period has passed. This mechanism is used throughout the kernel and is well understood. 

There is only one problem: holding an RCU read lock requires disabling preemption, essentially putting the holding thread into atomic context. Peter [expressed his concerns](<https://lwn.net/Articles/570401/>) about disabling preemption in this way. Current `get_online_cpus()` callers assume they can do things like memory allocation that might sleep; that would not be possible if that code had to run with preemption disabled. The other potential problem is that some systems have a lot of CPUs; keeping preemption disabled while iterating over 4096 CPUs could introduce substantial latencies into the system. For these reasons, Peter thought, disabling preemption was not the right way to solve the hotplug locking problem. 

Linus was, to put it mildly, [unimpressed](<https://lwn.net/Articles/570402/>) by this reasoning. It was, he said, the path to low-quality code. Working with preemption disabled, he said, is just the way things should be done in the core kernel: 

Yes, preempt_disable() is harder to use than sleeping locks. You need to do pre-allocation etc. But it is much *much* more efficient. 

And in the kernel, we care. We have the resources. Plus, we can also say "if you can't handle it, don't do it". We don't need new features so badly that we are willing to screw up core code. 

So the sleeping-lock approach has gone out of favor. But, if disabling preemption is to be used instead, solutions must be found to the atomic context and latency problems mentioned above. 

With regard to atomic context, the biggest issue is likely to be memory allocations which, normally, can sleep while the kernel works to free the needed space. There are two ways to handle memory allocations when preemption is disabled. One of those is to use the `GFP_ATOMIC` flag, but code using `GFP_ATOMIC` tends to draw a lot of critical attention from reviewers. The alternative is to either pre-allocate the memory before disabling preemption, or to temporarily re-enable preemption for long enough to perform the allocation. With the latter approach, naturally, it is usually necessary to check whether the state of the universe has changed while preemption was enabled. All told, it makes for more complex programming, but, as Linus noted, it can be very efficient. 

Latency problems can be addressed by disabling preemption inside the loop that passes over all CPUs, rather than outside of it. So preemption is disabled while any given CPU is being processed, but it is quickly re-enabled (then disabled again) between CPUs. That should eliminate any significant latencies, but, once again, the code needs to be prepared for things changing while preemption is enabled. 

Changing CPU hotplug locking along these lines would eliminate the need for the complex locking code that was examined last week. But there is a cost to be paid elsewhere: all code that uses `get_online_cpus()` must be audited and possibly changed to work under the new regime. Peter has [agreed](<https://lwn.net/Articles/570405/>) that this approach is workable, though, and he seems willing to carry out this audit. That work appears to be underway as of this writing. 

To some observers, this sequence of events highlights the difficulties of kernel programming: a talented developer works to create some tricky code that makes things better, only to be told that the approach is wrong. In truth, early patch postings are often better seen as a characterization of the problem than the final solution. As long as developers are willing to let go of their approach when confronted with something better, things work out for the best for everybody involved. That would appear to be the case here; the resulting kernel will perform better while using code that is simpler and adheres more closely to common programming practices.
