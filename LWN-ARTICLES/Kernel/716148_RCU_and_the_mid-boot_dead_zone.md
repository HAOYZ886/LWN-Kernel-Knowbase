---
title: "RCU and the mid-boot dead zone"
url: https://lwn.net/Articles/716148/
date: "March 7, 2017"
category: "Read-copy-update"
author: "March 7, 2017 This article was contributed by Paul McKenney"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

March 7, 2017

This article was contributed by Paul McKenney

When discussing RCU with mainstream formal-verification researchers, there often comes a time when they ask for RCU's specification. There is of course a specification of a sort, which was first published [here](<https://lwn.net/Articles/652156/>), [here](<https://lwn.net/Articles/652677/>), and [here](<https://lwn.net/Articles/653326/>); it is currently maintained [in the Linux-kernel source tree](<http://lxr.free-electrons.com/source/Documentation/RCU/Design/Requirements/Requirements.html>). However, these "specifications" are empirical in nature: As hardware, other parts of the kernel, and workloads change, RCU's specification also changes. This is not what mainstream formal-verification researchers want to hear, so I usually tell them stories of how I learned about various aspects of the RCU specification. This article tells one of those stories. 

But first, let's review [RCU's grace-period guarantee](<https://lwn.net/Articles/652156/#Grace-Period Guarantee>). This guarantee requires that RCU's synchronous grace-period primitives wait for any pre-existing RCU read-side critical sections. For example, consider the following two in-kernel tasks: 

> 

```
>     int x, y, r1, r2;
>     
>     void task0(void)
>     {
>     	WRITE_ONCE(x, 1);
>     	synchronize_rcu();
>     	WRITE_ONCE(y, 1);
>     }
>     
>     void task1(void)
>     {
>     	rcu_read_lock();
>     	r1 = READ_ONCE(x);
>     	r2 = READ_ONCE(y);
>     	rcu_read_unlock();
>     }
>
```

Suppose that `task1()`'s load from `x` returns zero. This means that some part of `task1()`'s RCU read-side critical section (delimited by `rcu_read_lock()` and `rcu_read_unlock()`) executed prior to `task0()`'s store to `x`, which in turn means that this critical section started before `task0()`'s RCU grace period. RCU therefore guarantees that the rest of `task1()`'s critical section section completes before that grace period ends, which in turn means that the read from `y` will return zero. Similarly, if `task1()`'s read from `y` returns one, part of `task1()`'s RCU read-side critical section has executed after `task0()`'s RCU grace period. RCU therefore guarantees that the entirety of `task1()`'s critical section executes after the start of the grace period, which in turn means that the read from `x` will return one. 

In short, RCU read-side critical sections are not permitted to completely overlap RCU grace periods. 

During early boot, it is trivially easy to provide this guarantee because there is only one task and preemption is disabled. This means that the fact that `synchronize_rcu()` has been called means that all pre-existing readers must have been completed. Therefore, RCU's grace-period primitives can be no-ops during early boot. But early boot ends as soon as the kernel starts spawning kthreads. 

At run time, RCU's grace-period guarantee is provided by the run-time RCU machinery, which by that time has been fully initialized. But the run-time RCU machinery cannot operate correctly until after all of RCU's kthreads have been spawned and initialized, which clearly cannot happen until some time after the kernel starts spawning kthreads. 

Let's call time period between early boot and run time the _mid-boot dead zone_. This dead zone starts when the kernel spawns the first kthread, and ends once all of RCU's kthreads have been spawned and are ready. As noted [here](<https://lwn.net/Articles/653326/#Early Boot>), RCU's synchronous grace periods might well deadlock during the mid-boot dead zone. 

Hoping that nobody calls for a synchronous grace period during the mid-boot phase worked well for some years. However, I made the mistake of accidentally causing `synchronize_rcu_expedited()`, `synchronize_rcu_bh_expedited()`, and `synchronize_sched_expedited()` to operate correctly during the mid-boot dead zone. The ACPI developers noticed that these primitives worked, and promptly took full advantage of my lapse, perhaps completely unintentionally. Because I didn't make these functions log a warning if used during the dead zone, these developers had absolutely no hint that they were skating on thin ice. Had they built with `CONFIG_SMP=n` or booted with the `rcu_normal` kernel-boot parameter, RCU would have complained bitterly. However `CONFIG_SMP=n` is used primarily for deep embedded systems, and `rcu_normal` is used primarily on realtime systems, so it is not all that surprising that they didn't test them. 

However, the ACPI developers _did_ notice once v4.9 came out, because that was the release in which I switched `synchronize_rcu_expedited()`, `synchronize_rcu_bh_expedited()`, and `synchronize_sched_expedited()` to workqueues. This change eliminated some ugly interactions with POSIX signals, however it also re-introduced the mid-boot dead zone, which had the minor downside of complete and utter failure for the ACPI developers. 

**Quick quiz** : But wouldn't this mid-boot dead zone end when workqueues are initialized, which happens much earlier than the spawning of RCU's kthreads?   
Answer

Although this could be [fixed in ACPI](<https://marc.info/?l=linux-kernel&m=148410615124160&w=2>), it is easy to imagine a use case that really needed a real RCU grace period. It is therefore preferable to get RCU's mid-boot dead zone out of ACPI's way. If nothing else, eliminating RCU's mid-boot dead zone should save me considerable time explaining that dead zone to future Linux-kernel developers. As usual, this was easier said than done. 

My first thought was to spawn RCU's kthreads much earlier in the boot process, thus narrowing the mid-boot dead zone, so that the ACPI use fell outside of that zone. However, RCU creates different numbers and types of kthreads under different kernel configurations, which complicates the task of creating all these kthreads at one point in the code. This approach therefore did not make it past the design phase, although it did consume at least its share of paper and ink. 

My second thought was to introduce kthreads into RCU's expedited grace-period primitives, given that the expedited code can be driven by a single kthread. Once this is in place, non-expedited synchronous grace periods can be forced to use the expedited code path during the dead zone, which would allow full functionality. This is much simpler than the first approach, and resulted in this reasonably simple [patch](<https://bugzilla.kernel.org/attachment.cgi?id=251301>). Borislav Petkov tested this patch and found that it [fixed the problem](<https://bugzilla.kernel.org/show_bug.cgi?id=192111#c19>), which was another plus. 

However, this patch had the disadvantage of turning RCU into a special kernel subsystem that creates its kthreads before any other kernel subsystem. This might work fine for awhile, but [Murphy](<https://en.wikipedia.org/wiki/Murphy's_law>) says that it is only a matter of time before some other kernel subsystem also needs to be the first to spawn its kthreads. In addition, there is still a dead zone, albeit a very short one. But if kthread creation itself ever needed to invoke synchronous RCU grace periods, this approach would be completely broken. It would be much better if RCU grace periods simply worked throughout the entire boot process. 

My third thought was to make expedited RCU grace periods go back to their 4.8 behavior, so that the requesting task drives the expedited grace period. In order to avoid the ugliness involving POSIX signals, expedited grace periods would switch back to workqueues as soon as RCU's kthreads had been spawned. This assumes that in-kernel tasks never send each other POSIX signals during the mid-boot dead zone, which seems a safe assumption for the moment, and which can be worked around if needed. In addition, it results in a reasonably small [patch](<https://bugzilla.kernel.org/attachment.cgi?id=251331>). 

The great strength of this approach is that there is no longer a mid-boot dead zone: `synchronize_rcu_expedited()`, `synchronize_rcu_bh_expedited()`, `synchronize_sched_expedited()`, `synchronize_rcu()`, `synchronize_rcu_bh()`, and `synchronize_sched()` may be invoked throughout the entire boot process. This in turn simplifies RCU's specification, at a price of only about seventy lines of code added to the kernel, and without the addition of any kthreads. In addition, RCU can continue to spawn its kthreads at `early_initcall()` time, so that RCU need not be the special first subsystem to create kthreads. Finally, the switch to normal run-time operation can happen at `core_initcall()` time: there is no need to switch to run-time mode immediately after RCU's kthreads have been spawned. 

It is still early days for this patch, but current results are quite encouraging. 

This experience resulted in several lessons (re)learned: 

  1. Maintaining uniform semantics across the Linux kernel's boot-time and run-time code can be quite challenging, but greatly improves ease-of-use. 
  2. If you don't make it warn, it won't be considered illegal. 
  3. If you didn't make it warn, but then make it no longer work, you will likely have unhappy users. 

Last, but by no means least, RCU's specification is empirical, and this is the story of how I learned about yet another new-to-me aspect of that specification. 

#### Acknowledgments

I own thanks to Lv Zheng, Borislav Petkov, Stan Kain, Ivan (AKA `waffolz@hotmail.com`), Emanuel Castelo, Bruno Pesavento, Frederic Bezies, and Rafael J. Wysocki for reporting, reviewing, testing, and otherwise keeping me honest. I also owe thanks to Jim Wasko for his support of this effort. 

####  Quick Quiz answer

**Quick Quiz** : But wouldn't this mid-boot dead zone end when workqueues are initialized, which happens much earlier than the spawning of RCU's kthreads? 

**Answer** : In theory, yes. In practice, the kernel might have been booted with `rcu_normal`, which would cause the expedited grace periods to use the non-expedited code path. So in this case, the mid-boot dead zone for `synchronize_rcu_expedited()`, `synchronize_rcu_bh_expedited()`, and `synchronize_sched_expedited()` is exactly the same as that for `synchronize_rcu()`, `synchronize_rcu_bh()`, and `synchronize_sched()`, which ends after RCU's kthreads has been spawned. 

**Back to Quick Quiz 1**.
