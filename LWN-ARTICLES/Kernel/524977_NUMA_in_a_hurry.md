---
title: NUMA in a hurry
url: https://lwn.net/Articles/524977/
date: "November 14, 2012"
category: "Memory management-NUMA systems; NUMA; Scheduler-NUMA"
author: "By Jonathan Corbet November 14, 2012"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
November 14, 2012

The kernel's behavior on non-uniform memory access (NUMA) systems is, by most accounts, suboptimal; processes tend to get separated from their memory, leading to lots of cross-node traffic and poor performance. Until now, the work to improve this situation has been a story of two competing patch sets; it recently [appeared](<https://lwn.net/Articles/522093/>) that one of them may be set to be merged as the result of decisions made outside of the community's view. But nothing in memory management is ever simple, so it should be unsurprising that the NUMA scheduling discussion has become more complicated. 

On November 6, memory management hacker Mel Gorman, who had not contributed code of his own toward NUMA scheduling so far, posted a new patch series called "[Foundation for automatic NUMA balancing](<https://lwn.net/Articles/523065/>)," or "balancenuma" for short. He pointed out that there were objections to both of the existing approaches to NUMA scheduling and that it was proving hard to merge the best from each. So his objective was to add enough infrastructure to the memory management subsystem to make it easy to experiment with different NUMA placement policies. He also implemented a placeholder policy of his own: 

The actual policy it implements is a very stupid greedy policy called "Migrate On Reference Of pte_numa Node (MORON)". While stupid, it can be faster than the vanilla kernel and the expectation is that any clever policy should be able to beat MORON. 

In short, the MORON policy works by instantly migrating pages whenever a cross-node reference is detected using the NUMA hinting mechanism. Mel's [second version](<https://lwn.net/Articles/524667/>), posted one week later, fixes a number of problems, adds the "home node" concept (that tries to keep processes and their memory on a single "home" NUMA node), and adds some statistics gathering to implement a "CPU follows memory" policy that can move a process to a new home node if it appears that better memory locality would result. 

Andrea Arcangeli, author of the AutoNUMA approach, [said that balancenuma "looks OK"](<https://lwn.net/Articles/524999/>) and that AutoNUMA could be built on top of it. Ingo Molnar, instead, was less accepting, [saying](<https://lwn.net/Articles/525000/>) ""I've picked up a number of cleanups from your series and propagated them into tip:numa/core tree."" He later added [a request](<https://lwn.net/Articles/525001/>) that Mel rebase his work on top of the numa/core tree. He clearly did not see the patch set as a "foundation" on which to build. A new numa/core patch set was [posted](<https://lwn.net/Articles/524718/>) on November 13\. 

Peter Zijlstra, meanwhile, has posted an "[enhanced NUMA scheduling with adaptive affinity](<https://lwn.net/Articles/524535/>)" patch set. This one does away with the "home node" concept altogether; instead, it looks at memory access patterns to determine where a process's memory lives and who that memory might be shared with. Based on that information, the CPU affinity mechanism is used to move processes to the appropriate nodes. Peter says: 

Note that this adaptive NUMA affinity mechanism integrated into the scheduler is essentially free of heuristics - only the access patterns determine which tasks are related and grouped. As a result this adaptive affinity code is able to move both threads and processes close(r) to each other if they are related - and let them spread if they are not. 

This patch set has not gotten a lot of review comments, and it does not appear to have been folded into the numa/core series as of this writing. 

#### What will happen in 3.8?

The numa/core approach remains in linux-next, which is intended to be the final stage for code that is intended to be merged. And, indeed, Ingo has [reiterated](<https://lwn.net/Articles/525002/>) that he plans to merge this code for the 3.8 cycle, saying ""numa/core sums up the consensus so far"". The use of that language might rightly raise some eyebrows; when there are between two and four competing patch sets (depending on how one counts) aimed at the same problem, the term "consensus" does not usually come to mind. And, indeed, it seems that this consensus does not yet exist. 

Andrew Morton has been overtly grumpy; the existence of numa/core in linux-next has made the management of his tree (which is based on linux-next) difficult — his tree needs to be ready for the 3.8 merge window where, [he thinks](<https://lwn.net/Articles/525004/>), numa/core should not be under consideration: 

And yes, I'm assuming you're not targeting 3.8. Given the history behind this and the number of people who are looking at it, that's too hasty... And I must say that I deeply regret not digging my heels in when this went into -next all those months ago. It has caused a ton of trouble for me and for a lot of other people. 

Hugh Dickins, a developer who is not normally associated with this sort of discussion, [chimed in](<https://lwn.net/Articles/525006/>) as well: 

People are still reviewing and comparing competing solutions. Maybe this latest will prove to be closest to the right answer, maybe it will not. It's, what, about two days old right now? 

If we had wanted to push in a good solution a little prematurely, we would surely have chosen Andrea's AutoNUMA months ago, despite efforts to block it; and maybe we shall still want to go that way. 

Please, forget about v3.8, cut this branch out of linux-next, and seek consensus around getting it right for v3.9. 

Rik van Riel [agreed](<https://lwn.net/Articles/525007/>), saying ""Having unreviewed (some of it NAKed) code sitting in tip.git and you trying to force it upstream is not the right way to go."" He also suggested that, if anything should be considered for merging in 3.8, it would be Mel's foundation patches. 

And that is where the discussion stands as of this writing. There is a lot of uncertainty about what might happen with NUMA scheduling in 3.8, meaning that, most likely, nothing will happen at all. It is highly unlikely that Linus would merge the numa/core set in the face of the above complaints; he would be far more likely to sit back and tell the developers involved to work out something they can all agree with. So this is a discussion that might go on for a while yet. 

Making changes to the memory management subsystem is a famously hard thing to do, especially when the changes are as large as those being considered here. But there is another factor that is complicating this particular situation. As the term "NUMA scheduling" suggests, this is not just a memory management problem. The path to improved NUMA performance will require coordinated changes to — and greater integration between — the memory management subsystem and the CPU scheduler. It's telling that the developers on one side of this divide are primarily associated with scheduler development, while those on the other side are mostly memory management folks. Each camp is, in a sense, invading the other's turf in an attempt to create a comprehensive solution to the problem; it is not surprising that some disagreements have emerged. 

Also implicit in this situation is that Linus is unlikely to attempt to resolve the disagreement by decree. There are too many developers and too many interrelated core subsystems involved. So some sort of rough consensus will have to be found. Your editor's explicitly unreliable prediction is that little NUMA-related work will be merged in the 3.8 development cycle. Under pressure from several directions, the developers involved will figure out how to resolve their biggest differences in the next few months. The resulting code will likely be at least partially merged for 3.9 — later than many would wish, but the end result is likely to be better than would be seen with a patch set rushed into 3.8.
