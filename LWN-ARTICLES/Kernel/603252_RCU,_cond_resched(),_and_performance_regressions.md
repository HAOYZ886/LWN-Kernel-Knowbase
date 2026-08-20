---
title: "RCU, cond_resched(), and performance regressions"
url: https://lwn.net/Articles/603252/
date: "June 24, 2014"
category: "Performance regressions; Read-copy-update"
author: "By Jonathan Corbet June 24, 2014"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
June 24, 2014

Performance regressions are a constant problem for kernel developers. A seemingly innocent change might cause a significant performance degradation, but only for users and workloads that the original developer has no access to. Sometimes these regressions can lurk for years until the affected users update their kernels and notice that things are running more slowly. The good news is that the development community is responding with more testing aimed at detecting performance regressions. This testing found a classic example of this kind of bug in 3.16; the bug merits a look as an example of how hard it can be to keep things working optimally for a wide range of users. 

#### The birth of a regression

The kernel's read-copy-update (RCU) mechanism enables a great deal of kernel scalability by facilitating lock-free changes to data structures and batching of cleanup operations. A fundamental aspect of RCU's operation is the detection of "quiescent states" on each processor; a quiescent state is one in which no kernel code can hold a reference to any RCU-protected data structure. Initially, quiescent states were defined as times when the processor was running in user space, but things have gotten rather more complex since then. (See LWN's [lengthy list of RCU articles](<https://lwn.net/Kernel/Index/#Read-copy-update>) for lots of details on how this all works). 

The kernel's [full tickless mode](<https://lwn.net/Articles/549580/>), which is only now becoming ready for serious use, can make the detection of quiescent states more difficult. A CPU running in the tickless mode will, due to the constraints of that mode, be running a single process. If that process stays within the kernel for a long time, no quiescent states will be observed. That, in turn, prevents RCU from declaring the end of a "grace period" and running the (possibly lengthy) set of accumulated RCU callbacks. Delayed grace periods can result in excessive latencies elsewhere in the kernel or, if things go really badly, out-of-memory problems. 

One might argue (as some developers did) that code that loops in the kernel in this way already has serious problems. But such situations do come about. Eric Dumazet [mentioned](<https://lwn.net/Articles/603259/>) one: a process calling `exit()` when it has thousands of sockets open. Each of those open sockets will result in structures being freed via RCU; that can lead to a long list of work to be done while that same process is still closing sockets and, thus, preventing RCU processing by looping in the kernel. 

RCU developer Paul McKenney put together [a solution](<http://git.kernel.org/linus/ac1bea85781e9004da9b3e8a4b097c18492d857c>) to this problem based on a simple insight: the kernel already has a mechanism for allowing other things to happen while some sort of lengthy operation is in progress. Code that is known to be prone to long loops will, on occasion, call `cond_resched()` to give the scheduler a chance to run a higher-priority process. In the tickless situation, there will be no higher-priority process, though, so, in current kernels, `cond_resched()` does nothing of any use in the tickless mode. 

But kernel code can only call `cond_resched()` in places where it can handle being scheduled out of the CPU. So it cannot be running in an atomic context and, thus, cannot hold references to any RCU-protected data structures. In other words, a call to `cond_resched()` marks a quiescent state; all that is needed is to tell RCU about it. 

As it happens, `cond_resched()` is called in a lot of performance-sensitive places, so it is not possible to add a lot of overhead there. So Paul did not call into RCU to signal a quiescent state with every `cond_resched()` call; instead, that function was modified to increment a per-CPU counter and, using that counter, only call into RCU once for every 256 (by default) `cond_resched()` calls. That appeared to fix the problem with minimal overhead, so the patch was merged during the 3.16 merge window. 

Soon thereafter, Dave Hansen [reported](<https://lwn.net/Articles/603262/>) that one of his benchmarks (a program which opens and closes a lot of files while doing little else) had slowed down, and that, with bisection, he had identified the `cond_resched()` change as the culprit. Interestingly, the problem is not with `cond_resched()` itself, which remained fast as intended. Instead, the change caused RCU grace periods to happen more often than before; that caused RCU callbacks to be processed in smaller batches and led to increased contention in the slab memory allocator. By changing the threshold for quiescent states from every 256 `cond_resched()` calls to a much larger number, Dave was able to get back to a 3.15 level of performance. 

#### Fixing the problem

One might argue that the proper fix is simply to raise that threshold for all users. But doing so doesn't just restore performance; it also restores the problem that the `cond_resched()` change was intended to fix. The challenge, then, is finding a way to fix one workload's problem without penalizing other workloads. 

There is an additional challenge in that some developers would like to make `cond_resched()` into a complete no-op on fully preemptable kernels. After all, if the kernel is preemptable, there should be no need to poll for conditions that would require calling into the scheduler; preemption will simply take care of that when the need arises. So fixes that depend on `cond_resched()` continuing to do something may fail on preemptable kernels in the future. 

Paul's [first fix](<https://lwn.net/Articles/603263/>) took the form of a series of patches making changes in a few places. There was still a check in `cond_resched()`, but that check took a different form. The RCU core was modified to take note when a specific processor holds up the conclusion of a grace period for an excessive period of time; when that condition was detected, a per-CPU flag would be set. Then, `cond_resched()` need only check that flag and, if it is set, note the passing of a quiescent period. That change reduced the frequency of grace periods, restoring much of the lost performance. 

In addition, Paul introduced a new function called `cond_resched_rcu_qs()`, otherwise known as "the slow version of `cond_resched()`". By default, it does the same thing as ordinary `cond_resched()`, but the intent is that it would continue to perform the RCU grace period check even if `cond_resched()` is changed to skip that check — or to do nothing at all. The patch changed `cond_resched()` calls to `cond_resched_rcu_qs()` in a handful of strategic places where problems have been observed in the past. 

This solution worked, but it left some developers unhappy. For those who are trying to get the most performance out of their CPUs, any overhead in a function like `cond_resched()` is too much. So Paul came up with [a different approach](<https://lwn.net/Articles/603266/>) that requires no checks in `cond_resched()` at all. Instead, when the RCU core notices that a CPU has held up the grace period for too long, it sends an inter-processor interrupt (IPI) to that processor. That IPI will be delivered when the target processor is not running in atomic context; it is, thus, another good time to note a quiescent state. 

This solution might be surprising at first glance: IPIs are expensive and, thus, are not normally seen as the way to improve scalability. But this approach has two advantages: it removes the monitoring overhead from the performance-sensitive CPUs, and the IPIs only happen when a problem has been detected. So, most of the time, it should have no impact on CPUs running in the tickless mode at all. It would thus appear that this solution is preferable, and that this particular performance regression has been solved. 

#### How good is good enough?

At least, it would appear that way if it weren't for the fact that Dave [still observes a slowdown](<https://lwn.net/Articles/603269/>), though it is much smaller than it was before. The solution is, thus, not perfect, but Paul is [inclined to declare victory](<https://lwn.net/Articles/603270/>) on this one anyway: 

So, given that short grace periods help other workloads (I have the scars to prove it), and given that the patch fixes some real problems, and given that the large number for rcutree.jiffies_till_sched_qs got us within 3%, shouldn't we consider this issue closed? 

Dave still [isn't entirely happy](<https://lwn.net/Articles/603344/>) with the situation; he noted that the regression is closer to 10% with the default settings, and said ""This change of existing behavior removes some of the benefits that my system gets out of RCU"". Paul [responded](<https://lwn.net/Articles/603345/>) that he is ""not at all interested in that micro-benchmark becoming the kernel's straightjacket"" and sent in [a pull request](<https://lwn.net/Articles/603346/>) including the second version of the fix. If there are any real-world workloads that are adversely affected by this change, he suggested, there are a number of ways to tune the system to mitigate the problem. 

Regardless of whether this issue is truly closed or not, this regression demonstrates some of the hazards of kernel development on contemporary systems. Scalability pressures lead to complex code trying to ensure that everything happens at the right time with minimal overhead. But it will never be possible for a developer to test with all possible workloads, so there will often be one that shows a surprising performance regression in response to a change. Fixing one workload may well penalize another; making changes that do not hurt any workloads may be close to impossible. But, given enough testing and attention to the problems revealed by the tests, most problems can hopefully be found and corrected before they affect production users.
