---
title: Adding periods to SCHED_DEADLINE
url: https://lwn.net/Articles/396634/
date: "July 20, 2010"
category: "Realtime-Deadline scheduling; Scheduler-Deadline scheduling"
author: "By Jonathan Corbet July 20, 2010"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
July 20, 2010

The Linux scheduler, in both the mainline and realtime versions, provides a couple of scheduling classes for realtime tasks. These classes implement the classic POSIX priority-based semantics, wherein the highest-priority runnable task is guaranteed to have access to the CPU. While this scheduler works as advertised, priority-based scheduling has a number of problems and has not been the focus of realtime research for some time. Cool schedulers in this century are based on deadlines instead. Linux does not yet have a deadline scheduler, though [there is one in the works](<http://lwn.net/Articles/356576/>). A recent discussion on implementing the full deadline model has shown, once again, just how complex it can be to get deadline scheduling right in the real world. 

Deadline scheduling does away with priorities, replacing them with a three-parameter tuple: a worst-case execution time (or budget), a deadline, and a period. In essence, a process tells the scheduler that it will require up to a certain amount of CPU time (the budget) by the given deadline, and that the deadline optionally repeats with the given period. So, for example, a video-processing application might request 1ms of CPU time to process the next incoming frame, expected in 10ms, with a 33ms period thereafter for subsequent frames. Deadline scheduling is appealing because it allows the specification of a process's requirements in a natural way which is not affected by any other processes running in the system. There is also great value, though, in using the deadline parameters to guarantee that a process will be able to meet its deadline, and to reject any process which might cause a failure to keep those guarantees. 

The `SCHED_DEADLINE` scheduler being developed by Dario Faggioli appears to be on track for an eventual mainline merger, though nobody, yet, has been so foolish to set a deadline for that particular task. This scheduler works, but, thus far, it takes a bit of a shortcut: in `SCHED_DEADLINE`, the deadline and the period are assumed to be the same. This simplification makes the "admission test" - the decision as to whether to accept a new `SCHED_DEADLINE` task - relatively easy. Each process gets a "bandwidth" parameter, being the ratio of the CPU budget to the deadline/period value. As long as the sum of the bandwidth values for all processes on a given CPU does not exceed 1.0, the scheduler can guarantee that the deadlines will be met. 

As Dario recently [brought up on linux-kernel](<https://lwn.net/Articles/396640/>), there are users who would like to be able to specify separate deadline and period values. Adjusting the scheduler to implement those semantics is not particularly hard. Coming up with an admission test which insures that deadlines can still be met is rather harder, though. Once the period and the deadline are separated from each other, it becomes possible for processes to miss their deadlines even if the total bandwidth of the CPU has not been oversubscribed. 

To see how this might come about, consider [an example](<https://lwn.net/Articles/396641/>) posted by Dario. Imagine a process which needs 4ms of CPU time with a period of 10ms and a deadline of 5ms. A timeline of how that process might be scheduled could look like this: 

![\[Scheduling timeline\]](https://static.lwn.net/images/ns/kernel/deadline1.png)

Here, the scheduler is able to run the process within its deadline; indeed, there is 1ms of time to spare. Now, though, if a second process comes in with the same set of requirements, the results will not be as good: 

![\[Scheduling timeline\]](https://static.lwn.net/images/ns/kernel/deadline2.png)

Despite the fact that 20% of the available CPU time remains unused, process P2 will miss its deadline by 3ms. In a hard realtime situation, that tardiness could prove fatal. Clearly, the scheduler should reject P2 in this situation. The problem is that detecting this kind of problem is not easy, especially if the scheduler is (as seems reasonable) expected to leave some CPU time for applications and not use it all performing complex admission calculations. For this reason, admission decision algorithms are currently an area of considerable research interest in the academic realtime community. See [this paper by Alejandro Masrur et al. [PDF]](<ftp://ftp.rcs.ei.tum.de/pub/papers/rtsg/edffast.pdf>) or [this one by Marko Bertogna [PDF]](<http://retis.sssup.it/~marko/papers/ICPP09.pdf>) for examples of how involved it can get. 

There are a couple of ways to simplify the problem. One of those would be to change the bandwidth calculation to be the ratio of the CPU budget to the relative deadline time (rather than to the period). For the example processes shown above, each has a bandwidth of 0.8 using this formula; the scheduler, on seeing that the second process would bump the total to 1.6, could then decide to reject it. Using this calculation, the scheduler can, once again, guarantee that deadlines will be met, but at a cost: this formula will cause the rejection of processes that, in reality, could be scheduled without trouble. It is an overly pessimistic heuristic which will prevent full utilization of the available resources. 

An alternative, proposed by Dario, would be to stick with the period-based bandwidth values for admission decisions and to take the risk that some deadlines might not be met. In this case, user-space developers would be responsible for ensuring that the full set of tasks on the system can be scheduled. They might take some comfort in the fact that, since the overall bandwidth of the CPU would still not be oversubscribed, the amount by which a deadline could be missed would be deterministically bounded. 

That idea [did not survive its encounter](<https://lwn.net/Articles/396646/>) with Peter Zijlstra, who thinks it ruins everything that a deadline scheduler is supposed to provide: 

The whole reason SCHED_FIFO and friends suck so much is that they don't provide any kind of isolation, and thus, as an Operating-System abstraction they're an utter failure. If you take out admission control you end up with a similar situation. 

Peter's suggestion, instead, was to split deadline scheduling logically into two different schedulers. The hard realtime scheduler would retain the highest priority, and would require that the deadline and period values be the same. If, at some future time, a suitable admission controller is developed then that requirement could be relaxed as long as deadlines could still be guaranteed. 

Below the hard realtime scheduler would be a soft realtime scheduler which would have access to (most of) the CPU bandwidth left unused by the hard scheduler. That scheduler could accept processes using period-based bandwidth values with the explicit understanding that deadlines might be missed by bounded amounts. Soft realtime is entirely good enough for a great many applications, so there is no real reason not to provide it as long as hard realtime is not adversely affected. 

So that is probably how things will go, though the real shape of the solution will not be seen until Dario posts the next version of the `SCHED_DEADLINE` patch. Even after this problem is solved, though, deadline scheduling has a number of other challenges to overcome, with good multi-core performance being near the top of the list. So, while Linux will almost certainly have a deadline scheduler at some point, it's still hard to say just when that might be. 

(Readers who are interested in the intersection of academic and practical realtime work might want to peruse the recently-released [proceedings [PDF]](<http://www.artist-embedded.org/docs/Events/2010/OSPERT/OSPERT2010-Proceedings.pdf>) from the OSPERT 2010 conference, held in Brussels in early July.)
