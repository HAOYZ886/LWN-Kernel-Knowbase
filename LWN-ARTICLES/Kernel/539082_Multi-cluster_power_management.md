---
title: "Multi-cluster power management"
url: https://lwn.net/Articles/539082/
date: "February 20, 2013"
category: "Architectures-Arm; big.LITTLE"
author: "By Jonathan Corbet February 20, 2013"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
February 20, 2013

The ARM "[big.LITTLE](<https://lwn.net/Articles/481055/>)" architecture is an interesting beast: it combines clusters of two distinct ARM-based CPU designs into a single processor. One cluster contains relatively slow Cortex-A7 CPUs that are highly power-efficient, while the other cluster is made up of fast, power-hungry Cortex-A15 CPUs. These CPUs can be powered up and down in any combination, but there are additional power savings if an entire cluster can be powered down at once. Power-efficient scheduling is currently a challenge for Linux even on homogeneous architectures; big.LITTLE throws another degree of freedom into the mix that the scheduler is absolutely unprepared to deal with, currently. 

As a result, the initial approach to big.LITTLE is to treat each pair of fast and slow CPUs as if it were a single CPU with high- and low-frequency modes. That approach reduces the problem to writing an appropriate cpufreq governor at the cost of forcing one CPU in each pair to be powered down at any given time. The big.LITTLE patch set is more fully described in the article linked above; that [patch set](<http://article.gmane.org/gmane.linux.ports.arm.kernel/208625>) is coming along but is not yet ready for merging into the mainline. One piece of the larger patch set that might be ready for 3.9, though, is the ["multi-cluster power management" (MCPM) code](<https://lwn.net/Articles/536324/>). 

The Linux kernel has reasonably good CPU power management, but that code, like the scheduler, was not designed with multiple, dissimilar clusters in mind. Fixing that requires adding logic that can determine when entire clusters must be powered up and down, along with the code that actually implements those transitions. The MCPM subsystem is concerned with the latter part of the problem, which is not as easy as one might expect. 

Multi-cluster power management involves the definition of a state machine that implements a 2x3 table of states. Along one axis are the three states describing the cluster's current power situation: `CLUSTER_DOWN`, `CLUSTER_UP`, and `CLUSTER_GOING_DOWN`. The first two are steady states, while the third indicates that the cluster is being powered down, but that the power-down operation is not yet complete. The other axis in the state table describes whether the kernel running on some CPU has decided that the cluster needs to be powered up or not; those states are called `INBOUND_NOT_COMING_UP` and `INBOUND_COMING_UP`. The table as a whole thus contains six states, along with a well-defined set of rules describing transitions between those states. 

#### Shutdown

To begin with, imagine a cluster that is in a small portion of the state space: it is either fully powered up or fully powered down: 

> ![\[state
diagram\]](https://static.lwn.net/images/2013/mcpm/mcpm1.png)

The cluster is running or not; in either one of the above state combinations, there is no plan to bring up the cluster (the `INBOUND_COMING_UP` substate would make no sense in a fully-running cluster in any case). 

If we start from the top of the diagram (`CLUSTER_UP`), we can then trace out the sequence of steps needed to bring the cluster down. The first of those, once the power-down decision has been made, is to determine which CPU is (in the MCPM terminology) the "last man" that is in charge of shutting everything down and turning off the lights on its way out. Since the cluster is fully operational, that decision is relatively easy; a would-be last man simply acquires the relevant spinlock and elects itself into the position. Once that has happened, the last man pushes the cluster through to the `CLUSTER_DOWN` state: 

> ![\[state
diagram\]](https://static.lwn.net/images/2013/mcpm/mcpm2.png)

All transitions marked with solid red arrows are executed by the last man CPU. Once the decision to power down has been made, the cluster moves to `CLUSTER_GOING_DOWN`, where the cleanup work is done. Among other things, the last man will wait until all other CPUs in the cluster have powered themselves down. Once everything is ready, the last man pushes the cluster into `CLUSTER_DOWN`, powering itself down in the process. 

#### Coming back up

Bringing the cluster back up is a similar process, but with an interesting challenge: the CPUs in the cluster must elect a "first man" CPU to perform the initialization work far enough that the kernel can run safely on all the other CPUs. The problem is that, when a cluster first powers up, there may be no memory coherence between the CPUs in that cluster, so spinlocks are not a reliable mechanism for mutual exclusion. Some other mechanism must be used to safely choose a first man; that mechanism is called "voting mutexes" or "vlocks." 

The core idea behind vlocks is that, while atomic instructions will not work between CPUs, it is still possible to use memory barriers to ensure that other CPUs can see a specific memory change. Acquiring a vlock in this environment is a multi-step operation: a CPU will indicate that it is about to vote for a lock holder, then vote for itself. Once (1) at least one CPU has voted for itself, and (2) all CPUs interested in voting have had their say, the CPU that voted last wins. The [vlocks.txt documentation file](<https://lwn.net/Articles/539084/>) included with the patch set provides the following pseudocode to illustrate the algorithm: 

```
int currently_voting[NR_CPUS] = { 0, };
    	int last_vote = -1; /* no votes yet */
    
    	bool vlock_trylock(int this_cpu)
    	{
    		/* signal our desire to vote */
    		currently_voting[this_cpu] = 1;
    		if (last_vote != -1) {
    			/* someone already volunteered himself */
    			currently_voting[this_cpu] = 0;
    			return false; /* not ourself */
    		}
    
    		/* let's suggest ourself */
    		last_vote = this_cpu;
    		currently_voting[this_cpu] = 0;
    
    		/* then wait until everyone else is done voting */
    		for_each_cpu(i) {
    			while (currently_voting[i] != 0)
    				/* wait */;
    		}
    
    		/* result */
    		if (last_vote == this_cpu)
    			return true; /* we won */
    		return false;
    	}
```

Missing from the pseudocode is the use of memory barriers to make each variable change visible across the cluster; in truth, the memory caches for the cluster have not been enabled at the time that the first-man election takes place, so few barriers are necessary. Needless to say, vlocks are relatively slow, but that doesn't matter much when compared to a heavyweight operation like powering up an entire cluster. 

Once a first man has been chosen, it drives the cluster through a set of states on its way back to full functionality: 

> ![\[state
diagram\]](https://static.lwn.net/images/2013/mcpm/mcpm3.png)

The dotted green lines indicate state transitions executed by the inbound, first-man CPU. When a decision is made to power the cluster up, the first man will switch to the `CLUSTER_DOWN / INBOUND_COMING_UP` combination. While the cluster is in this state, the first man is the only CPU running; its job is to initialize things to the point that the other CPUs can safely resume the kernel with properly-functioning mutual exclusion primitives. Once that has been achieved, the cluster moves to `CLUSTER_UP / INBOUND_COMING_UP` while the other CPUs come on line; a final transition to `CLUSTER_UP / INBOUND_NOT_COMING_UP` happens shortly thereafter. 

That describes the basic mechanism, but leaves one interesting question unaddressed: what happens when CPUs disagree about whether the cluster should go up or down? Such disagreements will not happen during the power-up process; the cluster is being brought online to execute a specific task that will still need to be done. But it is possible for the kernel as a whole to change its mind about powering a cluster down; an unexpected interrupt or load spike could indicate that the cluster is still needed. In that case, a new first man may make an appearance while the last man is trying to clock out and go home. This situation is handled by having the first man transition the cluster into the sixth state combination: 

> ![\[state
diagram\]](https://static.lwn.net/images/2013/mcpm/mcpm4.png)

The `CLUSTER_GOING_DOWN / INBOUND_COMING_UP` state encapsulates the conflicted situation where the CPUs differ on the desired state. The eventual outcome needs to be a powered-up, functioning cluster. The last man must occasionally check for this state transition as it goes through its power-down rituals; when it notices that the cluster actually wants to be up, it faces a choice: 

> ![\[state
diagram\]](https://static.lwn.net/images/2013/mcpm/mcpm5.png)

The optimal solution would be to abort the power-down process, unwind any work that has been done, and put the cluster into the `CLUSTER_UP / INBOUND_COMING_UP` state, at which point the first man can finish the job. Should that not be practical, though, the last man can complete the job and switch to `CLUSTER_DOWN / INBOUND_COMING_UP` instead; the first man will then go through the full power-up operation. Either way, the end result will be a functioning cluster. 

#### A few closing notes

The above text pretty much describes the process used to change a cluster's power state; most of the rest is just architecture-specific details. For the curious, a lot more information can be found in [cluster-pm-race-avoidance.txt](<https://lwn.net/Articles/539087/>), included with the MCPM patch set. It is noteworthy that the entire MCPM patch set is contained within the ARM architecture subtree; indeed, the entire big.LITTLE patch is ARM-specific. Perhaps that is how it needs to be, but it is also not difficult to imagine that other architectures may, at some point, follow ARM into the world of heterogeneous clusters. There may come a time when many of the lessons learned here will need to be applied to generic code. 

Traditionally, ARM developers have confined themselves to working with a specific ARM subarchitecture, leading to a lot of duplicated (and substandard) code under `arch/arm` as a whole. More recently, there has been a big push to work across the ARM subarchitectures; that has resulted in a lot of cleaned up support code and abstractions for ARM as a whole. But, possibly, the ARM developers are still a little bit nervous about stepping outside of `arch/arm` and making changes to the core kernel when those changes are needed. Given that there are probably more Linux systems running on ARM processors than any other, it would be natural to expect that the needs of the ARM architecture would drive the evolution of the kernel as a whole. That is certainly happening, but, one could argue, it could be happening more often and more consistently. 

One could instead argue that the big.LITTLE patch set is a short-term hack intended to get Linux running on the relevant hardware until a proper solution can be implemented. The "proper solution" is still likely to need MCPM, though, and, in any case, this kind of hack has a tendency to stick around for a long time. There is almost certainly a long list of use cases for which the basic big.LITTLE approach gives more than adequate results, while getting proper performance out of a true, scheduler-based solution may take years of tricky work. Cpufreq-based Big.LITTLE support may need to persist for a long time while a scheduler-based approach is implemented and stabilized. 

That work is currently underway in the form of the [big LITTLE MP](<https://lwn.net/Articles/501501/>) project; there are [patches](<https://lwn.net/Articles/539089/>) being passed around within Linaro now. Needless to say, this work does touch the core scheduler, with over 1000 lines added to `kernel/sched/fair.c`. Thus far, though, this work has been done by ARM developers with little code from core scheduler developers and no exposure on the linux-kernel mailing list. One can only imagine that, once the linux-kernel posting is made, there will be a reviewer comment or two to address. So big LITTLE MP is probably not headed for the mainline right away. 

Big LITTLE MP may well be one of the first significant core kernel changes to be driven by the needs of the mobile and embedded community. It will almost certainly not be the last. The changing nature of the computing world has already made itself felt by bringing vast numbers of developers into the kernel community. Increasingly, one can expect those developers to take their place in the decision-making process for the kernel as a whole. Once upon a time, it was said that the kernel was entirely driven by the needs of enterprises. To the extent that was true, the situation is changing; we are currently partway through a transition to where enterprise developers have a lot of help from the mobile and embedded community.
